# AYBIZA AWS Infrastructure with Terraform

## Overview
This document provides complete Terraform configurations for deploying the AYBIZA AI voice agent platform on AWS with multi-region support, auto-scaling, and production-grade security.

## Architecture Overview

### Multi-Region Deployment
```
Primary: us-east-1 (N. Virginia)
Secondary: us-west-2 (Oregon)  
EU: eu-west-1 (Ireland)
```

### Infrastructure Components
- **ECS Fargate**: Container orchestration
- **Aurora PostgreSQL**: Global database with read replicas
- **ElastiCache Redis**: Distributed caching
- **Application Load Balancer**: Traffic distribution
- **VPC**: Network isolation and security
- **CloudWatch**: Monitoring and logging
- **Secrets Manager**: Secure credential storage

## Project Structure

```
terraform/
├── modules/
│   ├── vpc/
│   ├── ecs/
│   ├── rds/
│   ├── elasticache/
│   ├── alb/
│   ├── security/
│   └── monitoring/
├── environments/
│   ├── development/
│   ├── staging/
│   └── production/
├── global/
│   ├── route53/
│   ├── cloudfront/
│   └── iam/
└── scripts/
    ├── deploy.sh
    ├── destroy.sh
    └── validate.sh
```

## Core Modules

### 1. VPC Module
```hcl
# modules/vpc/main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "region" {
  description = "AWS region"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
}

locals {
  name_prefix = "aybiza-${var.environment}"
  
  tags = {
    Environment = var.environment
    Project     = "aybiza"
    ManagedBy   = "terraform"
    Region      = var.region
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-igw"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-public-${count.index + 1}"
    Type = "public"
  })
}

# Private Subnets for Application
resource "aws_subnet" "private_app" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-private-app-${count.index + 1}"
    Type = "private-app"
  })
}

# Private Subnets for Database
resource "aws_subnet" "private_db" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index + 20)
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-private-db-${count.index + 1}"
    Type = "private-db"
  })
}

# NAT Gateways
resource "aws_eip" "nat" {
  count = length(var.availability_zones)

  domain = "vpc"

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-nat-eip-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  count = length(var.availability_zones)

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-nat-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-public-rt"
  })
}

resource "aws_route_table" "private_app" {
  count = length(var.availability_zones)

  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-private-app-rt-${count.index + 1}"
  })
}

resource "aws_route_table" "private_db" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-private-db-rt"
  })
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private_app" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.private_app[count.index].id
  route_table_id = aws_route_table.private_app[count.index].id
}

resource "aws_route_table_association" "private_db" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.private_db[count.index].id
  route_table_id = aws_route_table.private_db.id
}

# VPC Endpoints for AWS Services
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.region}.s3"

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-s3-endpoint"
  })
}

resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-ecr-api-endpoint"
  })
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-ecr-dkr-endpoint"
  })
}

resource "aws_vpc_endpoint" "logs" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.logs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-logs-endpoint"
  })
}

# Security Group for VPC Endpoints
resource "aws_security_group" "vpc_endpoints" {
  name_prefix = "${local.name_prefix}-vpc-endpoints"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.cidr_block]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-vpc-endpoints-sg"
  })
}

# Outputs
output "vpc_id" {
  value = aws_vpc.main.id
}

output "vpc_cidr_block" {
  value = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_app_subnet_ids" {
  value = aws_subnet.private_app[*].id
}

output "private_db_subnet_ids" {
  value = aws_subnet.private_db[*].id
}

output "internet_gateway_id" {
  value = aws_internet_gateway.main.id
}

output "nat_gateway_ids" {
  value = aws_nat_gateway.main[*].id
}
```

### 2. ECS Module
```hcl
# modules/ecs/main.tf
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "region" {
  description = "AWS region"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnet_ids" {
  description = "Private subnet IDs"
  type        = list(string)
}

variable "public_subnet_ids" {
  description = "Public subnet IDs"
  type        = list(string)
}

variable "target_group_arn" {
  description = "ALB target group ARN"
  type        = string
}

variable "security_group_ids" {
  description = "Security group IDs"
  type        = list(string)
}

locals {
  name_prefix = "aybiza-${var.environment}"
  
  tags = {
    Environment = var.environment
    Project     = "aybiza"
    ManagedBy   = "terraform"
    Region      = var.region
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = local.name_prefix

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = local.tags
}

# ECS Cluster Capacity Providers
resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    base              = 1
    weight            = 100
    capacity_provider = "FARGATE"
  }

  default_capacity_provider_strategy {
    base              = 0
    weight            = 50
    capacity_provider = "FARGATE_SPOT"
  }
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "app" {
  name              = "/ecs/${local.name_prefix}"
  retention_in_days = 7

  tags = local.tags
}

# ECS Task Definition
resource "aws_ecs_task_definition" "app" {
  family                   = "${local.name_prefix}-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 2048
  memory                   = 4096
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn           = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "aybiza-app"
      image = "aybiza/app:latest"
      
      cpu    = 2048
      memory = 4096
      
      essential = true
      
      portMappings = [
        {
          containerPort = 4000
          protocol      = "tcp"
        }
      ]
      
      environment = [
        {
          name  = "PORT"
          value = "4000"
        },
        {
          name  = "PHX_HOST"
          value = "api.aybiza.com"
        },
        {
          name  = "DATABASE_URL"
          value = "ecto://postgres:password@${aws_rds_cluster.main.endpoint}:5432/aybiza"
        },
        {
          name  = "REDIS_URL"
          value = "redis://${aws_elasticache_replication_group.main.primary_endpoint_address}:6379"
        }
      ]
      
      secrets = [
        {
          name      = "SECRET_KEY_BASE"
          valueFrom = "${aws_secretsmanager_secret.app_secrets.arn}:SECRET_KEY_BASE::"
        },
        {
          name      = "DEEPGRAM_API_KEY"
          valueFrom = "${aws_secretsmanager_secret.app_secrets.arn}:DEEPGRAM_API_KEY::"
        },
        {
          name      = "TWILIO_ACCOUNT_SID"
          valueFrom = "${aws_secretsmanager_secret.app_secrets.arn}:TWILIO_ACCOUNT_SID::"
        },
        {
          name      = "TWILIO_AUTH_TOKEN"
          valueFrom = "${aws_secretsmanager_secret.app_secrets.arn}:TWILIO_AUTH_TOKEN::"
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.app.name
          awslogs-region        = var.region
          awslogs-stream-prefix = "ecs"
        }
      }
      
      healthCheck = {
        command = [
          "CMD-SHELL",
          "curl -f http://localhost:4000/health || exit 1"
        ]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])

  tags = local.tags
}

# ECS Service
resource "aws_ecs_service" "app" {
  name            = "${local.name_prefix}-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  platform_version = "1.4.0"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = var.security_group_ids
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = var.target_group_arn
    container_name   = "aybiza-app"
    container_port   = 4000
  }

  # Auto Scaling
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight           = 70
    base             = 2
  }

  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight           = 30
    base             = 0
  }

  # Deployment Configuration
  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
    
    deployment_circuit_breaker {
      enable   = true
      rollback = true
    }
  }

  # Service Discovery
  service_registries {
    registry_arn = aws_service_discovery_service.app.arn
  }

  depends_on = [
    aws_iam_role_policy_attachment.ecs_execution,
    aws_iam_role_policy_attachment.ecs_task
  ]

  tags = local.tags
}

# Service Discovery
resource "aws_service_discovery_private_dns_namespace" "main" {
  name        = "${local.name_prefix}.local"
  description = "Private DNS namespace for AYBIZA services"
  vpc         = var.vpc_id

  tags = local.tags
}

resource "aws_service_discovery_service" "app" {
  name = "app"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id

    dns_records {
      ttl  = 10
      type = "A"
    }

    routing_policy = "MULTIVALUE"
  }

  health_check_grace_period_seconds = 60

  tags = local.tags
}

# Auto Scaling
resource "aws_appautoscaling_target" "ecs_target" {
  max_capacity       = 20
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "ecs_policy_cpu" {
  name               = "${local.name_prefix}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }

    target_value       = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 300
  }
}

resource "aws_appautoscaling_policy" "ecs_policy_memory" {
  name               = "${local.name_prefix}-memory-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }

    target_value       = 80.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 300
  }
}

# IAM Roles
resource "aws_iam_role" "ecs_execution" {
  name = "${local.name_prefix}-ecs-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })

  tags = local.tags
}

resource "aws_iam_role" "ecs_task" {
  name = "${local.name_prefix}-ecs-task-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })

  tags = local.tags
}

# IAM Policy Attachments
resource "aws_iam_role_policy_attachment" "ecs_execution" {
  role       = aws_iam_role.ecs_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role_policy" "ecs_execution_custom" {
  name = "${local.name_prefix}-ecs-execution-custom"
  role = aws_iam_role.ecs_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          aws_secretsmanager_secret.app_secrets.arn
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "${aws_cloudwatch_log_group.app.arn}:*"
      }
    ]
  })
}

resource "aws_iam_role_policy" "ecs_task_custom" {
  name = "${local.name_prefix}-ecs-task-custom"
  role = aws_iam_role.ecs_task.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "bedrock:InvokeModel",
          "bedrock:InvokeModelWithResponseStream"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::aybiza-${var.environment}-*/*"
      }
    ]
  })
}

# Secrets Manager
resource "aws_secretsmanager_secret" "app_secrets" {
  name                    = "${local.name_prefix}-app-secrets"
  description             = "Application secrets for AYBIZA"
  recovery_window_in_days = 7

  tags = local.tags
}

resource "aws_secretsmanager_secret_version" "app_secrets" {
  secret_id = aws_secretsmanager_secret.app_secrets.id
  secret_string = jsonencode({
    SECRET_KEY_BASE     = random_password.secret_key_base.result
    DEEPGRAM_API_KEY    = var.deepgram_api_key
    TWILIO_ACCOUNT_SID  = var.twilio_account_sid
    TWILIO_AUTH_TOKEN   = var.twilio_auth_token
  })
}

resource "random_password" "secret_key_base" {
  length  = 64
  special = true
}

# Variables for secrets
variable "deepgram_api_key" {
  description = "Deepgram API key"
  type        = string
  sensitive   = true
}

variable "twilio_account_sid" {
  description = "Twilio Account SID"
  type        = string
  sensitive   = true
}

variable "twilio_auth_token" {
  description = "Twilio Auth Token"
  type        = string
  sensitive   = true
}

# Outputs
output "cluster_id" {
  value = aws_ecs_cluster.main.id
}

output "cluster_name" {
  value = aws_ecs_cluster.main.name
}

output "service_name" {
  value = aws_ecs_service.app.name
}

output "task_definition_arn" {
  value = aws_ecs_task_definition.app.arn
}

output "log_group_name" {
  value = aws_cloudwatch_log_group.app.name
}
```

### 3. Multi-Region Configuration
```hcl
# environments/production/main.tf
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "aybiza-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# Provider configurations for multi-region
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
  
  default_tags {
    tags = {
      Environment = "production"
      Project     = "aybiza"
      ManagedBy   = "terraform"
    }
  }
}

provider "aws" {
  alias  = "us_west_2"
  region = "us-west-2"
  
  default_tags {
    tags = {
      Environment = "production"
      Project     = "aybiza"
      ManagedBy   = "terraform"
    }
  }
}

provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
  
  default_tags {
    tags = {
      Environment = "production"
      Project     = "aybiza"
      ManagedBy   = "terraform"
    }
  }
}

# Data sources
data "aws_availability_zones" "us_east_1" {
  provider = aws.us_east_1
  state    = "available"
}

data "aws_availability_zones" "us_west_2" {
  provider = aws.us_west_2
  state    = "available"
}

data "aws_availability_zones" "eu_west_1" {
  provider = aws.eu_west_1
  state    = "available"
}

# US East 1 (Primary)
module "vpc_us_east_1" {
  source = "../../modules/vpc"
  
  providers = {
    aws = aws.us_east_1
  }

  environment        = "production"
  region            = "us-east-1"
  cidr_block        = "10.0.0.0/16"
  availability_zones = slice(data.aws_availability_zones.us_east_1.names, 0, 3)
}

module "ecs_us_east_1" {
  source = "../../modules/ecs"
  
  providers = {
    aws = aws.us_east_1
  }

  environment           = "production"
  region               = "us-east-1"
  vpc_id               = module.vpc_us_east_1.vpc_id
  private_subnet_ids   = module.vpc_us_east_1.private_app_subnet_ids
  public_subnet_ids    = module.vpc_us_east_1.public_subnet_ids
  target_group_arn     = module.alb_us_east_1.target_group_arn
  security_group_ids   = [module.security_us_east_1.ecs_security_group_id]

  deepgram_api_key    = var.deepgram_api_key
  twilio_account_sid  = var.twilio_account_sid
  twilio_auth_token   = var.twilio_auth_token
}

# US West 2 (Secondary)
module "vpc_us_west_2" {
  source = "../../modules/vpc"
  
  providers = {
    aws = aws.us_west_2
  }

  environment        = "production"
  region            = "us-west-2"
  cidr_block        = "10.1.0.0/16"
  availability_zones = slice(data.aws_availability_zones.us_west_2.names, 0, 3)
}

module "ecs_us_west_2" {
  source = "../../modules/ecs"
  
  providers = {
    aws = aws.us_west_2
  }

  environment           = "production"
  region               = "us-west-2"
  vpc_id               = module.vpc_us_west_2.vpc_id
  private_subnet_ids   = module.vpc_us_west_2.private_app_subnet_ids
  public_subnet_ids    = module.vpc_us_west_2.public_subnet_ids
  target_group_arn     = module.alb_us_west_2.target_group_arn
  security_group_ids   = [module.security_us_west_2.ecs_security_group_id]

  deepgram_api_key    = var.deepgram_api_key
  twilio_account_sid  = var.twilio_account_sid
  twilio_auth_token   = var.twilio_auth_token
}

# EU West 1 (GDPR Compliant)
module "vpc_eu_west_1" {
  source = "../../modules/vpc"
  
  providers = {
    aws = aws.eu_west_1
  }

  environment        = "production"
  region            = "eu-west-1"
  cidr_block        = "10.2.0.0/16"
  availability_zones = slice(data.aws_availability_zones.eu_west_1.names, 0, 3)
}

module "ecs_eu_west_1" {
  source = "../../modules/ecs"
  
  providers = {
    aws = aws.eu_west_1
  }

  environment           = "production"
  region               = "eu-west-1"
  vpc_id               = module.vpc_eu_west_1.vpc_id
  private_subnet_ids   = module.vpc_eu_west_1.private_app_subnet_ids
  public_subnet_ids    = module.vpc_eu_west_1.public_subnet_ids
  target_group_arn     = module.alb_eu_west_1.target_group_arn
  security_group_ids   = [module.security_eu_west_1.ecs_security_group_id]

  deepgram_api_key    = var.deepgram_api_key
  twilio_account_sid  = var.twilio_account_sid
  twilio_auth_token   = var.twilio_auth_token
}

# Global RDS Aurora
module "rds_global" {
  source = "../../modules/rds"
  
  providers = {
    aws.primary   = aws.us_east_1
    aws.secondary = aws.us_west_2
    aws.eu        = aws.eu_west_1
  }

  environment = "production"
  
  primary_vpc_id      = module.vpc_us_east_1.vpc_id
  primary_subnet_ids  = module.vpc_us_east_1.private_db_subnet_ids
  
  secondary_vpc_id    = module.vpc_us_west_2.vpc_id
  secondary_subnet_ids = module.vpc_us_west_2.private_db_subnet_ids
  
  eu_vpc_id          = module.vpc_eu_west_1.vpc_id
  eu_subnet_ids      = module.vpc_eu_west_1.private_db_subnet_ids
}

# Variables
variable "deepgram_api_key" {
  description = "Deepgram API key"
  type        = string
  sensitive   = true
}

variable "twilio_account_sid" {
  description = "Twilio Account SID"
  type        = string
  sensitive   = true
}

variable "twilio_auth_token" {
  description = "Twilio Auth Token"
  type        = string
  sensitive   = true
}

# Outputs
output "us_east_1_cluster_name" {
  value = module.ecs_us_east_1.cluster_name
}

output "us_west_2_cluster_name" {
  value = module.ecs_us_west_2.cluster_name
}

output "eu_west_1_cluster_name" {
  value = module.ecs_eu_west_1.cluster_name
}

output "global_database_endpoint" {
  value = module.rds_global.global_cluster_endpoint
}
```

## Deployment Scripts

### 1. Deployment Script
```bash
#!/bin/bash
# scripts/deploy.sh

set -e

ENVIRONMENT="${1:-production}"
REGION="${2:-us-east-1}"

echo "Deploying AYBIZA infrastructure to $ENVIRONMENT in $REGION"

# Validate Terraform configuration
echo "Validating Terraform configuration..."
terraform -chdir=environments/$ENVIRONMENT validate

# Plan deployment
echo "Planning deployment..."
terraform -chdir=environments/$ENVIRONMENT plan \
  -var-file="$ENVIRONMENT.tfvars" \
  -out="$ENVIRONMENT.tfplan"

# Ask for confirmation
read -p "Do you want to apply this plan? (y/N): " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    echo "Deployment cancelled"
    exit 1
fi

# Apply deployment
echo "Applying deployment..."
terraform -chdir=environments/$ENVIRONMENT apply "$ENVIRONMENT.tfplan"

# Clean up plan file
rm "environments/$ENVIRONMENT/$ENVIRONMENT.tfplan"

echo "Deployment completed successfully!"

# Verify deployment
echo "Verifying deployment..."
aws ecs describe-clusters \
  --region $REGION \
  --clusters "aybiza-$ENVIRONMENT" \
  --query 'clusters[0].status'

echo "Infrastructure deployment verification completed!"
```

### 2. Monitoring Setup Script
```bash
#!/bin/bash
# scripts/setup-monitoring.sh

ENVIRONMENT="${1:-production}"
REGIONS=("us-east-1" "us-west-2" "eu-west-1")

echo "Setting up monitoring for AYBIZA infrastructure"

for region in "${REGIONS[@]}"; do
    echo "Setting up monitoring in $region..."
    
    # Create CloudWatch alarms
    aws cloudwatch put-metric-alarm \
      --region $region \
      --alarm-name "AYBIZA-$ENVIRONMENT-$region-HighCPU" \
      --alarm-description "High CPU utilization" \
      --metric-name CPUUtilization \
      --namespace AWS/ECS \
      --statistic Average \
      --period 300 \
      --threshold 80 \
      --comparison-operator GreaterThanThreshold \
      --evaluation-periods 2 \
      --dimensions Name=ServiceName,Value="aybiza-$ENVIRONMENT-app" Name=ClusterName,Value="aybiza-$ENVIRONMENT" \
      --alarm-actions "arn:aws:sns:$region:$(aws sts get-caller-identity --query Account --output text):aybiza-alerts"
    
    # Create dashboard
    aws cloudwatch put-dashboard \
      --region $region \
      --dashboard-name "AYBIZA-$ENVIRONMENT-$region" \
      --dashboard-body file://monitoring/dashboard.json
done

echo "Monitoring setup completed!"
```

## Security Best Practices

### 1. IAM Policies with Least Privilege
```hcl
# modules/security/iam.tf
resource "aws_iam_policy" "ecs_task_minimal" {
  name        = "${local.name_prefix}-ecs-task-minimal"
  description = "Minimal permissions for ECS tasks"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "bedrock:InvokeModel"
        ]
        Resource = [
          "arn:aws:bedrock:*:*:foundation-model/anthropic.claude-3-*"
        ]
        Condition = {
          StringEquals = {
            "aws:RequestedRegion": ["us-east-1", "us-west-2", "eu-west-1"]
          }
        }
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject"
        ]
        Resource = [
          "arn:aws:s3:::aybiza-${var.environment}-assets/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = [
          "arn:aws:logs:*:*:log-group:/ecs/${local.name_prefix}:*"
        ]
      }
    ]
  })

  tags = local.tags
}
```

### 2. Network Security
```hcl
# modules/security/security_groups.tf
resource "aws_security_group" "ecs_tasks" {
  name_prefix = "${local.name_prefix}-ecs-tasks"
  vpc_id      = var.vpc_id

  # Allow inbound from ALB only
  ingress {
    from_port       = 4000
    to_port         = 4000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  # Allow outbound HTTPS for external APIs
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow outbound to RDS
  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds.id]
  }

  # Allow outbound to Redis
  egress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.redis.id]
  }

  tags = merge(local.tags, {
    Name = "${local.name_prefix}-ecs-tasks"
  })
}
```

This Terraform configuration provides a complete, production-ready infrastructure for the AYBIZA platform with multi-region support, auto-scaling, security best practices, and monitoring capabilities.