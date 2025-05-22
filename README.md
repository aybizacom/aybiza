
# AYBIZA AI Voice Agent Platform

## Overview

AYBIZA is an enterprise-grade AI voice agent platform built with a hybrid edge-cloud architecture, empowering businesses to create, deploy, and manage AI-powered voice agents that handle phone calls with near-human conversation capabilities. Our platform combines Cloudflare's global edge network with AWS backend services to process speech in real-time, generate natural AI responses, and support complex automation workflows.

## Key Features

- **Ultra-Low Latency**: End-to-end voice processing in under 50ms via edge optimization
- **Global Scale**: Built on Cloudflare's 300+ edge locations and AWS multi-region architecture
- **Flexible Phone Number Integration**: Support for managed numbers, SIP trunks, and Twilio accounts
- **Enterprise Security**: Comprehensive security with edge-based DDoS protection and compliance features
- **Intelligent Conversations**: Powered by state-of-the-art LLMs via Amazon Bedrock (Claude 3.7, 3.5, 3)
- **High-Quality Voice**: Natural-sounding speech using Deepgram's TTS technology
- **Edge-First Architecture**: Cloudflare Workers for intelligent routing and preprocessing
- **Future Automation Framework**: Extensible design for no-code automation workflows

## Hybrid Architecture

AYBIZA leverages a cutting-edge hybrid architecture combining the best of edge computing and cloud services:

### Edge Layer (Cloudflare)
- **Global DNS & CDN**: Anycast DNS with intelligent geographic routing
- **Edge Computing**: Cloudflare Workers for preprocessing and optimal routing
- **Security**: WAF, DDoS protection (10+ Tbps), and bot management
- **Caching**: Intelligent caching for agent configurations and responses

### Backend Layer (AWS)
- **Compute**: ECS Fargate with multi-region deployment
- **AI Processing**: Amazon Bedrock with Claude models across regions
- **Data Storage**: Aurora PostgreSQL with TimescaleDB + ElastiCache Redis
- **Voice Processing**: Deepgram integration for STT/TTS

## Technology Stack

### Edge Infrastructure
- **Edge Computing**: Cloudflare Workers (JavaScript)
- **Edge Storage**: Cloudflare KV for caching
- **Edge Security**: Cloudflare WAF, DDoS protection, bot management
- **DNS & CDN**: Cloudflare global network (300+ locations)

### Backend Infrastructure
- **Runtime**: Elixir 1.18.3/Erlang OTP 27.3.4
- **Web Framework**: Phoenix 1.7.21
- **Database**: PostgreSQL 16.9 with TimescaleDB 2.20.0
- **Media Processing**: Membrane Framework 0.11.0
- **Voice Services**: Deepgram API v1
- **LLM Provider**: Amazon Bedrock (Claude 3.7 Sonnet, 3.5 Sonnet v2, 3.5 Haiku, 3 Haiku)
- **Call Provider**: Twilio API (2010-04-01)
- **Infrastructure**: AWS (ECS, Aurora, ElastiCache, Bedrock)
- **Containerization**: Docker
- **CI/CD**: GitHub Actions

## Project Structure

```
/aybiza
  /cloudflare-workers       # Edge computing functions
    /voice-router          # Intelligent request routing
    /cache-manager         # Edge caching logic
    /auth-validator        # Edge authentication
  /apps
    /voice_gateway         # WebSocket and call routing
    /voice_pipeline        # Audio processing pipeline
    /conversation_engine   # Conversation management
    /agent_manager         # Agent configuration
    /phone_number_manager  # Flexible phone number management
    /call_analytics        # Call data and analytics
    /security              # Security and compliance
  /config                  # Shared configuration
  /deps                    # Dependencies
  /docker                  # Docker configuration
  /docs                    # Documentation
  /priv                    # Private assets
  /terraform               # Infrastructure as code
```

## Getting Started

### Prerequisites

- Elixir 1.18.3+ and Erlang/OTP 27.3.4+
- Docker and Docker Compose
- PostgreSQL 16.9+ with TimescaleDB extension
- Cloudflare account (Pro plan recommended)
- AWS account with appropriate services
- Twilio account with voice capabilities
- Deepgram API key
- Amazon Bedrock access with Claude models enabled

### Local Development Setup

1. Clone the repository:
   ```bash
   git clone git@github.com:aybizacom/aybiza.git
   cd aybiza
   ```

2. Create a `.env` file with required environment variables:
   ```
   # Cloudflare Configuration
   CLOUDFLARE_API_TOKEN=your_cloudflare_api_token
   CLOUDFLARE_ACCOUNT_ID=your_cloudflare_account_id
   CLOUDFLARE_ZONE_ID=your_cloudflare_zone_id
   
   # Twilio Configuration
   TWILIO_ACCOUNT_SID=your_twilio_account_sid
   TWILIO_AUTH_TOKEN=your_twilio_auth_token
   
   # Deepgram Configuration
   DEEPGRAM_API_KEY=your_deepgram_api_key
   
   # AWS Configuration (for Bedrock)
   AWS_ACCESS_KEY_ID=your_aws_access_key_id
   AWS_SECRET_ACCESS_KEY=your_aws_secret_access_key
   AWS_REGION=us-east-1
   
   # Application Configuration
   SECRET_KEY_BASE=a_long_secret_key_base
   PHX_HOST=localhost
   PORT=4000
   ```

3. Set up development infrastructure:
   ```bash
   # Start local services
   docker-compose up -d
   
   # Install dependencies
   mix deps.get
   
   # Set up database
   mix ecto.setup
   
   # Compile assets
   mix assets.deploy
   ```

4. Deploy Cloudflare Workers (development):
   ```bash
   cd cloudflare-workers
   npm install
   wrangler publish --env development
   ```

5. Start the application:
   ```bash
   mix phx.server
   ```

6. Access the application:
   - Local server: http://localhost:4000
   - Edge endpoint: https://dev.aybiza.com (after Cloudflare setup)

### Working with the Hybrid Architecture

The AYBIZA platform is structured as a hybrid system with edge and backend components:

#### Edge Components (Cloudflare)
- **Voice Router**: Intelligently routes calls to optimal AWS regions
- **Cache Manager**: Manages edge caching for agent configs and responses
- **Auth Validator**: Handles authentication and validation at the edge
- **Analytics Collector**: Collects performance metrics globally

#### Backend Components (AWS)
- **voice_gateway**: Handles WebSocket connections with Twilio
- **voice_pipeline**: Processes audio streams and manages the STT/LLM/TTS pipeline
- **conversation_engine**: Manages conversation state and flow
- **agent_manager**: Stores and retrieves agent configurations
- **call_analytics**: Captures and analyzes call data
- **security**: Implements security controls and compliance requirements

## Architecture Overview

### Hybrid Request Flow

```
Incoming call → Twilio → Cloudflare Edge → Optimal AWS Region Selection →
  Voice Gateway → Audio Processing → LLM (Bedrock) → Response Generation →
    Edge Caching → Twilio → Caller
```

### Edge-First Processing

```
Request → Cloudflare Worker → Edge Cache Check →
  [Cache Hit] → Return Cached Response (< 10ms)
  [Cache Miss] → Route to AWS → Process → Cache at Edge → Return Response
```

### Multi-Region Backend

```
Cloudflare Edge (Global)
├── US East (Primary)
│   ├── ECS Fargate Cluster
│   ├── Aurora PostgreSQL
│   ├── ElastiCache Redis
│   └── Bedrock (Claude 3.7/3.5)
├── EU West (Primary) 
│   ├── ECS Fargate Cluster
│   ├── Aurora PostgreSQL
│   └── Bedrock (Claude 3.5)
└── US West (Secondary)
    ├── ECS Fargate Cluster
    └── Bedrock (Claude 3.5 Haiku)
```

## Documentation

Comprehensive documentation is available in the docs directory. For a complete guide, see [docs/00_documentation_index.md](docs/00_documentation_index.md).

### Quick Navigation

#### Project Foundation
- [docs/01_initial_prompt.md](docs/01_initial_prompt.md) - Project overview and initial requirements
- [docs/02_technical_requirements.md](docs/02_technical_requirements.md) - Detailed technical specifications
- [docs/03_system_architecture.md](docs/03_system_architecture.md) - Hybrid system architecture
- [docs/04_database_schema.md](docs/04_database_schema.md) - Database design and entity relationships

#### Infrastructure & Integration
- [docs/06_aws_integration_documentation.md](docs/06_aws_integration_documentation.md) - Cloudflare+AWS hybrid integration
- [docs/23_global_edge_computing_architecture.md](docs/23_global_edge_computing_architecture.md) - Global edge architecture
- [docs/50_aws_deployment_strategy.md](docs/50_aws_deployment_strategy.md) - Hybrid deployment strategy

#### Voice Pipeline
- [docs/08_voice_pipeline_architecture.md](docs/08_voice_pipeline_architecture.md) - Voice processing with edge optimization

#### Security & Compliance
- [docs/09_security_compliance_implementation.md](docs/09_security_compliance_implementation.md) - Security architecture

## Development Workflow

We use GitHub Actions with enhanced Cloudflare integration:

1. Create a feature branch from `main`
2. Implement changes with appropriate tests
3. Deploy and test Cloudflare Workers locally
4. Submit a merge request to `main`
5. CI pipeline runs tests, builds containers, and deploys to edge + AWS
6. Automatic deployment to development environment
7. Manual promotion to staging and production

### CI/CD Pipeline Enhancement

```yaml
stages:
  - test
  - build
  - deploy-aws
  - deploy-cloudflare
  - verify

deploy-cloudflare:
  stage: deploy-cloudflare
  script:
    - wrangler publish --env $ENVIRONMENT
    - cloudflare-cli dns update --record api --content $NEW_ORIGIN_IP
```

## Testing

The platform includes comprehensive testing across edge and backend:

- **Unit Tests**: Test individual components in isolation
- **Integration Tests**: Test component interactions
- **Edge Tests**: Test Cloudflare Worker functionality
- **End-to-End Tests**: Test complete workflows with external services
- **Performance Tests**: Validate latency and throughput requirements
- **Multi-Region Tests**: Test failover and geographic routing

Run tests locally:

```bash
# Backend tests
mix test

# Edge tests
cd cloudflare-workers
npm test

# Integration tests
mix test --only integration

# Performance tests
mix test --only performance
```

## Security

Security is implemented at both edge and backend layers:

### Edge Security (Cloudflare)
- DDoS protection with 10+ Tbps capacity
- Web Application Firewall with custom rules
- Bot management and threat intelligence
- SSL/TLS termination with modern cipher suites
- Rate limiting and geographic blocking

### Backend Security (AWS)
- End-to-end encryption for all communications
- Secure credential management with AWS Secrets Manager
- Comprehensive audit logging and monitoring
- Multi-tenant isolation and data protection
- Input validation and sanitization
- VPC with private subnets and security groups

### Compliance Features
- GDPR compliance with EU data residency
- SOC 2 Type II controls
- HIPAA-ready architecture
- PCI DSS compliance for payment processing
- Audit trails and compliance reporting

## Performance & Scalability

### Performance Targets
- **Edge Response Time**: < 10ms for cached requests
- **US/UK Markets**: < 50ms end-to-end latency
- **Global Markets**: < 100ms end-to-end latency
- **Voice Processing**: < 200ms total pipeline latency
- **Availability**: 99.99% global availability

### Scalability Features
- **Auto-scaling**: ECS Fargate with predictive scaling
- **Global Edge**: 300+ Cloudflare locations worldwide
- **Multi-Region**: Active-active deployment across AWS regions
- **Database Scaling**: Aurora PostgreSQL with read replicas
- **Cache Scaling**: ElastiCache Redis with cluster mode

## Cost Optimization

The hybrid architecture delivers significant cost savings:

```yaml
Monthly Cost Comparison:
  AWS-Only Architecture: $8,500
  Cloudflare+AWS Hybrid: $6,070
  Monthly Savings: $2,430 (28.6%)

Cost Per Call:
  Hybrid Architecture: $0.006
  AWS-Only: $0.0085
  Savings: $0.0025 per call
```

### Cost Optimization Features
- Edge caching reduces origin requests by 50-70%
- Geographic routing minimizes data transfer costs
- Intelligent scaling based on demand patterns
- Reserved capacity for predictable workloads
- Spot instances for non-critical environments

## Monitoring & Observability

### Hybrid Monitoring Stack
- **Edge Monitoring**: Cloudflare Analytics and Real User Monitoring
- **Backend Monitoring**: AWS CloudWatch, X-Ray, and Container Insights
- **Unified Dashboard**: End-to-end request tracing and performance metrics
- **Alerting**: Multi-layer alerting for edge and backend issues

### Key Metrics
- Global request distribution and latency heatmaps
- Cache hit ratios and edge performance
- Voice processing pipeline metrics
- Cost optimization and budget tracking
- Security events and threat analysis

## License

Proprietary - All rights reserved

## Contact

For questions or support, contact the development team at info@aybiza.com.

## Contributing

This is a private repository. For development team members:

1. Follow the coding standards in [docs/29_elixir_best_practices_guide.md](docs/29_elixir_best_practices_guide.md)
2. Create comprehensive tests for all functionality
3. Document all modules and functions with typespecs
4. Follow security guidelines and compliance requirements
5. Test both edge and backend components thoroughly

## Deployment Environments

- **Development**: https://dev.aybiza.com
- **Staging**: https://staging.aybiza.com  
- **Production**: https://api.aybiza.com

All environments use the hybrid Cloudflare+AWS architecture with appropriate scaling and security configurations.