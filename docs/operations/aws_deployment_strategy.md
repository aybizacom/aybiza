# Enhanced AWS Container Deployment Strategy for AYBIZA

This document outlines the comprehensive strategy for deploying AYBIZA AI Voice Agent Platform using an enhanced AWS container deployment pipeline with Cloudflare+AWS hybrid architecture.

## Architecture Overview

AYBIZA will be deployed using a hybrid edge-cloud architecture combining Cloudflare's global edge network with AWS native container deployment services:

```
GitHub → GitHub Actions → AWS CodeBuild → Amazon ECR → AWS CodeDeploy → Amazon ECS Fargate
                            ↓                          ↓                    ↓
                    Container Security Scan    Multi-Region Replication   Blue/Green Deploy
                            ↓                          ↓                    ↓
                    Cloudflare Edge → Geographic Routing → AWS Regional Backend
                            ↓
                Aurora PostgreSQL + ElastiCache Redis + AWS Bedrock (Claude)
```

## Enhanced Container Deployment Pipeline

### Core AWS Services Integration

#### 1. Amazon ECR (Elastic Container Registry)
**Primary container registry for production workloads**

**Key Benefits:**
- Native AWS integration with ECS/Fargate
- Advanced security with vulnerability scanning
- Multi-region image replication
- Immutable image tags
- Geographic distribution for faster pulls

**Multi-Region Configuration:**
```yaml
ECR Repositories:
  us-east-1 (Primary):
    - aybiza/voice-gateway
    - aybiza/voice-pipeline
    - aybiza/conversation-engine
    - aybiza/agent-manager
    - aybiza/phone-manager
    - aybiza/call-analytics
    
  Regional Replicas:
    eu-west-2: Cross-region replication enabled
    us-west-2: Cross-region replication enabled
    ap-southeast-1: Cross-region replication enabled
```

#### 2. AWS CodeBuild
**Enhanced container building service following the Git → CodeBuild → ECR pattern**

**Key Features:**
- Privileged Docker builds for container creation
- Native ECR integration for seamless image push
- Build environment optimization for faster builds
- Parallel builds across regions for global deployment
- Advanced caching strategies for dependency management
- Automated security scanning integration
- Multi-architecture builds (ARM64/x86_64)

**Enhanced Build Specification (buildspec.yml):**
```yaml
version: 0.2

env:
  variables:
    IMAGE_REPO_NAME: aybiza
    IMAGE_TAG: latest
    AWS_DEFAULT_REGION: us-east-1
    
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
      
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -f Dockerfile.prod -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $REPOSITORY_URI:$IMAGE_TAG
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $REPOSITORY_URI:latest
      
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:latest
      - echo Writing image definitions file...
      - printf '[{"name":"aybiza-container","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
      
artifacts:
  files:
    - imagedefinitions.json
    - appspec.yml
```

#### 3. AWS CodeDeploy
**Blue/Green deployment service**

**Deployment Configuration:**
- Blue/Green deployments with zero downtime
- Automatic rollback on failure detection
- Health check integration
- Traffic shifting strategies
- Automated testing integration

**Application Specification (appspec.yml):**
```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "aybiza-container"
          ContainerPort: 4000
        PlatformVersion: "LATEST"
        NetworkConfiguration:
          AwsvpcConfiguration:
            Subnets: ["subnet-12345", "subnet-67890"]
            SecurityGroups: ["sg-12345"]
            AssignPublicIp: "ENABLED"
            
Hooks:
  - BeforeInstall:
      - location: scripts/stop_application.sh
        timeout: 300
        runas: root
  - ApplicationStart:
      - location: scripts/start_application.sh
        timeout: 300
        runas: root
  - ApplicationStop:
      - location: scripts/validate_deployment.sh
        timeout: 300
        runas: root
  - ValidateService:
      - location: scripts/validate_service.sh
        timeout: 300
        runas: root
```

#### 4. AWS CodePipeline
**Complete CI/CD orchestration following the native AWS pipeline pattern**

**Pipeline Architecture (Git → CodeBuild → ECR → CodeDeploy → ECS/Fargate):**
```
Git Repository (Source)
        ↓
   AWS CodePipeline (Orchestrator)
        ↓
   AWS CodeBuild (Build & Push to ECR)
        ↓
   Amazon ECR (Container Registry)
        ↓
   AWS CodeDeploy (Blue/Green Deployment)
        ↓
   Amazon ECS/Fargate (Container Runtime)
        ↓
   Application Load Balancer (Traffic Distribution)
```

**Pipeline Stages:**
1. **Source**: Git repository integration with automatic triggers
2. **Build**: CodeBuild with Docker builds and ECR push
3. **Test**: Automated testing and security scanning
4. **Deploy to Staging**: CodeDeploy to staging environment
5. **Manual Approval**: Human approval gate for production
6. **Deploy to Production**: CodeDeploy to production with blue/green
7. **Post-Deploy**: Monitoring and validation

**CodePipeline Configuration:**
**Artifact Management Strategy:**
```yaml
Artifact Flow:
  Source Stage:
    Output: SourceOutput (source code, Dockerfile, configs)
    Location: S3://aybiza-pipeline-artifacts/source/
    
  Build Stage:
    Input: SourceOutput
    Output: BuildOutput (container image metadata, deployment configs)
    Contents:
      - imagedefinitions.json (ECS image definitions)
      - appspec.yml (CodeDeploy application specification)
      - taskdef.json (ECS task definition template)
      - imageDetail.json (ECR image URI and metadata)
    Location: S3://aybiza-pipeline-artifacts/build/
    
  Deploy Stages:
    Input: BuildOutput
    Process: CodeDeploy reads artifacts to perform blue/green deployment
    Validation: Health checks and rollback triggers
```

**Pipeline Stage Configuration:**
```yaml
Pipeline Stages:
  1. Source:
     Provider: GitHub
     Trigger: Webhook on push to main/staging/production branches
     Output: Source code and configuration files
     
  2. Security-Scan:
     Provider: CodeBuild
     Purpose: Vulnerability scanning and compliance checks
     Blockers: Critical/High vulnerabilities prevent progression
     
  3. Build:
     Provider: CodeBuild  
     Purpose: Docker build and push to ECR
     Output: Container image + deployment artifacts
     
  4. Deploy-Staging:
     Provider: CodeDeploy to ECS
     Strategy: Blue/Green with 10% canary traffic
     Validation: Automated health checks
     
  5. Integration-Tests:
     Provider: CodeBuild
     Purpose: End-to-end testing against staging
     
  6. Manual-Approval:
     Provider: Manual approval gate
     Reviewers: DevOps team + Product owner
     
  7. Deploy-Production:
     Provider: CodeDeploy to ECS (Multi-region)
     Strategy: Blue/Green with 10% canary → 50% → 100%
     Regions: us-east-1, eu-west-2, us-west-2
     
  8. Post-Deploy-Validation:
     Provider: CodeBuild
     Purpose: Production smoke tests and monitoring setup
```

## Container Security and Compliance

### Image Vulnerability Scanning
```yaml
ECR Scan Configuration:
  Scan on Push: Enabled
  Enhanced Scanning: Enabled (Inspector integration)
  CVE Database: Continuously updated
  Scan Results: Integration with AWS Security Hub
  
Security Policies:
  - Block deployment of HIGH/CRITICAL vulnerabilities
  - Automated patching recommendations
  - Compliance reporting (SOC 2, HIPAA)
  - Image signing with AWS Signer
```

### Security Scanning Pipeline
```yaml
Security Gates:
  1. Static Code Analysis (SonarQube/CodeQL)
  2. Dependency Vulnerability Scan (Snyk/OWASP)
  3. Container Image Scan (ECR Inspector)
  4. Infrastructure Scan (Checkov/Terrascan)
  5. Runtime Security (Falco/Twistlock)
```

## Multi-Region Deployment Strategy

### Regional Container Distribution
```yaml
Primary Regions:
  us-east-1:
    Role: Primary production region
    ECR: Master repository
    ECS: Production clusters (2+ instances)
    CodeDeploy: Blue/green deployment
    
  eu-west-2:
    Role: European production region
    ECR: Replicated from us-east-1
    ECS: Production clusters (2+ instances)
    CodeDeploy: Blue/green deployment
    
  us-west-2:
    Role: Secondary/failover region
    ECR: Replicated from us-east-1
    ECS: Standby clusters (1 instance)
    CodeDeploy: Blue/green deployment

Cross-Region Replication:
  Schedule: Real-time replication
  Latency: < 1 minute for image availability
  Failover: Automatic region switching
  Consistency: Eventually consistent across regions
```

## Environment-Specific Configurations

### Multi-Branch Pipeline Strategy

#### Development Environment (main branch)
```yaml
Pipeline Configuration:
  Trigger: Push to main branch
  Stages:
    1. Source (GitHub main branch)
    2. Security Scan (basic vulnerability check)
    3. Build (CodeBuild → ECR)
    4. Deploy (direct ECS deployment)
    5. Basic Tests (health check only)
  
Infrastructure:
  - Container Registry: Amazon ECR (development repo)
  - Deployment: Direct ECS task definition update
  - Region: Single region (us-east-1)
  - Scaling: Fixed 1 instance
  - Security: Basic vulnerability scanning
  - Environment Variables: Development configs
```

#### Staging Environment (staging branch)
```yaml
Pipeline Configuration:
  Trigger: Push to staging branch OR promotion from main
  Stages:
    1. Source (GitHub staging branch)
    2. Security Scan (comprehensive vulnerability + compliance)
    3. Build (CodeBuild → ECR with staging tags)
    4. Deploy-Staging (CodeDeploy blue/green)
    5. Integration Tests (full test suite)
    6. Performance Tests (load testing)
  
Infrastructure:
  - Container Registry: Amazon ECR with staging tags
  - Deployment: CodeDeploy with basic blue/green (30% → 100%)
  - Region: Primary region + 1 replica (us-east-1, us-west-2)
  - Scaling: 1-3 instances with basic auto-scaling
  - Security: Full vulnerability scanning + compliance checks
  - Environment Variables: Staging configs with production-like data
```

#### Production Environment (production branch)
```yaml
Pipeline Configuration:
  Trigger: Push to production branch OR manual promotion from staging
  Stages:
    1. Source (GitHub production branch)
    2. Security Scan (comprehensive + additional compliance)
    3. Build (CodeBuild → ECR with production tags)
    4. Deploy-Staging-Validation (re-validate in staging)
    5. Manual Approval (DevOps + Product approval)
    6. Deploy-Production-Primary (us-east-1)
    7. Deploy-Production-Secondary (eu-west-2, us-west-2)
    8. Post-Deploy-Validation (smoke tests across regions)
    9. Monitoring-Setup (alerts and dashboards)
  
Infrastructure:
  - Container Registry: Amazon ECR with multi-region replication
  - Deployment: CodeDeploy with advanced blue/green + canary (10% → 50% → 100%)
  - Region: Multi-region (us-east-1, eu-west-2, us-west-2)
  - Scaling: 2-20 instances per region with advanced auto-scaling
  - Security: Full security pipeline + runtime monitoring + compliance reporting
  - Environment Variables: Production configs with encrypted secrets
```

#### Branch Protection and Promotion Strategy
```yaml
Branch Protection Rules:
  main:
    - Require pull request reviews (1 reviewer)
    - Require status checks (tests, security scan)
    - No direct pushes to main
    
  staging:
    - Require pull request reviews (2 reviewers)
    - Require status checks (comprehensive tests)
    - Only accept merges from main or hotfix branches
    
  production:
    - Require pull request reviews (3 reviewers including DevOps)
    - Require manual approval in pipeline
    - Only accept merges from staging branch
    - Require signed commits

Promotion Workflow:
  1. Feature Branch → main (via PR)
  2. main → staging (via PR + automated pipeline)
  3. staging → production (via PR + manual approval + pipeline)
  
Hotfix Workflow:
  1. hotfix/issue-name → staging (via emergency PR)
  2. staging → production (expedited approval)
  3. production → main (backport merge)
```

## Dual CI/CD Approach: AWS Native vs GitHub Actions

### Option 1: AWS Native CodePipeline (Recommended for Enterprise)

**Pure AWS Pipeline following the transcript pattern:**

```bash
# Create the pipeline using AWS CLI
aws codepipeline create-pipeline --cli-input-json file://pipeline-definition.json

# Set up CloudWatch Events to trigger pipeline on Git commits
aws events put-rule \
  --name "aybiza-pipeline-trigger" \
  --event-pattern '{"source":["aws.codecommit"],"detail-type":["CodeCommit Repository State Change"]}'

# Add pipeline as target
aws events put-targets \
  --rule "aybiza-pipeline-trigger" \
  --targets "Id"="1","Arn"="arn:aws:codepipeline:us-east-1:ACCOUNT-ID:pipeline/aybiza-container-pipeline"
```

**Benefits of AWS Native Approach:**
- Native AWS service integration
- Tighter security with IAM roles
- Built-in artifact management
- AWS native monitoring and logging
- Easier compliance auditing
- No external dependency on GitHub Actions

**Git Source Integration Options:**

**Option A: GitHub Integration (Recommended)**
```yaml
Source Configuration:
  Provider: GitHub
  Authentication: OAuth Token (stored in AWS Secrets Manager)
  Webhook Configuration:
    - Push events to main/staging/production branches
    - Pull request events for validation
    - Tag creation for release deployments
  
  Advantages:
    - No migration from current GitHub workflow
    - Rich GitHub ecosystem integration
    - Advanced PR review workflows
    - GitHub Actions can complement CodePipeline
```

**Option B: AWS CodeCommit Migration**
```yaml
Source Configuration:
  Provider: AWS CodeCommit
  Authentication: IAM roles and policies
  Event Configuration:
    - CloudWatch Events for push notifications
    - Native AWS integration
    - Cross-region replication
  
  Migration Strategy:
    1. Create CodeCommit repositories
    2. Migrate git history from GitHub
    3. Update pipeline source configuration
    4. Set up developer access and workflows
  
  Advantages:
    - Full AWS native integration
    - Enhanced security with IAM
    - No external dependencies
    - Simplified compliance auditing
```

**Git Trigger Configuration:**
```yaml
EventBridge Rules for Git Integration:
  Rule 1: Main Branch Deployment
    EventPattern:
      source: ["aws.codecommit"] # or GitHub webhook
      detail-type: ["CodeCommit Repository State Change"]
      detail:
        event: ["referenceCreated", "referenceUpdated"]
        referenceName: ["main"]
    Targets:
      - CodePipeline: aybiza-dev-pipeline
      
  Rule 2: Staging Branch Deployment  
    EventPattern:
      detail:
        referenceName: ["staging"]
    Targets:
      - CodePipeline: aybiza-staging-pipeline
      
  Rule 3: Production Branch Deployment
    EventPattern:
      detail:
        referenceName: ["production"]
    Targets:
      - CodePipeline: aybiza-production-pipeline

GitHub Webhook Alternative:
  Webhook URL: https://webhooks.amazonaws.com/trigger/pipeline
  Events: ["push", "pull_request"]
  Secret: Stored in AWS Secrets Manager
  Headers: GitHub signature validation
```

**Pipeline Automation Script:**
```bash
#!/bin/bash
# setup-aws-pipeline.sh - Enhanced with Git integration

# Variables
REGION="us-east-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
PIPELINE_NAME="aybiza-container-pipeline"
CODEBUILD_PROJECT="aybiza-container-build"

# Create S3 bucket for artifacts
aws s3 mb s3://aybiza-codepipeline-artifacts-${ACCOUNT_ID}

# Set up EventBridge rules for Git triggers
aws events put-rule \
  --name "aybiza-main-branch-trigger" \
  --event-pattern '{"source":["aws.codecommit"],"detail-type":["CodeCommit Repository State Change"],"detail":{"referenceName":["main"]}}' \
  --description "Trigger pipeline on main branch changes"

# Add pipeline as target
aws events put-targets \
  --rule "aybiza-main-branch-trigger" \
  --targets "Id"="1","Arn"="arn:aws:codepipeline:${REGION}:${ACCOUNT_ID}:pipeline/${PIPELINE_NAME}"

# Create CodeBuild project  
aws codebuild create-project --cli-input-json file://codebuild-project.json

# Create CodePipeline
aws codepipeline create-pipeline --cli-input-json file://pipeline-definition.json

# Set up CloudWatch monitoring
aws logs create-log-group --log-group-name "/aws/codepipeline/${PIPELINE_NAME}"

# Create CloudWatch dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "AYBIZA-Pipeline-Dashboard" \
  --dashboard-body file://dashboard-definition.json

echo "✅ AWS CodePipeline with Git integration setup complete!"
echo "Pipeline URL: https://console.aws.amazon.com/codesuite/codepipeline/pipelines/${PIPELINE_NAME}/view"
echo "EventBridge Rules: https://console.aws.amazon.com/events/home?region=${REGION}#/rules"
```

### Option 2: GitHub Actions Workflow (Current Implementation)

```yaml
name: Enhanced AWS Container Deployment Pipeline

on:
  push:
    branches: [main, staging, production]
  pull_request:
    branches: [main, staging, production]

env:
  AWS_DEFAULT_REGION: us-east-1
  ECR_REPOSITORY: aybiza
  ECS_SERVICE: aybiza-service
  ECS_CLUSTER: aybiza-cluster

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: timescale/timescaledb:2.13.0-pg15
        env:
          POSTGRES_DB: aybiza_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
    - uses: actions/checkout@v4
    - name: Set up Elixir
      uses: erlef/setup-beam@v1
      with:
        elixir-version: '1.18.3'
        otp-version: '27.3.4'
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          deps
          _build
        key: ${{ runner.os }}-mix-${{ hashFiles('**/mix.lock') }}
    - name: Install dependencies
      run: |
        mix local.hex --force
        mix local.rebar --force
        mix deps.get
    - name: Run tests
      env:
        DATABASE_URL: postgres://postgres:postgres@localhost:5432/aybiza_test
        REDIS_URL: redis://localhost:6379
        MIX_ENV: test
      run: |
        mix format --check-formatted
        mix credo --strict
        mix test
        mix sobelow --config

  security-scan:
    runs-on: ubuntu-latest
    needs: test
    steps:
    - uses: actions/checkout@v4
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  build:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/staging' || github.ref == 'refs/heads/production'
    
    outputs:
      image-uri: ${{ steps.build-image.outputs.image }}
      
    steps:
    - uses: actions/checkout@v4
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
        role-session-name: GitHubActions-${{ github.run_id }}
        aws-region: ${{ env.AWS_DEFAULT_REGION }}
        
    - name: Start CodeBuild
      uses: aws-actions/aws-codebuild-run-build@v1
      id: codebuild
      with:
        project-name: aybiza-container-build
        env-vars-for-codebuild: |
          IMAGE_TAG=${{ github.sha }}
          ENVIRONMENT=${{ github.ref_name }}
          
    - name: Get CodeBuild artifacts
      id: build-image
      run: |
        # Get the image URI from CodeBuild artifacts
        IMAGE_URI=$(aws codebuild batch-get-builds --ids ${{ steps.codebuild.outputs.aws-build-id }} --query 'builds[0].environment.environmentVariables[?name==`REPOSITORY_URI`].value' --output text):${{ github.sha }}
        echo "image=$IMAGE_URI" >> $GITHUB_OUTPUT

  scan-image:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
        aws-region: ${{ env.AWS_DEFAULT_REGION }}
        
    - name: Scan image for vulnerabilities
      run: |
        aws ecr start-image-scan --repository-name $ECR_REPOSITORY --image-id imageTag=${{ github.sha }}
        aws ecr wait image-scan-complete --repository-name $ECR_REPOSITORY --image-id imageTag=${{ github.sha }}
        
        # Get scan results
        SCAN_RESULTS=$(aws ecr describe-image-scan-findings --repository-name $ECR_REPOSITORY --image-id imageTag=${{ github.sha }})
        CRITICAL_COUNT=$(echo $SCAN_RESULTS | jq '.imageScanFindings.findingCounts.CRITICAL // 0')
        HIGH_COUNT=$(echo $SCAN_RESULTS | jq '.imageScanFindings.findingCounts.HIGH // 0')
        
        echo "Critical vulnerabilities: $CRITICAL_COUNT"
        echo "High vulnerabilities: $HIGH_COUNT"
        
        # Fail if critical vulnerabilities found
        if [ $CRITICAL_COUNT -gt 0 ]; then
          echo "❌ Critical vulnerabilities found. Blocking deployment."
          exit 1
        fi

  deploy-staging:
    needs: [build, scan-image]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/staging' || github.ref == 'refs/heads/production'
    environment: staging
    
    steps:
    - uses: actions/checkout@v4
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
        aws-region: ${{ env.AWS_DEFAULT_REGION }}
        
    - name: Deploy to staging with CodeDeploy
      run: |
        # Update task definition with new image
        TASK_DEFINITION=$(aws ecs describe-task-definition --task-definition aybiza-staging --query taskDefinition)
        NEW_TASK_DEFINITION=$(echo $TASK_DEFINITION | jq --arg IMAGE_URI "${{ needs.build.outputs.image-uri }}" '.containerDefinitions[0].image = $IMAGE_URI | del(.taskDefinitionArn) | del(.revision) | del(.status) | del(.requiresAttributes) | del(.placementConstraints) | del(.compatibilities) | del(.registeredAt) | del(.registeredBy)')
        echo $NEW_TASK_DEFINITION > task-def.json
        
        # Register new task definition
        aws ecs register-task-definition --cli-input-json file://task-def.json
        
        # Start CodeDeploy deployment
        aws deploy create-deployment \
          --application-name aybiza-staging \
          --deployment-group-name aybiza-staging-dg \
          --deployment-config-name CodeDeployDefault.ECSBlueGreenCanary10Percent5Minutes \
          --description "Automated deployment from GitHub Actions - ${{ github.sha }}"

  deploy-production:
    needs: [build, scan-image, deploy-staging]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/production'
    environment: production
    
    steps:
    - uses: actions/checkout@v4
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
        aws-region: ${{ env.AWS_DEFAULT_REGION }}
        
    - name: Deploy to production with CodeDeploy (Multi-Region)
      run: |
        # Deploy to all production regions
        # Multi-region deployment following AWS native pattern
        for REGION in us-east-1 eu-west-2 us-west-2; do
          echo "Deploying to region: $REGION following Git → CodeBuild → ECR → CodeDeploy → ECS pattern"
          
          # Ensure ECR image is replicated to target region
          aws ecr put-replication-configuration \
            --replication-configuration file://ecr-replication-config.json \
            --region us-east-1
          
          # Wait for image replication
          aws ecr wait image-exists \
            --repository-name aybiza \
            --image-id imageTag=${{ github.sha }} \
            --region $REGION
          
          # Update task definition for this region with regional ECR URI
          REGIONAL_ECR_URI="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/aybiza:${{ github.sha }}"
          TASK_DEFINITION=$(aws ecs describe-task-definition --task-definition aybiza-production --region $REGION --query taskDefinition)
          NEW_TASK_DEFINITION=$(echo $TASK_DEFINITION | jq --arg IMAGE_URI "$REGIONAL_ECR_URI" '.containerDefinitions[0].image = $IMAGE_URI | del(.taskDefinitionArn) | del(.revision) | del(.status) | del(.requiresAttributes) | del(.placementConstraints) | del(.compatibilities) | del(.registeredAt) | del(.registeredBy)')
          echo $NEW_TASK_DEFINITION > task-def-$REGION.json
          
          # Register new task definition
          aws ecs register-task-definition --cli-input-json file://task-def-$REGION.json --region $REGION
          
          # Start CodeDeploy blue/green deployment following the AWS pattern
          DEPLOYMENT_ID=$(aws deploy create-deployment \
            --application-name aybiza-production \
            --deployment-group-name aybiza-production-dg \
            --deployment-config-name CodeDeployDefault.ECSBlueGreenCanary10Percent5Minutes \
            --description "AWS native multi-region deployment - ${{ github.sha }}" \
            --region $REGION \
            --query 'deploymentId' \
            --output text)
          
          echo "Started deployment $DEPLOYMENT_ID in region $REGION"
          
          # Monitor deployment progress
          aws deploy wait deployment-successful \
            --deployment-id $DEPLOYMENT_ID \
            --region $REGION
          
          echo "✅ Deployment successful in region $REGION"
        done

  deploy-cloudflare:
    needs: [deploy-production]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/production'
    steps:
    - uses: actions/checkout@v4
    - name: Deploy Cloudflare Workers
      uses: cloudflare/wrangler-action@v3
      with:
        apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
        accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
        
  post-deploy-validation:
    needs: [deploy-production, deploy-cloudflare]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/production'
    steps:
    - name: Validate deployment across regions
      run: |
        # Health check endpoints
        ENDPOINTS=(
          "https://api.aybiza.com/health"
          "https://us-east-1.api.aybiza.com/health"
          "https://eu-west-2.api.aybiza.com/health"
          "https://us-west-2.api.aybiza.com/health"
        )
        
        for endpoint in "${ENDPOINTS[@]}"; do
          echo "Checking $endpoint"
          if curl -f --max-time 30 "$endpoint"; then
            echo "✅ $endpoint is healthy"
          else
            echo "❌ $endpoint is unhealthy"
            exit 1
          fi
        done
        
    - name: Run smoke tests
      run: |
        # Validate AWS native pipeline deployment
        echo "🧪 Running post-deployment validation following AWS best practices"
        
        # Basic API functionality tests
        curl -X POST https://api.aybiza.com/api/voice/test \
          -H "Content-Type: application/json" \
          -d '{"test": "voice_pipeline"}' \
          --fail --max-time 60
        
        # Validate ECS service health
        for REGION in us-east-1 eu-west-2 us-west-2; do
          echo "Checking ECS service health in $REGION"
          RUNNING_COUNT=$(aws ecs describe-services \
            --cluster aybiza-production \
            --services aybiza-service \
            --region $REGION \
            --query 'services[0].runningCount' \
            --output text)
          
          DESIRED_COUNT=$(aws ecs describe-services \
            --cluster aybiza-production \
            --services aybiza-service \
            --region $REGION \
            --query 'services[0].desiredCount' \
            --output text)
          
          if [ "$RUNNING_COUNT" -eq "$DESIRED_COUNT" ]; then
            echo "✅ ECS service healthy in $REGION ($RUNNING_COUNT/$DESIRED_COUNT)"
          else
            echo "❌ ECS service unhealthy in $REGION ($RUNNING_COUNT/$DESIRED_COUNT)"
            exit 1
          fi
        done
        
        # Validate container image deployment
        DEPLOYED_IMAGE=$(aws ecs describe-task-definition \
          --task-definition aybiza-production \
          --query 'taskDefinition.containerDefinitions[0].image' \
          --output text)
        
        if [[ "$DEPLOYED_IMAGE" == *"${{ github.sha }}"* ]]; then
          echo "✅ Correct container image deployed: $DEPLOYED_IMAGE"
        else
          echo "❌ Incorrect container image deployed: $DEPLOYED_IMAGE"
          exit 1
        fi
```

## Container Optimization and Performance

### Multi-Stage Build Optimization
```dockerfile
# Optimized production Dockerfile
FROM hexpm/elixir:1.18.3-erlang-27.3.4-debian-bookworm-20250517 AS build

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Cache dependencies
WORKDIR /app
COPY mix.exs mix.lock ./
COPY config config
RUN mix local.hex --force && \
    mix local.rebar --force && \
    mix deps.get --only prod

# Build application
COPY apps apps
RUN mix deps.compile && \
    mix compile && \
    mix release

# Production runtime image
FROM debian:bookworm-slim AS runtime

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    openssl \
    libncurses5 \
    curl \
    ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Security hardening
RUN useradd --create-home app
WORKDIR /app
COPY --from=build --chown=app:app /app/_build/prod/rel/aybiza ./
USER app

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:4000/api/health || exit 1

EXPOSE 4000
CMD ["/app/bin/aybiza", "start"]
```

### Container Resource Optimization
```yaml
ECS Task Definition Optimization:
  CPU Architecture: ARM64 (Graviton3) for cost efficiency
  Memory Allocation: Dynamic based on workload
  Network Mode: awsvpc for security
  Logging: CloudWatch with structured logs
  
Resource Configurations:
  Development:
    CPU: 256 (0.25 vCPU)
    Memory: 512 MB
    
  Staging:
    CPU: 512 (0.5 vCPU)
    Memory: 1024 MB
    
  Production:
    CPU: 1024-4096 (1-4 vCPU)
    Memory: 2048-8192 MB
    Auto Scaling: Target 70% CPU utilization
```

## Cost Optimization Strategy

### Container Cost Management
```yaml
Cost Optimization Techniques:
  1. ECR Lifecycle Policies:
     - Keep last 10 production images
     - Delete untagged images after 1 day
     - Archive old images to S3 Glacier
     
  2. ECS Fargate Spot:
     - Use Spot for non-critical workloads
     - 70% cost savings on development/testing
     - Automatic failover to On-Demand
     
  3. Multi-Architecture Builds:
     - ARM64 (Graviton) for 20% cost savings
     - x86_64 for compatibility when needed
     - Automatic architecture selection
     
  4. Resource Right-Sizing:
     - Continuous monitoring and optimization
     - Automated recommendations via AWS Compute Optimizer
     - Regular capacity planning reviews

Monthly Cost Comparison:
  Previous (GitHub Container Registry + Basic ECS): $2,800
  Enhanced (ECR + CodeBuild/Deploy + Optimizations): $3,200
  Additional Investment: $400 (14% increase)
  
Value Added:
  - Enhanced security and compliance
  - Zero-downtime deployments
  - Multi-region resilience
  - Automated rollbacks
  - Enterprise-grade monitoring
```

## Monitoring and Observability Enhancement

### Container-Specific Monitoring
```yaml
CloudWatch Container Insights:
  Metrics:
    - Container CPU/Memory utilization
    - Network I/O and packet drops
    - Storage I/O and utilization
    - Task startup and stop events
    
  Custom Metrics:
    - Voice call processing latency
    - Conversation engine response time
    - Database connection pool health
    - External API integration health
    
AWS X-Ray Integration:
  Tracing:
    - End-to-end request tracing
    - Service map visualization
    - Performance bottleneck identification
    - Error rate and latency analysis
    
  Service Map:
    ELB → ECS → Voice Pipeline → Bedrock → Response
     ↓      ↓         ↓           ↓         ↓
   X-Ray  X-Ray   X-Ray     X-Ray     X-Ray

CodePipeline Monitoring:
  Pipeline Metrics:
    - Pipeline execution success/failure rates
    - Stage execution duration
    - Deployment frequency and lead time
    - Mean time to recovery (MTTR)
    - Change failure rate
    
  CloudWatch Dashboards:
    - Real-time pipeline status
    - Deployment success trends
    - Build and test metrics
    - Multi-region deployment status
    
  EventBridge Integration:
    - Pipeline state changes
    - Deployment notifications
    - Failure alerting
    - Approval workflow triggers

Alerting Strategy:
  Critical Alerts:
    - Pipeline execution failures
    - Container deployment failures
    - High error rates (>5%)
    - Response time degradation (>2s)
    - Security vulnerabilities (Critical/High)
    - Multi-region deployment inconsistencies
    
  Warning Alerts:
    - Resource utilization (>80%)
    - Deployment duration (>10 minutes)
    - Cache hit ratio (<90%)
    - Queue depth increases
    - Pipeline approval pending >4 hours
    
  CodePipeline Specific Alerts:
    - Stage execution timeouts
    - Artifact corruption or unavailability
    - CodeBuild project failures
    - CodeDeploy rollback triggers
    - Cross-region replication delays
```

## Disaster Recovery and Business Continuity

### Enhanced Failover Strategy
```yaml
Container-Level Failover:
  Image Availability:
    - Multi-region ECR replication
    - Local image caching on ECS hosts
    - Fallback to previous image versions
    
  Service Resilience:
    - Blue/Green deployments prevent outages
    - Automatic rollback on health check failures
    - Circuit breakers for external dependencies
    
  Multi-Region Architecture:
    Primary Region Failure:
      1. Cloudflare DNS failover (30 seconds)
      2. ECS service scaling in backup region (2 minutes)
      3. Database failover to read replicas (1 minute)
      4. Session state recovery from Redis (immediate)
      
    RTO: 3 minutes
    RPO: 30 seconds

Backup and Recovery:
  Container Images:
    - ECR lifecycle policies retain critical versions
    - Cross-region replication ensures availability
    - S3 backup for long-term archival
    
  Configuration:
    - Task definitions versioned and backed up
    - Infrastructure as Code in Git
    - Automated configuration drift detection
```

## Security and Compliance Enhancement

### Container Security Framework
```yaml
Security Layers:
  1. Build-Time Security:
     - Base image vulnerability scanning
     - Dependency vulnerability analysis
     - Static code analysis integration
     - Secret detection and prevention
     
  2. Registry Security:
     - ECR vulnerability scanning
     - Image signing with AWS Signer
     - Access control with IAM policies
     - Audit logging for all operations
     
  3. Runtime Security:
     - ECS security groups and NACLs
     - VPC flow logs monitoring
     - Runtime threat detection
     - Behavioral analysis and alerting
     
  4. Compliance:
     - SOC 2 Type II compliance
     - HIPAA security controls
     - PCI DSS requirements
     - GDPR data protection

Container Hardening:
  - Minimal base images (distroless when possible)
  - Non-root user execution
  - Read-only root filesystem
  - Dropped Linux capabilities
  - Security context constraints
  - Network policy enforcement
```

## Rollback and Recovery Procedures

### Enhanced Rollback Strategy
```yaml
Rollback Triggers:
  Automatic:
    - Health check failures
    - Error rate spikes (>5%)
    - Performance degradation (>50% latency increase)
    - Security vulnerability detection
    
  Manual:
    - Business logic issues
    - Feature flag driven rollbacks
    - Planned maintenance rollbacks

Rollback Procedures:
  1. CodeDeploy Automatic Rollback:
     - Triggered by CloudWatch alarms
     - Blue/Green traffic shifting reversal
     - Previous task definition restoration
     - Duration: 2-5 minutes
     
  2. Manual ECR Image Rollback:
     ```bash
     # Get previous image
     PREVIOUS_IMAGE=$(aws ecr describe-images \
       --repository-name aybiza \
       --query 'sort_by(imageDetails,&imagePushedAt)[-2].imageTags[0]')
     
     # Update ECS service
     aws ecs update-service \
       --cluster aybiza-production \
       --service aybiza-service \
       --task-definition aybiza:$PREVIOUS_IMAGE
     ```
     
  3. Cross-Region Failover:
     - DNS failover via Cloudflare
     - Traffic routing to healthy regions
     - Data synchronization verification
     - Duration: 30 seconds - 2 minutes
```

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
- [ ] Set up Amazon ECR repositories across regions
- [ ] Configure ECR vulnerability scanning
- [ ] Migrate production images from GitHub Container Registry
- [ ] Update CI/CD pipeline for ECR integration

### Phase 2: AWS Native Pipeline Setup (Weeks 3-4)
- [ ] Implement AWS CodePipeline following Git → CodeBuild → ECR → CodeDeploy → ECS pattern
- [ ] Configure AWS CodeBuild for container builds with enhanced buildspec.yml
- [ ] Set up AWS CodeDeploy for blue/green deployments to ECS/Fargate
- [ ] Implement automated security scanning gates within pipeline
- [ ] Configure artifact management and cross-stage data flow

### Phase 3: Multi-Region & Monitoring (Weeks 5-6)
- [ ] Extend AWS CodePipeline for multi-region deployments
- [ ] Deploy CloudWatch Container Insights
- [ ] Configure AWS X-Ray distributed tracing
- [ ] Implement cost optimization strategies
- [ ] Set up comprehensive pipeline and container alerting

### Phase 4: Advanced Features & Hybrid Approach (Weeks 7-8)
- [ ] Implement dual pipeline approach (AWS CodePipeline + GitHub Actions)
- [ ] Deploy advanced security monitoring with AWS Security Hub integration
- [ ] Configure automated compliance reporting
- [ ] Implement chaos engineering testing
- [ ] Set up pipeline analytics and optimization recommendations

### Pipeline Migration Strategy

**Week 1-2: Parallel Setup**
- Run both GitHub Actions and AWS CodePipeline in parallel
- Compare deployment times, success rates, and monitoring capabilities
- Validate artifact consistency between approaches

**Week 3-4: Gradual Migration**
- Migrate staging environment to AWS CodePipeline first
- Keep production on GitHub Actions during validation
- Monitor and tune AWS native pipeline performance

**Week 5-6: Production Migration**
- Switch production deployments to AWS CodePipeline
- Keep GitHub Actions as backup/development pipeline
- Implement rollback procedures for both approaches

**Week 7-8: Optimization & Documentation**
- Optimize pipeline performance and costs
- Document runbooks for both pipeline approaches
- Train team on AWS native pipeline operations

## Pipeline Architecture Comparison

### AWS CodePipeline (Native) vs GitHub Actions

| Aspect | AWS CodePipeline | GitHub Actions |
|--------|------------------|----------------|
| **Integration** | Native AWS services | Third-party with AWS CLI |
| **Artifact Management** | Built-in S3 integration | Manual artifact handling |
| **Security** | IAM roles throughout | GitHub secrets + AWS credentials |
| **Monitoring** | CloudWatch native | External monitoring setup |
| **Cost** | Pay per pipeline execution | Free tier + paid minutes |
| **Compliance** | AWS native audit trails | Requires additional logging |
| **Debugging** | CloudWatch Logs integration | GitHub Actions logs |
| **Multi-region** | Native multi-region support | Requires custom scripting |

### When to Use Each Approach

**Use AWS CodePipeline when:**
- Enterprise compliance requirements (SOC 2, HIPAA)
- Heavy AWS service integration
- Need for AWS native audit trails
- Multi-region deployments are critical
- Cost predictability is important

**Use GitHub Actions when:**
- Development-focused workflows
- Need for community marketplace actions
- Tight integration with GitHub features
- Flexibility in workflow customization
- Hybrid cloud or multi-cloud deployments

## Conclusion

This enhanced AWS container deployment strategy provides two complementary approaches following the industry-standard Git → CodeBuild → ECR → CodeDeploy → ECS/Fargate pattern described in the AWS "Back to Basics" methodology. The implementation delivers:

**Core Benefits:**
- **Zero-downtime deployments** with blue/green strategies
- **Enhanced security** with comprehensive vulnerability scanning
- **Multi-region resilience** with automated failover
- **Cost optimization** through right-sizing and spot instances
- **Enterprise monitoring** with detailed observability
- **Compliance readiness** for SOC 2, HIPAA, and GDPR

**AWS Native Pipeline Advantages:**
- Full AWS service integration following the transcript's recommended architecture
- Native artifact management and security
- Built-in compliance and audit capabilities
- Simplified multi-region deployments
- Consistent AWS experience and support

The dual approach ensures flexibility while providing a clear migration path to AWS native services for enterprise-grade deployments. The phased implementation minimizes risk while maximizing the benefits of both GitHub's developer experience and AWS's enterprise container deployment capabilities.