## 5. AWS Integration Documentation

```markdown
# AYBIZA AI Voice Agent Platform - Cloudflare+AWS Hybrid Integration

## Hybrid Architecture Overview

The AYBIZA platform leverages a Cloudflare+AWS hybrid architecture to achieve optimal performance, security, and cost efficiency with Claude 3.7 Sonnet integration. This document outlines how Cloudflare services work in conjunction with AWS services to deliver ultra-low latency (<50ms US/UK, <100ms globally), enterprise-grade security, global scale, and 28.6% cost savings compared to AWS-only deployment.

## Architecture Components

### Edge Layer: Cloudflare Services

#### 1. Cloudflare DNS (Managed DNS)
- **Usage**: Global DNS management with anycast routing
- **Configuration**:
  - DNS record management via API
  - Geographic routing for regional optimization
  - Health checks and automatic failover
  - DNSSEC for security
- **Key Features Used**:
  - Anycast DNS for global performance
  - Load balancing with health checks
  - Geographic steering
  - Secondary DNS for redundancy
- **Implementation**:
  ```elixir
  # Example Cloudflare DNS configuration
  defmodule Aybiza.Cloudflare.DNSConfig do
    def dns_records do
      %{
        main_record: %{
          type: "A",
          name: "api.aybiza.com",
          content: "104.16.123.45",  # Cloudflare proxy IP
          proxied: true,
          ttl: 1  # Auto TTL
        },
        regional_records: [
          %{
            type: "A",
            name: "us.api.aybiza.com",
            content: aws_alb_us_east_1(),
            proxied: false,
            ttl: 300
          },
          %{
            type: "A", 
            name: "uk.api.aybiza.com",
            content: aws_alb_eu_west_2(),
            proxied: false,
            ttl: 300
          }
        ]
      }
    end
  end
  ```

#### 2. Cloudflare Workers (Edge Computing)
- **Usage**: Edge computing for voice processing optimization
- **Configuration**:
  - JavaScript workers for edge logic
  - KV storage for edge caching
  - Durable Objects for stateful edge processing
  - WebSocket support for real-time connections
- **Key Features Used**:
  - Sub-10ms edge processing
  - Intelligent request routing
  - Edge authentication
  - Audio preprocessing
- **Implementation**:
  ```javascript
  // Example Cloudflare Worker for voice routing
  export default {
    async fetch(request, env, ctx) {
      const url = new URL(request.url);
      
      // Voice call routing logic
      if (url.pathname.startsWith('/voice/')) {
        const clientLocation = request.cf.country;
        const optimalRegion = determineOptimalRegion(clientLocation);
        
        // Route to appropriate AWS region
        const targetUrl = `https://${optimalRegion}.api.aybiza.com${url.pathname}`;
        
        return fetch(targetUrl, {
          method: request.method,
          headers: request.headers,
          body: request.body
        });
      }
      
      return new Response('Not found', { status: 404 });
    }
  };
  
  function determineOptimalRegion(country) {
    const regionMap = {
      'US': 'us-east-1.internal',
      'CA': 'us-east-1.internal', 
      'GB': 'eu-west-2.internal',
      'DE': 'eu-west-2.internal',
      'FR': 'eu-west-2.internal',
      // Default to US East for global traffic
      'default': 'us-east-1.internal'
    };
    
    return regionMap[country] || regionMap['default'];
  }
  ```

#### 3. Cloudflare CDN (Content Delivery Network)
- **Usage**: Global content delivery and caching
- **Configuration**:
  - Aggressive caching for static assets
  - Cache rules for API responses
  - Edge SSL/TLS termination
  - Origin health monitoring
- **Key Features Used**:
  - 300+ global edge locations
  - Argo Smart Routing
  - Origin Shield
  - Custom cache rules
- **Implementation**:
  - Configured via Cloudflare Dashboard and API

#### 4. Cloudflare Web Application Firewall (WAF)
- **Usage**: Edge security and DDoS protection
- **Configuration**:
  - OWASP Core Rule Set
  - Custom rules for voice-specific threats
  - Rate limiting and bot management
  - Managed rulesets
- **Key Features Used**:
  - DDoS protection (10+ Tbps capacity)
  - Bot management
  - Custom security rules
  - IP reputation filtering
- **Implementation**:
  ```elixir
  # Example WAF rule configuration
  defmodule Aybiza.Cloudflare.WAFRules do
    def voice_protection_rules do
      [
        %{
          action: "block",
          expression: "(http.request.uri.path contains \"/voice/\" and cf.threat_score > 50)"
        },
        %{
          action: "challenge", 
          expression: "(rate(1m) > 100)"
        },
        %{
          action: "allow",
          expression: "(ip.src in {trusted_ip_list})"
        }
      ]
    end
  end
  ```

### Compute Layer: AWS Services

#### 1. AWS Fargate (Platform Version 1.4.0)
- **Usage**: Primary container execution environment for application services
- **Configuration**:
  - CPU and Memory: 2 vCPU / 4GB per task (adjustable based on load)
  - Operating System: Amazon Linux 2
  - Platform Version: 1.4.0
  - Task Role: Custom IAM role with least privilege
- **Integration with Cloudflare**:
  - Application Load Balancer with health checks
  - Origin configuration in Cloudflare
  - SSL certificate management
- **Implementation**:
  ```elixir
  # Example ECS service configuration with Cloudflare integration
  defmodule Aybiza.AWS.ECSConfig do
    def fargate_service_config do
      %{
        cluster: System.get_env("AWS_ECS_CLUSTER_NAME", "aybiza-cluster"),
        service_name: "voice-gateway",
        task_definition: "voice-gateway:#{System.get_env("TASK_DEFINITION_VERSION", "1")}",
        desired_count: String.to_integer(System.get_env("DESIRED_TASK_COUNT", "2")),
        launch_type: "FARGATE",
        platform_version: "1.4.0",
        network_configuration: %{
          awsvpc_configuration: %{
            subnets: String.split(System.get_env("PRIVATE_SUBNET_IDS"), ","),
            security_groups: String.split(System.get_env("SECURITY_GROUP_IDS"), ","),
            assign_public_ip: "DISABLED"
          }
        },
        load_balancers: [
          %{
            target_group_arn: System.get_env("TARGET_GROUP_ARN"),
            container_name: "voice-gateway",
            container_port: 4000
          }
        ],
        health_check_grace_period_seconds: 60,
        scheduling_strategy: "REPLICA"
      }
    end
  end
  ```

#### 2. Amazon ECS (API Version 2014-11-13)
- **Usage**: Container orchestration for application services
- **Configuration**:
  - Capacity Provider Strategy: FARGATE_SPOT (dev/test), FARGATE (prod)
  - Service Discovery: AWS Cloud Map
  - Task Definition: Elixir application + sidecar pattern
- **Integration with Cloudflare**:
  - Target groups for Application Load Balancer
  - Health checks configured for Cloudflare origin monitoring
- **Implementation**:
  ```elixir
  # Example Auto Scaling configuration
  defmodule Aybiza.AWS.AutoScalingConfig do
    def scaling_config do
      %{
        service_namespace: "ecs",
        resource_id: "service/#{System.get_env("AWS_ECS_CLUSTER_NAME")}/#{System.get_env("ECS_SERVICE_NAME")}",
        scalable_dimension: "ecs:service:DesiredCount",
        min_capacity: String.to_integer(System.get_env("MIN_CAPACITY", "2")),
        max_capacity: String.to_integer(System.get_env("MAX_CAPACITY", "10")),
        target_tracking_scaling_policy_configuration: %{
          target_value: 75.0,
          predefined_metric_specification: %{
            predefined_metric_type: "ECSServiceAverageCPUUtilization"
          },
          scale_out_cooldown: 60,
          scale_in_cooldown: 300
        }
      }
    end
  end
  ```

#### 3. AWS Lambda (Runtime nodejs20.x)
- **Usage**: Event-driven processing and webhooks
- **Configuration**:
  - Runtime: Node.js 20.x
  - Architecture: arm64
  - Memory: 1024 MB (configurable)
  - Timeout: 30 seconds
- **Integration with Cloudflare**:
  - API Gateway integration
  - Custom domain with Cloudflare DNS
- **Implementation**:
  ```elixir
  # Example Lambda client for webhook processing
  defmodule Aybiza.AWS.LambdaClient do
    def invoke_webhook_processor(webhook_id, payload) do
      ExAws.Lambda.invoke(
        "aybiza-webhook-processor",
        %{webhook_id: webhook_id, payload: payload},
        %{
          invocation_type: "RequestResponse", 
          log_type: "None"
        }
      )
      |> ExAws.request()
    end
  end
  ```

### Database Services

#### 1. Amazon Aurora PostgreSQL (16.9 or higher)
- **Usage**: Primary relational database with TimescaleDB extension
- **Configuration**:
  - Instance Class: r6g.large (prod), r6g.small (dev/test)
  - Multi-AZ: Yes (production only)
  - Storage: Autoscaling up to 1TB
  - Backup: Daily with 7-day retention
- **Integration with Cloudflare**:
  - Private subnet deployment
  - Connection through AWS ALB
  - No direct exposure to internet
- **Implementation**:
  ```elixir
  # Example Ecto configuration for Aurora PostgreSQL 16.9 with TimescaleDB 2.20.0
  config :aybiza, Aybiza.Repo,
    adapter: Ecto.Adapters.Postgres,
    extensions: [{Timescale.Adapters.Postgres, []}],  # TimescaleDB support
    username: System.get_env("DB_USERNAME", "postgres"),
    password: System.get_env("DB_PASSWORD", "postgres"),
    hostname: System.get_env("DB_HOSTNAME"),
    port: String.to_integer(System.get_env("DB_PORT", "5432")),
    database: System.get_env("DB_NAME", "aybiza_#{Mix.env()}"),
    pool_size: String.to_integer(System.get_env("DB_POOL_SIZE", "10")),
    # Enhanced configuration for PostgreSQL 16
    queue_target: 5000,
    queue_interval: 1000,
    ssl: true,
    ssl_opts: [
      verify: :verify_peer,
      cacertfile: System.get_env("CA_CERT_PATH", "/app/ca_cert.pem"),
      server_name_indication: to_charlist(System.get_env("DB_HOSTNAME")),
      versions: [:"tlsv1.2", :"tlsv1.3"]  # Support TLS 1.3 with PG 16
    ],
    prepare: :named,
    parameters: [
      application_name: "aybiza"
    ],
    timeout: 60_000,
    ownership_timeout: 60_000,
    migration_timestamps: [type: :utc_datetime_usec]
  ```

#### 2. Amazon ElastiCache for Redis (8.0.0)
- **Usage**: Caching, pub/sub, and distributed state
- **Configuration**:
  - Node Type: r6g.medium (prod), t4g.small (dev/test)
  - Engine Version: 8.0.0
  - Parameter Group: Custom for voice processing
  - Multi-AZ: Yes (production)
- **Integration with Cloudflare**:
  - Private subnet deployment
  - Session storage for edge authentication
- **Implementation**:
  ```elixir
  # Example Redis configuration (Updated for Redis 8.0)
  defmodule Aybiza.Redis.Client do
    def config do
      %{
        host: System.get_env("REDIS_HOST"),
        port: String.to_integer(System.get_env("REDIS_PORT", "6379")),
        password: System.get_env("REDIS_PASSWORD"),
        database: String.to_integer(System.get_env("REDIS_DB", "0")),
        ssl: true,
        socket_opts: [
          timeout: 5000,
          send_timeout: 5000,
          tcp_nodelay: true
        ],
        reconnection_attempts: 10,
        reconnection_backoff: 100,
        # Redis 8.0 specific features
        enable_query_engine: true,  # New Redis Query Engine
        enable_json: true,          # Built-in JSON support
        enable_search: true,        # Integrated RediSearch
        enable_timeseries: true,    # Integrated RedisTimeSeries
        enable_bloom: true          # Integrated RedisBloom
      }
    end
  end
  ```

### Networking & Edge Integration

#### 1. Amazon API Gateway (API Version 2015-07-09)
- **Usage**: RESTful API management and WebSocket endpoint for Twilio
- **Configuration**:
  - Endpoint Type: Regional
  - Protocol: HTTP/2, WebSocket
  - Stage variables for environment configuration
  - Custom domain with Cloudflare DNS
- **Integration with Cloudflare**:
  - Custom domain routing through Cloudflare
  - Origin server for Cloudflare Workers
  - SSL certificate from AWS Certificate Manager
- **Implementation**:
  ```elixir
  # Example API Gateway client for WebSocket management
  defmodule Aybiza.AWS.APIGatewayClient do
    def post_to_connection(connection_id, data) do
      ExAws.ApiGatewayManagementApi.post_to_connection(
        connection_id,
        data
      )
      |> ExAws.request()
    end
    
    def get_connection(connection_id) do
      ExAws.ApiGatewayManagementApi.get_connection(connection_id)
      |> ExAws.request()
    end
    
    def delete_connection(connection_id) do
      ExAws.ApiGatewayManagementApi.delete_connection(connection_id)
      |> ExAws.request()
    end
  end
  ```

#### 2. Application Load Balancer (ALB)
- **Usage**: Load balancing for ECS services
- **Configuration**:
  - Multi-AZ deployment
  - Health checks for ECS targets
  - SSL termination
  - WebSocket support
- **Integration with Cloudflare**:
  - Origin server for Cloudflare
  - Health monitoring via Cloudflare
  - IP allowlisting for Cloudflare edge IPs

### Security Services

#### 1. AWS Secrets Manager (API Version 2017-10-17)
- **Usage**: Secure storage for API keys and credentials
- **Configuration**:
  - Automatic Rotation: Enabled for critical secrets
  - KMS Key: Custom CMK
  - Resource Policy: Least privilege
  - Tags: For cost allocation
- **Integration with Cloudflare**:
  - Secure storage of Cloudflare API tokens
  - Automated credential rotation
- **Implementation**:
  ```elixir
  # Example Secrets Manager client
  defmodule Aybiza.AWS.SecretsManagerClient do
    def get_secret(secret_id) do
      ExAws.SecretsManager.get_secret_value(secret_id)
      |> ExAws.request()
      |> case do
        {:ok, %{"SecretString" => secret_string}} ->
          {:ok, Jason.decode!(secret_string)}
        error ->
          error
      end
    end
    
    def get_cloudflare_credentials do
      get_secret("aybiza/cloudflare/api-credentials")
    end
  end
  ```

#### 2. AWS KMS (API Version 2014-11-01)
- **Usage**: Encryption key management
- **Configuration**:
  - Key Type: Customer managed
  - Key Spec: SYMMETRIC_DEFAULT
  - Key Usage: ENCRYPT_DECRYPT
  - Key Rotation: Enabled (yearly)
- **Integration with Cloudflare**:
  - Encrypted storage of Cloudflare configurations
  - Key management for edge processing

### AI/ML Services

#### 1. Amazon Bedrock (API Version 2023-09-30)
- **Usage**: LLM foundation models for conversation
- **Configuration**:
  - Models: 
    - Claude 3.7 Sonnet (anthropic.claude-3-7-sonnet-20250219-v1:0) - Latest with extended thinking
    - Claude 3.5 Sonnet v2 (anthropic.claude-3-5-sonnet-20241022-v2:0)
    - Claude 3.5 Haiku (anthropic.claude-3-5-haiku-20241022-v1:0) - Fastest intelligence
    - Claude 3 Haiku (anthropic.claude-3-haiku-20240307-v1:0) - Most cost-effective
  - Provisioned Throughput: Yes (production) for consistent latency
  - Streaming: Enabled for all models (critical for low latency)
  - Region: us-east-1, us-west-2, eu-west-1 (choose based on latency requirements)
- **Integration with Cloudflare**:
  - Edge caching of model responses
  - Geographic routing to optimal Bedrock regions
- **Implementation**:
  ```elixir
  # Example Bedrock client with Cloudflare integration
  defmodule Aybiza.AWS.BedrockClient do
    @claude_3_7_sonnet "anthropic.claude-3-7-sonnet-20250219-v1:0"
    @claude_3_5_sonnet "anthropic.claude-3-5-sonnet-20241022-v2:0"
    @claude_3_5_haiku "anthropic.claude-3-5-haiku-20241022-v1:0"
    @claude_3_haiku "anthropic.claude-3-haiku-20240307-v1:0"
    
    def generate_text(prompt, options \\ %{}) do
      model_id = select_model(options)
      region = determine_optimal_bedrock_region()
      
      payload = %{
        "anthropic_version" => "bedrock-2023-05-31",
        "max_tokens" => Map.get(options, :max_tokens, 1024),
        "messages" => [
          %{"role" => "user", "content" => prompt}
        ],
        "temperature" => Map.get(options, :temperature, 0.7),
        "top_p" => Map.get(options, :top_p, 0.9),
        "top_k" => Map.get(options, :top_k, 250),
        "stop_sequences" => Map.get(options, :stop_sequences, [])
      }
      
      ExAws.Bedrock.invoke_model(
        model_id,
        payload,
        %{region: region}
      )
      |> ExAws.request()
      |> case do
        {:ok, response} ->
          # Cache response at edge if applicable
          cache_response_at_edge(model_id, prompt, response)
          parse_bedrock_response(response)
        error ->
          error
      end
    end
    
    defp determine_optimal_bedrock_region do
      # Use Cloudflare geolocation to determine optimal region
      case get_request_region() do
        "US" -> "us-east-1"
        "EU" -> "eu-west-1"
        _ -> "us-east-1"  # Default
      end
    end
  end
  ```

## Hybrid Integration Patterns

### 1. Edge-First Processing
```elixir
defmodule Aybiza.Hybrid.EdgeProcessor do
  @moduledoc """
  Edge-first processing with AWS fallback
  """
  
  def process_voice_request(request) do
    # Try edge processing first
    case process_at_edge(request) do
      {:ok, result} ->
        {:ok, result}
      
      {:error, :edge_unavailable} ->
        # Fallback to AWS processing
        process_in_aws(request)
    end
  end
  
  defp process_at_edge(request) do
    # Use Cloudflare Workers for initial processing
    CloudflareWorkers.process_voice(request)
  end
  
  defp process_in_aws(request) do
    # Full AWS pipeline processing
    AWS.VoicePipeline.process(request)
  end
end
```

### 2. Intelligent Caching
```elixir
defmodule Aybiza.Hybrid.CacheStrategy do
  @moduledoc """
  Hybrid caching strategy using Cloudflare and AWS
  """
  
  def get_cached_response(key) do
    # Try Cloudflare KV first (edge cache)
    case CloudflareKV.get(key) do
      {:ok, value} -> {:ok, value}
      {:error, :not_found} ->
        # Try Redis (regional cache)
        case Redis.get(key) do
          {:ok, value} ->
            # Populate edge cache
            CloudflareKV.put(key, value, ttl: 300)
            {:ok, value}
          {:error, :not_found} ->
            {:error, :not_found}
        end
    end
  end
end
```

### 3. Global Load Balancing
```elixir
defmodule Aybiza.Hybrid.LoadBalancer do
  @moduledoc """
  Global load balancing using Cloudflare and AWS
  """
  
  def route_request(request) do
    # Use Cloudflare's geographic information
    country = get_request_country(request)
    health_status = check_regional_health()
    
    target_region = determine_target_region(country, health_status)
    route_to_region(request, target_region)
  end
  
  defp determine_target_region(country, health_status) do
    primary_region = get_primary_region_for_country(country)
    
    if health_status[primary_region] == :healthy do
      primary_region
    else
      get_fallback_region(primary_region)
    end
  end
end
```

## Configuration Management

### Cloudflare Configuration
```elixir
config :aybiza, :cloudflare,
  api_token: {:system, "CLOUDFLARE_API_TOKEN"},
  zone_id: {:system, "CLOUDFLARE_ZONE_ID"},
  account_id: {:system, "CLOUDFLARE_ACCOUNT_ID"},
  
  # DNS configuration
  dns: %{
    main_domain: "aybiza.com",
    api_subdomain: "api.aybiza.com",
    regional_subdomains: %{
      us: "us.api.aybiza.com",
      eu: "eu.api.aybiza.com"
    }
  },
  
  # Worker configuration
  workers: %{
    voice_router: "voice-router-worker",
    auth_validator: "auth-validator-worker"
  },
  
  # WAF configuration
  waf: %{
    ruleset_id: {:system, "CLOUDFLARE_WAF_RULESET_ID"},
    custom_rules_enabled: true
  }
```

### AWS Integration Configuration
```elixir
config :ex_aws,
  debug_requests: System.get_env("AWS_DEBUG_REQUESTS") == "true",
  access_key_id: {:system, "AWS_ACCESS_KEY_ID"},
  secret_access_key: {:system, "AWS_SECRET_ACCESS_KEY"},
  region: {:system, "AWS_REGION"}

# Enhanced configuration for hybrid architecture
config :aybiza, :hybrid_integration,
  edge_processing_enabled: true,
  fallback_to_aws: true,
  cache_strategy: :edge_first,
  
  # Regional configuration
  regions: %{
    us_east_1: %{
      alb_endpoint: "us-east-1-alb.internal.aybiza.com",
      bedrock_region: "us-east-1",
      health_check_endpoint: "/health"
    },
    eu_west_2: %{
      alb_endpoint: "eu-west-2-alb.internal.aybiza.com", 
      bedrock_region: "eu-west-1",
      health_check_endpoint: "/health"
    }
  }
```

## Monitoring and Observability

### Hybrid Monitoring Strategy
```elixir
defmodule Aybiza.Hybrid.Monitoring do
  @moduledoc """
  Monitoring across Cloudflare and AWS
  """
  
  def collect_metrics do
    %{
      cloudflare: collect_cloudflare_metrics(),
      aws: collect_aws_metrics(),
      hybrid: collect_hybrid_metrics()
    }
  end
  
  defp collect_cloudflare_metrics do
    %{
      edge_requests: CloudflareAnalytics.get_requests(),
      cache_hit_ratio: CloudflareAnalytics.get_cache_ratio(),
      security_events: CloudflareAnalytics.get_security_events(),
      worker_executions: CloudflareAnalytics.get_worker_metrics()
    }
  end
  
  defp collect_aws_metrics do
    %{
      ecs_metrics: CloudWatch.get_ecs_metrics(),
      alb_metrics: CloudWatch.get_alb_metrics(),
      bedrock_metrics: CloudWatch.get_bedrock_metrics(),
      rds_metrics: CloudWatch.get_rds_metrics()
    }
  end
  
  defp collect_hybrid_metrics do
    %{
      end_to_end_latency: measure_end_to_end_latency(),
      failover_events: count_failover_events(),
      cache_efficiency: calculate_cache_efficiency()
    }
  end
end
```

## Cost Optimization

### Hybrid Cost Strategy
```elixir
defmodule Aybiza.Hybrid.CostOptimizer do
  @moduledoc """
  Cost optimization across Cloudflare and AWS
  """
  
  def optimize_costs do
    %{
      edge_optimization: optimize_edge_costs(),
      aws_optimization: optimize_aws_costs(),
      traffic_optimization: optimize_traffic_routing()
    }
  end
  
  defp optimize_edge_costs do
    %{
      worker_optimization: "Use Workers for compute-heavy tasks",
      cache_optimization: "Maximize cache hit ratios",
      bandwidth_optimization: "Use Argo Smart Routing"
    }
  end
  
  defp optimize_aws_costs do
    %{
      compute_optimization: "Use Fargate Spot for dev/staging",
      storage_optimization: "Implement S3 lifecycle policies",
      data_transfer_optimization: "Minimize cross-region transfer"
    }
  end
end
```

## Deployment Strategy

### Container Deployment Architecture

The AYBIZA platform uses **Amazon ECR (Elastic Container Registry)** as the primary container registry, following the **Git → CodeBuild → ECR → CodeDeploy → ECS/Fargate** pipeline pattern for enterprise-grade deployments:

```elixir
defmodule Aybiza.Deployment.ContainerStrategy do
  @moduledoc """
  Hybrid deployment strategy using AWS CodePipeline + Cloudflare Workers
  """
  
  def deployment_pipeline do
    %{
      container_registry: "Amazon ECR",
      pipeline_approach: "AWS CodePipeline (Production) + GitHub Actions (Development)",
      deployment_method: "CodeDeploy Blue/Green to ECS Fargate",
      edge_deployment: "Cloudflare Workers via Wrangler",
      
      regions: %{
        primary: "us-east-1",
        secondary: ["eu-west-2", "us-west-2"]
      },
      
      container_repositories: %{
        voice_gateway: "123456789012.dkr.ecr.us-east-1.amazonaws.com/aybiza/voice-gateway",
        voice_pipeline: "123456789012.dkr.ecr.us-east-1.amazonaws.com/aybiza/voice-pipeline",
        conversation_engine: "123456789012.dkr.ecr.us-east-1.amazonaws.com/aybiza/conversation-engine",
        agent_manager: "123456789012.dkr.ecr.us-east-1.amazonaws.com/aybiza/agent-manager"
      }
    }
  end
  
  def deploy_hybrid_architecture(environment) do
    %{
      backend_deployment: deploy_aws_containers(environment),
      edge_deployment: deploy_cloudflare_workers(environment),
      integration: configure_hybrid_integration(environment)
    }
  end
  
  defp deploy_aws_containers(environment) do
    case environment do
      :development ->
        # GitHub Actions direct deployment for speed
        deploy_via_github_actions()
      
      :staging ->
        # CodePipeline with basic blue/green
        deploy_via_codepipeline(:staging)
      
      :production ->
        # Full CodePipeline with multi-region blue/green
        deploy_via_codepipeline(:production)
    end
  end
  
  defp deploy_cloudflare_workers(environment) do
    # Deploy Workers to complement AWS backend
    %{
      voice_router: deploy_worker("voice-router", environment),
      auth_validator: deploy_worker("auth-validator", environment),
      edge_processor: deploy_worker("edge-processor", environment)
    }
  end
end
```

### Phase 1: Cloudflare Edge Deployment (Months 1-2)
1. Set up Cloudflare DNS and CDN
2. Deploy basic edge routing with Workers
3. Configure WAF and security rules
4. Implement SSL/TLS termination
5. **Set up Cloudflare Workers deployment pipeline**

### Phase 2: AWS Container Infrastructure (Months 2-3)
1. **Set up Amazon ECR repositories** across regions
2. **Configure AWS CodePipeline** for Git → CodeBuild → ECR → CodeDeploy → ECS flow
3. Deploy AWS infrastructure (ECS Fargate, RDS, etc.)
4. Configure Application Load Balancers
5. Set up origin servers in Cloudflare
6. Implement health checks and monitoring
7. **Configure CodeDeploy blue/green deployments**

### Phase 3: Hybrid Pipeline Integration (Months 3-4)
1. **Integrate Cloudflare Workers deployment with AWS CodePipeline**
2. Optimize edge caching strategies
3. Implement intelligent routing between edge and AWS backend
4. Deploy edge processing capabilities
5. Fine-tune performance and costs
6. **Set up multi-region container deployment automation**

### Hybrid Deployment Configuration

```elixir
# config/hybrid_deployment.exs
config :aybiza, :hybrid_deployment,
  # AWS Container Deployment
  aws_pipeline: %{
    codepipeline_name: "aybiza-container-pipeline",
    codebuild_project: "aybiza-container-build",
    ecr_repositories: [
      "aybiza/voice-gateway",
      "aybiza/voice-pipeline", 
      "aybiza/conversation-engine",
      "aybiza/agent-manager"
    ],
    ecs_clusters: %{
      development: "aybiza-dev-cluster",
      staging: "aybiza-staging-cluster",
      production: "aybiza-production-cluster"
    },
    codedeploy_applications: %{
      staging: "aybiza-staging-app",
      production: "aybiza-production-app"
    }
  },
  
  # Cloudflare Workers Deployment
  cloudflare_workers: %{
    deployment_method: "wrangler",
    workers: [
      "voice-router",
      "auth-validator",
      "edge-processor"
    ],
    kv_namespaces: [
      "aybiza-cache",
      "aybiza-config"
    ]
  },
  
  # Integration Points
  integration: %{
    workers_to_aws: "Direct ALB integration",
    aws_to_workers: "EventBridge + API Gateway",
    shared_secrets: "AWS Secrets Manager + Cloudflare Env Vars"
  }
```

### Container Security Integration

```elixir
defmodule Aybiza.Security.HybridContainerSecurity do
  @moduledoc """
  Security configuration for hybrid container deployment
  """
  
  def security_configuration do
    %{
      # ECR Security
      ecr_scanning: %{
        scan_on_push: true,
        vulnerability_scanning: "Enhanced (Inspector integration)",
        compliance_scanning: "CIS, NIST frameworks"
      },
      
      # CodePipeline Security
      pipeline_security: %{
        artifact_encryption: "AWS KMS encryption",
        iam_roles: "Least privilege service roles",
        secret_management: "AWS Secrets Manager integration"
      },
      
      # Cloudflare Workers Security
      workers_security: %{
        origin_ca_certificates: "Cloudflare Origin CA",
        authenticated_origin_pulls: true,
        worker_environment_variables: "Encrypted at rest and in transit"
      },
      
      # Cross-Platform Security
      hybrid_security: %{
        mtls_between_services: true,
        api_authentication: "JWT + Cloudflare Access",
        network_security: "VPC + Cloudflare Zero Trust"
      }
    }
  end
end
```

## Conclusion

This Cloudflare+AWS hybrid architecture provides AYBIZA with the best of both worlds: Cloudflare's global edge network for ultra-low latency and security, combined with AWS's comprehensive cloud services for scalable backend processing. The hybrid approach ensures optimal performance, security, and cost efficiency while maintaining the flexibility to adapt to changing requirements.
```