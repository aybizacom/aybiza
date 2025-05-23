## 6. Docker Configuration Documentation

```markdown
# AYBIZA AI Voice Agent Platform - Docker Configuration

## Development Environment

### docker-compose.yml
```yaml
version: '3.8'

services:
  postgres:
    image: timescale/timescaledb:2.13.0-pg15
    container_name: aybiza_postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: aybiza_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init:/docker-entrypoint-initdb.d
    networks:
      - aybiza_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7.2.4-alpine
    container_name: aybiza_redis
    command: redis-server --requirepass redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - aybiza_network
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "redis", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      

  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: aybiza_app
    environment:
      MIX_ENV: dev
      ELIXIR_VERSION: 1.16.0
      OTP_VERSION: 26.1.2
      DATABASE_URL: postgres://postgres:postgres@postgres:5432/aybiza_dev
      REDIS_URL: redis://:redis@redis:6379
      TWILIO_ACCOUNT_SID: ${TWILIO_ACCOUNT_SID}
      TWILIO_AUTH_TOKEN: ${TWILIO_AUTH_TOKEN}
      DEEPGRAM_API_KEY: ${DEEPGRAM_API_KEY}
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
      AWS_REGION: ${AWS_REGION}
      AWS_BEDROCK_MODELS: ${AWS_BEDROCK_MODELS}
      AWS_DEFAULT_BEDROCK_MODEL: ${AWS_DEFAULT_BEDROCK_MODEL}
      SECRET_KEY_BASE: ${SECRET_KEY_BASE}
      PHX_HOST: localhost
      PHX_PORT: 4000
      # Debugging options
      ELIXIR_ERL_OPTIONS: "-kernel shell_history enabled"
      RELEASE_NODE: aybiza@aybiza_app
    ports:
      - "4000:4000"
    volumes:
      - .:/app
      - deps:/app/deps
      - build:/app/_build
    networks:
      - aybiza_network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: >
      bash -c "
        mix deps.get &&
        mix ecto.setup &&
        mix phx.server
      "

networks:
  aybiza_network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  deps:
  build:
```

### Dockerfile.dev
```dockerfile
FROM hexpm/elixir:1.18.3-erlang-28.0-debian-bookworm-20250117

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    inotify-tools \
    git \
    curl \
    wget \
    gnupg2 \
    postgresql-client \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set environment variables
ENV MIX_HOME=/opt/mix
ENV HEX_HOME=/opt/hex

# Install Hex and Rebar
RUN mix local.hex --force && \
    mix local.rebar --force

# Create app directory
WORKDIR /app

# Copy dependency files
COPY mix.exs mix.lock ./
COPY config config

# Get dependencies
RUN mix deps.get

# Copy umbrella apps for dependency resolution
COPY apps apps

# Compile dependencies
RUN mix deps.compile

# Set up development environment
ENV MIX_ENV=dev

# Expose Phoenix port
EXPOSE 4000

# Set the default command
CMD ["mix", "phx.server"]
```

## Production Environment

### docker-compose.prod.yml
```yaml
version: '3.8'

services:
  app:
    image: ${ECR_REPOSITORY_URI}:${IMAGE_TAG}
    container_name: aybiza_app
    restart: always
    environment:
      MIX_ENV: prod
      RELEASE_NODE: aybiza@aybiza_app
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      TWILIO_ACCOUNT_SID: ${TWILIO_ACCOUNT_SID}
      TWILIO_AUTH_TOKEN: ${TWILIO_AUTH_TOKEN}
      DEEPGRAM_API_KEY: ${DEEPGRAM_API_KEY}
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
      AWS_REGION: ${AWS_REGION}
      AWS_BEDROCK_MODELS: ${AWS_BEDROCK_MODELS}
      AWS_DEFAULT_BEDROCK_MODEL: ${AWS_DEFAULT_BEDROCK_MODEL}
      SECRET_KEY_BASE: ${SECRET_KEY_BASE}
      PHX_HOST: ${PHX_HOST}
      PHX_PORT: ${PHX_PORT}
    ports:
      - "4000:4000"
    networks:
      - aybiza_network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G

networks:
  aybiza_network:
    driver: bridge
```

### Dockerfile.prod
```dockerfile
FROM hexpm/elixir:1.18.3-erlang-28.0-debian-bookworm-20250117 AS build

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set environment variables
ENV MIX_ENV=prod
ENV MIX_HOME=/opt/mix
ENV HEX_HOME=/opt/hex

# Install Hex and Rebar
RUN mix local.hex --force && \
    mix local.rebar --force

# Create app directory
WORKDIR /app

# Copy dependency files
COPY mix.exs mix.lock ./
COPY config config
COPY apps apps

# Get dependencies
RUN mix deps.get --only prod

# Compile the project
RUN mix do compile

# Build release
RUN mix release

# Prepare release image
FROM debian:bookworm-slim AS app

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    openssl \
    libncurses5 \
    locales \
    curl \
    ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set locale
RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

WORKDIR /app

# Copy release from build stage
COPY --from=build /app/_build/prod/rel/aybiza ./

# Set run user
RUN useradd --create-home app
RUN chown -R app: /app
USER app

# Set environment variables
ENV HOME=/app
ENV RELEASE_NODE=aybiza@127.0.0.1

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:4000/api/health || exit 1

# Expose Phoenix port
EXPOSE 4000

# Run application
CMD ["/app/bin/aybiza", "start"]
```

## Multi-Stage Build for Voice Agent Components

For the voice agent components that require additional dependencies, we use a specialized Dockerfile:

```dockerfile
FROM hexpm/elixir:1.18.3-erlang-28.0-debian-bookworm-20250117 AS build

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    curl \
    wget \
    gnupg2 \
    libtool \
    autoconf \
    automake \
    libssl-dev \
    libopus-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set environment variables
ENV MIX_ENV=prod
ENV MIX_HOME=/opt/mix
ENV HEX_HOME=/opt/hex

# Install Hex and Rebar
RUN mix local.hex --force && \
    mix local.rebar --force

# Create app directory
WORKDIR /app

# Copy dependency files
COPY mix.exs mix.lock ./
COPY config config
COPY apps apps

# Get dependencies
RUN mix deps.get --only prod

# Compile the project
RUN mix do compile

# Build release
RUN mix release

# Prepare release image
FROM debian:bookworm-slim AS app

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    openssl \
    libncurses5 \
    locales \
    curl \
    ca-certificates \
    libopus0 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set locale
RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

WORKDIR /app

# Copy release from build stage
COPY --from=build /app/_build/prod/rel/aybiza ./

# Set run user
RUN useradd --create-home app
RUN chown -R app: /app
USER app

# Set environment variables
ENV HOME=/app
ENV RELEASE_NODE=aybiza@127.0.0.1

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:4000/api/health || exit 1

# Expose Phoenix port
EXPOSE 4000

# Run application
CMD ["/app/bin/aybiza", "start"]
```

## Container Optimization

### Memory Optimization

```dockerfile
# Example of memory optimization settings
ENV ERL_MAX_PORTS=65536
ENV ERL_FULLSWEEP_AFTER=10
ENV ERL_CRASH_DUMP=/var/log/aybiza/erl_crash.dump

# Set BEAM VM flags
ENV BEAM_FLAGS="+SDio 64 +sbwt none +sbwtdcpu none +sbwtdio none +swt very_low +sws medium +swt medium"
```

### Security Hardening

```dockerfile
# Security hardening
RUN chmod -R 550 /app/bin && \
    chmod -R 550 /app/releases && \
    chmod -R 550 /app/lib && \
    chmod -R 770 /app/var

# Add security labels (when used with Docker >= 1.13)
LABEL org.label-schema.schema-version="1.0"
LABEL org.label-schema.name="aybiza"
LABEL org.label-schema.vendor="AYBIZA"
LABEL org.label-schema.version="${BUILD_VERSION}"
LABEL org.label-schema.build-date="${BUILD_DATE}"
LABEL org.label-schema.description="AYBIZA AI Voice Agent Platform"
```

## Build and Run Instructions

### Development Environment
1. Create a `.env` file with required environment variables:
   ```
   TWILIO_ACCOUNT_SID=your_twilio_account_sid
   TWILIO_AUTH_TOKEN=your_twilio_auth_token
   DEEPGRAM_API_KEY=your_deepgram_api_key
   # AWS Bedrock authentication and configuration
   AWS_ACCESS_KEY_ID=your_aws_access_key_id
   AWS_SECRET_ACCESS_KEY=your_aws_secret_access_key
   AWS_REGION=us-east-1
   AWS_BEDROCK_MODELS="anthropic.claude-4-opus-20250120,anthropic.claude-4-sonnet-20250120,anthropic.claude-3-5-sonnet-20241022-v2:0,anthropic.claude-3-haiku-20240307-v1:0"
   AWS_DEFAULT_BEDROCK_MODEL=anthropic.claude-4-opus-20250120
   # NOTE: Claude 4 models (Opus/Sonnet) are NOW available on AWS Bedrock!
   # Some features like MCP tool use may have limitations on Bedrock - see aws_bedrock_claude_implementation.md
   SECRET_KEY_BASE=a_long_secret_key_base
   ```

2. Build and start the development environment:
   ```bash
   docker-compose up -d
   ```

3. Set up the database:
   ```bash
   docker-compose exec app mix ecto.setup
   ```

4. Access the application:
   - Phoenix server: http://localhost:4000
   - External access: via GitHub Codespaces URL

### Production Environment
1. Build the production Docker image:
   ```bash
   docker build -f Dockerfile.prod -t aybiza:latest .
   ```

2. Tag and push the image to your registry:
   ```bash
   docker tag aybiza:latest ${ECR_REPOSITORY_URI}:${IMAGE_TAG}
   docker push ${ECR_REPOSITORY_URI}:${IMAGE_TAG}
   ```

3. Deploy using docker-compose.prod.yml:
   ```bash
   docker-compose -f docker-compose.prod.yml up -d
   ```

## Volume Management
- **postgres_data**: Persistent storage for PostgreSQL
- **redis_data**: Persistent storage for Redis
- **deps**: Caches Elixir dependencies
- **build**: Caches compiled Elixir code

## Network Configuration
- All services are on the same bridge network (`aybiza_network`)
- Only necessary ports are exposed:
  - 4000: Phoenix application
  - 5432: PostgreSQL (dev only)
  - 6379: Redis (dev only)

## Python Runtime Container for Code Execution

### Development Environment with Python Support

For Claude 4's code execution capabilities, we need a Python 3.11.12 runtime container:

```dockerfile
# Dockerfile.python-runtime - Python execution environment for Claude 4
FROM python:3.11.12-slim-bookworm AS python-runtime

# Install essential packages for code execution
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    curl \
    wget \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Create sandboxed environment
RUN useradd --create-home --shell /bin/bash coderunner
WORKDIR /home/coderunner

# Set resource limits
ENV PYTHONUNBUFFERED=1
ENV PYTHONIOENCODING=utf-8
ENV MEMORY_LIMIT=1G
ENV STORAGE_LIMIT=5G

# Install common Python packages
COPY requirements-runtime.txt .
RUN pip install --no-cache-dir -r requirements-runtime.txt

USER coderunner

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD python -c "import sys; sys.exit(0)"

EXPOSE 8089

CMD ["python", "-m", "http.server", "8089"]
```

### Python Runtime Integration with Elixir

```yaml
# docker-compose.yml addition for Python runtime
services:
  python_runtime:
    build:
      context: .
      dockerfile: Dockerfile.python-runtime
    container_name: aybiza_python_runtime
    environment:
      MAX_MEMORY: 1G
      MAX_STORAGE: 5G
      EXECUTION_TIMEOUT: 30
    volumes:
      - python_workspace:/home/coderunner/workspace:rw
    networks:
      - aybiza_network
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETUID
      - SETGID
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M

volumes:
  python_workspace:
    driver: local
    driver_opts:
      type: tmpfs
      device: tmpfs
      o: size=5G,uid=1000,gid=1000
```

### Security Considerations for Code Execution

1. **Sandboxing**: Run Python code in isolated containers with restricted permissions
2. **Resource Limits**: Enforce memory (1 GiB) and storage (5 GiB) limits
3. **Network Isolation**: Restrict network access for code execution containers
4. **Temporary Storage**: Use tmpfs for workspace to ensure cleanup
5. **User Isolation**: Run as non-root user with minimal capabilities

### Integration with AYBIZA Voice Agents

```elixir
# Example Elixir module for Python code execution
defmodule Aybiza.CodeExecution.PythonRunner do
  @moduledoc """
  Manages Python code execution for Claude 4 agents
  """
  
  def execute_code(code, timeout \\ 30_000) do
    container_name = "aybiza_python_runtime"
    
    # Execute code in sandboxed environment
    case System.cmd("docker", [
      "exec", 
      "-i", 
      container_name,
      "timeout", 
      "#{timeout}s",
      "python", 
      "-c", 
      code
    ]) do
      {output, 0} -> {:ok, output}
      {error, _} -> {:error, error}
    end
  end
end
```

## Container Deployment Pipelines

### AWS CodePipeline Integration (Production Approach)

Following the **Git → CodeBuild → ECR → CodeDeploy → ECS/Fargate** pattern, AYBIZA uses AWS CodeBuild for enterprise-grade container building and deployment.

#### Primary CodeBuild Configuration (buildspec.yml)

```yaml
# buildspec.yml - Main container build specification
version: 0.2

env:
  variables:
    IMAGE_REPO_NAME: aybiza
    IMAGE_TAG: latest
    AWS_DEFAULT_REGION: us-east-1
    
  parameter-store:
    AWS_ACCOUNT_ID: /aybiza/aws-account-id
    
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
      - echo Build started on `date`
      
  build:
    commands:
      - echo Building Docker image for AYBIZA Voice Platform...
      - # Build production-optimized container
      - docker build -f Dockerfile.prod -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $REPOSITORY_URI:$IMAGE_TAG
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $REPOSITORY_URI:latest
      
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing Docker images to ECR...
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:latest
      - echo Writing image definitions file for CodeDeploy...
      - printf '[{"name":"aybiza-container","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
      - printf '{"ImageURI":"%s"}' $REPOSITORY_URI:$IMAGE_TAG > imageDetail.json
      
artifacts:
  files:
    - imagedefinitions.json
    - imageDetail.json
    - appspec.yml
    - taskdef.json
```

#### Multi-Component Build Configuration

```yaml
# buildspec-multi-component.yml - For umbrella app components
version: 0.2

env:
  variables:
    AWS_DEFAULT_REGION: us-east-1
    
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      
  build:
    commands:
      - echo Building multiple AYBIZA components...
      
      # Build Voice Gateway
      - echo "Building Voice Gateway component..."
      - docker build -f apps/voice_gateway/Dockerfile.prod -t aybiza/voice-gateway:$COMMIT_HASH .
      - docker tag aybiza/voice-gateway:$COMMIT_HASH $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/voice-gateway:$COMMIT_HASH
      
      # Build Voice Pipeline
      - echo "Building Voice Pipeline component..."
      - docker build -f apps/voice_pipeline/Dockerfile.prod -t aybiza/voice-pipeline:$COMMIT_HASH .
      - docker tag aybiza/voice-pipeline:$COMMIT_HASH $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/voice-pipeline:$COMMIT_HASH
      
      # Build Conversation Engine
      - echo "Building Conversation Engine component..."
      - docker build -f apps/conversation_engine/Dockerfile.prod -t aybiza/conversation-engine:$COMMIT_HASH .
      - docker tag aybiza/conversation-engine:$COMMIT_HASH $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/conversation-engine:$COMMIT_HASH
      
      # Build Agent Manager
      - echo "Building Agent Manager component..."
      - docker build -f apps/agent_manager/Dockerfile.prod -t aybiza/agent-manager:$COMMIT_HASH .
      - docker tag aybiza/agent-manager:$COMMIT_HASH $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/agent-manager:$COMMIT_HASH
      
  post_build:
    commands:
      - echo Pushing all images to ECR...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/voice-gateway:$COMMIT_HASH
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/voice-pipeline:$COMMIT_HASH
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/conversation-engine:$COMMIT_HASH
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/agent-manager:$COMMIT_HASH
      
      # Generate deployment artifacts for each component
      - echo "Generating deployment artifacts..."
      - printf '[
          {"name":"voice-gateway","imageUri":"%s"},
          {"name":"voice-pipeline","imageUri":"%s"},
          {"name":"conversation-engine","imageUri":"%s"},
          {"name":"agent-manager","imageUri":"%s"}
        ]' \
        "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/voice-gateway:$COMMIT_HASH" \
        "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/voice-pipeline:$COMMIT_HASH" \
        "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/conversation-engine:$COMMIT_HASH" \
        "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/aybiza/agent-manager:$COMMIT_HASH" \
        > imagedefinitions.json
        
artifacts:
  files:
    - imagedefinitions.json
    - appspec.yml
    - taskdef-*.json
```

#### Environment-Specific Build Configurations

```yaml
# buildspec-staging.yml - Staging environment builds
version: 0.2

env:
  variables:
    IMAGE_REPO_NAME: aybiza
    ENVIRONMENT: staging
    AWS_DEFAULT_REGION: us-east-1
    
phases:
  pre_build:
    commands:
      - echo Building for STAGING environment...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - IMAGE_TAG=staging-$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      
  build:
    commands:
      - echo Building Docker image with staging optimizations...
      - # Build with staging-specific settings
      - docker build -f Dockerfile.prod \
          --build-arg MIX_ENV=staging \
          --build-arg BUILD_ENV=staging \
          -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $REPOSITORY_URI:$IMAGE_TAG
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $REPOSITORY_URI:staging-latest
      
  post_build:
    commands:
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:staging-latest
      - printf '[{"name":"aybiza-container","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yml
```

### CodeBuild Integration with Docker Optimization

#### Enhanced Production Dockerfile for CodeBuild

```dockerfile
# Dockerfile.codebuild - Optimized for AWS CodeBuild
FROM hexpm/elixir:1.18.3-erlang-28.0-debian-bookworm-20250117 AS build

# Build arguments for environment-specific builds
ARG MIX_ENV=prod
ARG BUILD_ENV=production
ARG BUILD_VERSION
ARG BUILD_DATE

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    curl \
    wget \
    gnupg2 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set environment variables
ENV MIX_ENV=${MIX_ENV}
ENV BUILD_ENV=${BUILD_ENV}
ENV MIX_HOME=/opt/mix
ENV HEX_HOME=/opt/hex

# Install Hex and Rebar
RUN mix local.hex --force && \
    mix local.rebar --force

# Create app directory
WORKDIR /app

# Copy dependency files for better layer caching
COPY mix.exs mix.lock ./
COPY config config

# Get dependencies (cached layer)
RUN mix deps.get --only prod

# Copy application code
COPY apps apps
COPY priv priv

# Compile dependencies and application
RUN mix deps.compile && \
    mix compile

# Build release
RUN mix release

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
RUN useradd --create-home --shell /bin/bash app

WORKDIR /app

# Copy release from build stage
COPY --from=build --chown=app:app /app/_build/prod/rel/aybiza ./

USER app

# Health check for CodeDeploy
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:4000/api/health || exit 1

# Container metadata for AWS
LABEL org.opencontainers.image.source="https://github.com/aybiza/aybiza-platform"
LABEL org.opencontainers.image.version="${BUILD_VERSION}"
LABEL org.opencontainers.image.created="${BUILD_DATE}"
LABEL org.opencontainers.image.vendor="AYBIZA"
LABEL org.opencontainers.image.title="AYBIZA AI Voice Agent Platform"

EXPOSE 4000

CMD ["/app/bin/aybiza", "start"]
```

### ECR Integration Commands

```bash
# ECR Repository Management for CodeBuild
#!/bin/bash
# scripts/setup-ecr-repositories.sh

set -e

AWS_REGION=${AWS_REGION:-us-east-1}
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create ECR repositories for each component
REPOSITORIES=(
    "aybiza/voice-gateway"
    "aybiza/voice-pipeline"
    "aybiza/conversation-engine"
    "aybiza/agent-manager"
    "aybiza/phone-manager"
    "aybiza/call-analytics"
)

echo "🏗️ Setting up ECR repositories in $AWS_REGION"

for repo in "${REPOSITORIES[@]}"; do
    echo "Creating repository: $repo"
    
    aws ecr create-repository \
        --repository-name "$repo" \
        --region "$AWS_REGION" \
        --image-scanning-configuration scanOnPush=true \
        --encryption-configuration encryptionType=AES256 \
        || echo "Repository $repo already exists"
    
    # Set lifecycle policy
    aws ecr put-lifecycle-policy \
        --repository-name "$repo" \
        --region "$AWS_REGION" \
        --lifecycle-policy-text '{
            "rules": [
                {
                    "rulePriority": 1,
                    "selection": {
                        "tagStatus": "untagged",
                        "countType": "sinceImagePushed",
                        "countUnit": "days",
                        "countNumber": 1
                    },
                    "action": {
                        "type": "expire"
                    }
                },
                {
                    "rulePriority": 2,
                    "selection": {
                        "tagStatus": "tagged",
                        "countType": "imageCountMoreThan",
                        "countNumber": 10
                    },
                    "action": {
                        "type": "expire"
                    }
                }
            ]
        }'
done

echo "✅ ECR repositories setup complete"
```

### GitHub Actions Integration (Development Approach)

For development and testing environments, GitHub Actions provides rapid iteration:

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, staging, development ]
  pull_request:
    branches: [ main ]

env:
  AWS_DEFAULT_REGION: us-east-1

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: aybiza_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_HOST_AUTH_METHOD: trust
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
        otp-version: '28.0'
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          deps
          _build
        key: ${{ runner.os }}-mix-${{ hashFiles('**/mix.lock') }}
        restore-keys: ${{ runner.os }}-mix-
    
    - name: Install dependencies
      run: |
        mix local.hex --force
        mix local.rebar --force
        mix deps.get
    
    - name: Check formatting
      run: mix format --check-formatted
    
    - name: Run Credo
      run: mix credo --strict
    
    - name: Run tests
      env:
        DATABASE_URL: postgres://postgres:postgres@localhost:5432/aybiza_test
        REDIS_URL: redis://localhost:6379
        MIX_ENV: test
      run: mix test
    
    - name: Run security checks
      run: mix sobelow --config

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/staging'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        file: ./Dockerfile.prod
        push: true
        tags: |
          ghcr.io/${{ github.repository }}:${{ github.sha }}
          ghcr.io/${{ github.repository }}:latest
        build-args: |
          BUILD_DATE=${{ github.event.head_commit.timestamp }}
          BUILD_VERSION=${{ github.sha }}

    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/staging'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
    
    - name: Deploy to ECS
      run: |
        export ENVIRONMENT=$([ "${{ github.ref }}" = "refs/heads/main" ] && echo "development" || echo "staging")
        export CLUSTER_NAME=aybiza-${ENVIRONMENT}
        
        # Update ECS service with new image
        aws ecs update-service \
          --cluster ${CLUSTER_NAME} \
          --service aybiza-service \
          --force-new-deployment
        
        # Wait for deployment to complete
        aws ecs wait services-stable \
          --cluster ${CLUSTER_NAME} \
          --services aybiza-service
        
        # Verify deployment
        aws ecs describe-services \
          --cluster ${CLUSTER_NAME} \
          --services aybiza-service
        # Verify deployment
        aws ecs describe-services \
          --cluster ${CLUSTER_NAME} \
          --services aybiza-service
```

This GitHub Actions workflow provides:

1. **Automated Testing**: Runs tests, formatting checks, and security scans
2. **Docker Build**: Builds and pushes images to GitHub Container Registry
3. **AWS Deployment**: Deploys to ECS and verifies deployment success
4. **Environment Management**: Handles development and staging environments
5. **Caching**: Optimizes build times with dependency caching

## Environment Variables

### Production Environment
| Variable | Description | Required |
|----------|-------------|----------|
| TWILIO_ACCOUNT_SID | Twilio account identifier | Yes |
| TWILIO_AUTH_TOKEN | Twilio authentication token | Yes |
| DEEPGRAM_API_KEY | Deepgram API key | Yes |
| AWS_ACCESS_KEY_ID | AWS access key ID | Yes |
| AWS_SECRET_ACCESS_KEY | AWS secret access key | Yes |
| AWS_REGION | AWS region | Yes |
| SECRET_KEY_BASE | Phoenix secret key base | Yes |
| DATABASE_URL | PostgreSQL connection URL | Yes |
| REDIS_URL | Redis connection URL | Yes |

### Development Environment
| Variable | Description | Required |
|----------|-------------|----------|
| TWILIO_ACCOUNT_SID | Twilio account identifier | Yes |
| TWILIO_AUTH_TOKEN | Twilio authentication token | Yes |
| DEEPGRAM_API_KEY | Deepgram API key | Yes |
| AWS_ACCESS_KEY_ID | AWS access key ID | Yes |
| AWS_SECRET_ACCESS_KEY | AWS secret access key | Yes |
| AWS_REGION | AWS region | Yes |
| SECRET_KEY_BASE | Phoenix secret key base | Yes |

## Security Considerations

1. **Secrets Management**: All sensitive environment variables are stored as GitHub secrets
2. **Container Security**: Regular security scans with Sobelow
3. **Image Scanning**: Container images are scanned for vulnerabilities
4. **Least Privilege**: AWS IAM roles follow principle of least privilege
5. **Network Security**: Proper VPC and security group configuration

## Troubleshooting

### Common Issues

1. **Build Failures**: Check test results and formatting
2. **Deployment Failures**: Verify AWS credentials and ECS configuration
3. **Container Issues**: Check Docker logs and resource allocation
4. **Environment Variables**: Ensure all required variables are set

### Debugging Commands

```bash
# Check container logs
docker logs aybiza_app

# Check service status
docker-compose ps

# Restart services
docker-compose restart

# Rebuild from scratch
docker-compose down && docker-compose up --build
```

This configuration provides a robust, scalable, and secure deployment pipeline for the AYBIZA platform using GitHub Actions and Docker containerization.