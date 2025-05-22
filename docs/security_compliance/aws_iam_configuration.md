# AWS IAM Configuration Guide for AYBIZA Platform

This comprehensive guide details the AWS Identity and Access Management (IAM) setup required for the AYBIZA AI Voice Agent Platform's hybrid Cloudflare+AWS architecture, including user management, roles, policies, multi-region security, and best practices for Claude 3.7 Sonnet integration with ultra-low latency requirements.

## Table of Contents
1. [IAM Overview](#iam-overview)
2. [Account Structure](#account-structure)
3. [User Management](#user-management)
4. [Service Roles](#service-roles)
5. [Policy Configuration](#policy-configuration)
6. [Cross-Account Access](#cross-account-access)
7. [Security Best Practices](#security-best-practices)
8. [Authentication Configuration](#authentication-configuration)
9. [CI/CD Integration](#cicd-integration)
10. [Monitoring and Audit](#monitoring-and-audit)
11. [Implementation Checklist](#implementation-checklist)

## IAM Overview

AWS Identity and Access Management (IAM) enables secure control of access to AWS services and resources across multiple regions. For the AYBIZA platform's hybrid Cloudflare+AWS architecture, we implement a comprehensive IAM strategy to ensure secure operations across edge and cloud components, while maintaining the principle of least privilege and supporting ultra-low latency requirements (<50ms US/UK, <100ms globally).

## Account Structure

### Multi-Account Strategy

For enterprise-grade security and resource isolation, implement a multi-account AWS strategy:

1. **Management Account**:
   - IAM user management
   - Consolidated billing
   - AWS Organizations management
   - Security audit tools
   - Cross-account Cloudflare integration management

2. **Development Account**:
   - Development and testing environments
   - CI/CD pipelines with Cloudflare Workers
   - Developer-specific resources
   - Edge testing environments

3. **Staging Account**:
   - Pre-production environment
   - Performance testing with latency validation
   - Integration testing (Cloudflare+AWS)
   - Claude 3.7 Sonnet testing

4. **Production Account**:
   - Customer-facing production environment
   - Multi-region deployment (us-east-1, us-west-2, eu-west-1)
   - Strict access controls with zero-trust principles
   - High-availability configuration across edge and cloud

5. **Security & Compliance Account**:
   - Centralized security tooling
   - Hybrid architecture monitoring
   - Edge security compliance
   - Log aggregation and analysis
   - Compliance reporting

6. **Shared Services Account**:
   - Common backend services
   - Artifact repositories
   - Backup systems

### Resource Organization

Within each account, use:
- **AWS Resource Groups**: Organize resources by application component
- **Resource Tags**: Apply consistent tagging for:
  - `Application: aybiza`
  - `Environment: dev|staging|prod`
  - `Component: voice-gateway|voice-pipeline|...`
  - `Owner: team-name`
  - `Cost-Center: department-code`

## User Management

### IAM User Types

1. **Human Users**:
   - Individual IAM users for administrative personnel
   - Federated users via corporate identity provider for daily access

2. **Service Users**:
   - Programmatic access for CI/CD pipelines
   - Emergency access users (break-glass accounts)

### User Creation Process

For each human user:

1. Create IAM user without direct permissions
2. Assign to appropriate IAM groups
3. Enforce MFA enrollment
4. Set password policy
5. No permanent credentials for regular users

Example AWS CLI command:
```bash
aws iam create-user --user-name aybiza-admin-user

aws iam add-user-to-group --user-name aybiza-admin-user --group-name AybizaAdmins

aws iam enable-mfa-device --user-name aybiza-admin-user --serial-number arn:aws:iam::123456789012:mfa/aybiza-admin-user --authentication-code-1 123456 --authentication-code-2 789012
```

### IAM Groups

Create the following IAM groups with associated permissions:

| Group Name | Description | Permissions |
|------------|-------------|------------|
| AybizaAdmins | Platform administrators | Administrative access to all AYBIZA resources |
| AybizaDevelopers | Development team | Access to development resources, limited production read access |
| AybizaOps | Operations team | Deployment and monitoring permissions |
| AybizaSecurityTeam | Security personnel | Audit and security tool access |
| AybizaReadOnly | Support and stakeholders | Read-only access to specific resources |
| AybizaDataAnalysts | Data analysis team | Read access to analytics data and dashboards |

### Emergency Access

Create break-glass accounts for emergency use:
- Highly restricted users with necessary permissions
- Secured credentials stored in sealed physical containers
- Monitoring alerts when used
- Process for credential rotation after use

## Service Roles

### AWS CodePipeline Service Roles

1. **AybizaCodePipelineServiceRole**:
   - Core pipeline orchestration permissions
   - Access to S3 artifact store
   - Permission to invoke CodeBuild and CodeDeploy

Example CodePipeline service role policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketVersioning",
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::aybiza-codepipeline-artifacts-*",
        "arn:aws:s3:::aybiza-codepipeline-artifacts-*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "codebuild:BatchGetBuilds",
        "codebuild:StartBuild"
      ],
      "Resource": "arn:aws:codebuild:*:*:project/aybiza-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codedeploy:CreateApplication",
        "codedeploy:CreateDeployment",
        "codedeploy:CreateDeploymentGroup",
        "codedeploy:GetApplication",
        "codedeploy:GetApplicationRevision",
        "codedeploy:GetDeployment",
        "codedeploy:GetDeploymentConfig",
        "codedeploy:RegisterApplicationRevision"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:DescribeTasks",
        "ecs:ListTasks",
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::*:role/AybizaCodeDeployServiceRole",
        "arn:aws:iam::*:role/AybizaECSTaskExecutionRole",
        "arn:aws:iam::*:role/AybizaAppRole"
      ]
    }
  ]
}
```

2. **AybizaCodeBuildServiceRole**:
   - Build project execution permissions
   - ECR push/pull access
   - CloudWatch Logs write access

Example CodeBuild service role policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:/aws/codebuild/aybiza-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:GetAuthorizationToken",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::aybiza-codepipeline-artifacts-*/*",
        "arn:aws:s3:::aybiza-codebuild-cache/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:aybiza/*"
    }
  ]
}
```

3. **AybizaCodeDeployServiceRole**:
   - Blue/green deployment permissions for ECS
   - Load balancer management
   - Auto scaling group management

Example CodeDeploy service role policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:CreateTaskSet",
        "ecs:DeleteTaskSet",
        "ecs:DescribeServices",
        "ecs:UpdateServicePrimaryTaskSet",
        "elasticloadbalancing:DescribeTargetGroups",
        "elasticloadbalancing:DescribeListeners",
        "elasticloadbalancing:ModifyListener",
        "elasticloadbalancing:DescribeRules",
        "elasticloadbalancing:ModifyRule",
        "lambda:InvokeFunction",
        "cloudwatch:DescribeAlarms",
        "sns:Publish",
        "s3:GetObject"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::*:role/AybizaECSTaskExecutionRole",
        "arn:aws:iam::*:role/AybizaAppRole"
      ]
    }
  ]
}
```

### ECS Task Execution Roles

1. **AybizaECSTaskExecutionRole**:
   - Permissions to pull images from ECR
   - CloudWatch Logs write access
   - Secrets Manager access for environment variables

2. **AybizaAppRole**:
   - Application-specific permissions
   - Assumed by the container at runtime

Example TaskExecutionRole policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

### Lambda Execution Roles

1. **AybizaLambdaWebhookRole**:
   - Permissions for webhook processing
   - CloudWatch Logs access
   - SQS access for message processing

2. **AybizaLambdaProcessingRole**:
   - Permissions for background processing
   - S3 access for file operations
   - DynamoDB access for state management

Example Lambda role policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::aybiza-recordings/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/aybiza-sessions"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:*:*:aybiza-processing-queue"
    }
  ]
}
```

### Database Access Roles

1. **AybizaRDSAccessRole**:
   - IAM authentication to RDS
   - Assumed by application services

Example RDS access policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "rds-db:connect",
      "Resource": "arn:aws:rds-db:region:account-id:dbuser:db-resource-id/database_user"
    }
  ]
}
```

### API Gateway Execution Role

1. **AybizaAPIGatewayRole**:
   - Invoke Lambda functions
   - Write to CloudWatch Logs
   - Access VPC resources if needed

Example API Gateway execution policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:region:account-id:function:aybiza-webhook-processor"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

### Bedrock Access Role

1. **AybizaBedrockAccessRole**:
   - Invoke Bedrock models
   - Read from knowledge bases

Example Bedrock access policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:region::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0",
        "arn:aws:bedrock:region::foundation-model/anthropic.claude-3-haiku-20240307-v1:0"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:ListFoundationModels",
        "bedrock:GetFoundationModel"
      ],
      "Resource": "*"
    }
  ]
}
```

## Policy Configuration

### Standard Policies

1. **AybizaS3ReadOnly**:
   - Read-only access to S3 buckets
   - List bucket contents

2. **AybizaS3ReadWrite**:
   - Read and write access to S3 buckets
   - No bucket administration

3. **AybizaDynamoDBReadOnly**:
   - Read-only access to DynamoDB tables
   - Query and scan operations

4. **AybizaDynamoDBReadWrite**:
   - Read and write access to DynamoDB tables
   - No table administration

5. **AybizaSecretsReadOnly**:
   - Read-only access to specific Secrets Manager secrets

### Custom Policies

1. **AybizaVoiceGatewayPolicy**:
   - Permissions needed by the voice gateway component
   - API Gateway management
   - WebSocket management

Example Voice Gateway policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "execute-api:ManageConnections",
        "execute-api:Invoke"
      ],
      "Resource": "arn:aws:execute-api:region:account-id:api-id/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::aybiza-recordings/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage"
      ],
      "Resource": "arn:aws:sqs:region:account-id:aybiza-call-queue"
    }
  ]
}
```

2. **AybizaVoicePipelinePolicy**:
   - Permissions needed by the voice pipeline component
   - Deepgram API access management
   - Media processing permissions

3. **AybizaMonitoringPolicy**:
   - CloudWatch monitoring permissions
   - X-Ray tracing permissions
   - Log group and log stream management

4. **AybizaDeploymentPolicy**:
   - Permissions for CI/CD deployment
   - ECR access for image management
   - ECS task definition and service updates

### Boundary Policies

1. **AybizaRegionBoundary**:
   - Restrict operations to specific AWS regions
   - Apply to all roles as permission boundary

Example Region Boundary policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "NotAction": [
        "cloudfront:*",
        "iam:*",
        "route53:*",
        "support:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "eu-west-1"
          ]
        }
      }
    }
  ]
}
```

2. **AybizaResourceBoundary**:
   - Restrict access to resources with specific tags
   - Enforce consistent resource organization

Example Resource Boundary policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:ResourceTag/Application": "aybiza"
        }
      }
    }
  ]
}
```

## Cross-Account Access

### Organization Access Roles

1. **AybizaCrossAccountRole**:
   - Role in each account for cross-account access
   - Trust policy allowing assumption from management account

Example Trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::MANAGEMENT-ACCOUNT-ID:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxx"
        }
      }
    }
  ]
}
```

### Deployment Pipeline Roles

1. **AybizaDeploymentRole**:
   - Role in each environment account for CI/CD
   - Assumed by deployment pipeline (GitHub Actions OR CodePipeline)
   - Permissions to deploy application components

Example Deployment Role policy (Enhanced for CodePipeline):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:RegisterTaskDefinition",
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:ListTasks",
        "ecs:DescribeTasks"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codedeploy:CreateApplication",
        "codedeploy:CreateDeployment",
        "codedeploy:CreateDeploymentGroup",
        "codedeploy:GetApplication",
        "codedeploy:GetDeployment",
        "codedeploy:GetDeploymentConfig",
        "codedeploy:ListApplications",
        "codedeploy:ListDeployments",
        "codedeploy:StopDeployment"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:DescribeTargetGroups",
        "elasticloadbalancing:DescribeListeners",
        "elasticloadbalancing:ModifyListener",
        "elasticloadbalancing:DescribeRules",
        "elasticloadbalancing:ModifyRule"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::aybiza-codepipeline-artifacts-*/*",
        "arn:aws:s3:::aybiza-deployment-*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::*:role/AybizaECSTaskExecutionRole",
        "arn:aws:iam::*:role/AybizaAppRole",
        "arn:aws:iam::*:role/AybizaCodeDeployServiceRole"
      ]
    }
  ]
}
```

2. **AybizaEventBridgeRole**:
   - Role for EventBridge to trigger CodePipeline
   - Cross-service pipeline orchestration

Example EventBridge role policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codepipeline:StartPipelineExecution"
      ],
      "Resource": "arn:aws:codepipeline:*:*:pipeline/aybiza-*"
    }
  ]
}
```

## Security Best Practices

### Authentication Controls

1. **MFA Enforcement**:
   - Require MFA for all human users
   - Service control policy to enforce MFA

Example SCP to enforce MFA:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BlockMostAccessUnlessSignedInWithMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:ResyncMFADevice",
        "iam:ChangePassword",
        "iam:GetAccountPasswordPolicy",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

2. **Password Policy**:
   - Minimum 14 characters
   - Require symbols, numbers, uppercase and lowercase letters
   - Password expiration every 90 days
   - Password history of 24 passwords
   - No password reuse

AWS CLI command:
```bash
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 24
```

### Access Key Management

1. **Rotation Policy**:
   - 90-day rotation for all access keys
   - Automated alerts for key expiration

2. **Key Monitoring**:
   - Monitor for unused keys (30+ days)
   - Alerts for excessive key usage

AWS CLI commands:
```bash
# List access keys older than 90 days
aws iam list-access-keys --query 'AccessKeyMetadata[?CreateDate<=`2025-02-15`].AccessKeyId' --output text

# Create new access key before rotation
aws iam create-access-key --user-name aybiza-service-user

# Delete old access key after rotation
aws iam delete-access-key --user-name aybiza-service-user --access-key-id AKIAIOSFODNN7EXAMPLE
```

### Least Privilege Implementation

1. **Access Reviews**:
   - Quarterly access reviews for all users and roles
   - Remove unnecessary permissions
   - Validate resource access patterns

2. **IAM Access Analyzer**:
   - Enable in all accounts
   - Review findings weekly
   - Remediate excessive permissions

AWS CLI command:
```bash
aws accessanalyzer create-analyzer \
  --analyzer-name aybiza-analyzer \
  --type ACCOUNT
```

## Authentication Configuration

### Identity Federation

1. **AWS SSO Configuration**:
   - Set up AWS SSO for corporate identity integration
   - Configure permission sets for different roles
   - Map groups from identity provider to AWS SSO

2. **SAML 2.0 Integration**:
   - Configure SAML assertion for your identity provider
   - Set up role mapping from SAML attributes
   - Enable SAML provider in IAM

Example SAML provider creation:
```bash
aws iam create-saml-provider \
  --saml-metadata-document file://saml-metadata.xml \
  --name AybizaCorporateIdentity
```

### Application Authentication

1. **Cognito User Pools**:
   - Set up user pool for application authentication
   - Configure MFA requirements
   - Set password policies

2. **Cognito Identity Pools**:
   - Configure for temporary AWS credentials
   - Set up role mapping for authenticated users
   - Restrict access to required services

Example Cognito User Pool creation:
```bash
aws cognito-idp create-user-pool \
  --pool-name AybizaUserPool \
  --policies '{"PasswordPolicy":{"MinimumLength":12,"RequireUppercase":true,"RequireLowercase":true,"RequireNumbers":true,"RequireSymbols":true}}' \
  --mfa-configuration OPTIONAL \
  --auto-verified-attributes email
```

## CI/CD Integration

### AWS CodePipeline Integration Roles

1. **AybizaCodePipelineArtifactRole**:
   - S3 bucket access for pipeline artifacts
   - Cross-region artifact replication
   - Encryption key management

Example CodePipeline artifact role policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketVersioning",
        "s3:GetBucketLocation",
        "s3:ListBucket",
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::aybiza-codepipeline-artifacts-*",
        "arn:aws:s3:::aybiza-codepipeline-artifacts-*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:*:*:key/*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.*.amazonaws.com"
        }
      }
    }
  ]
}
```

### GitHub Actions Role (Alternative Approach)

1. **AybizaGitHubRole**:
   - Role for GitHub Actions pipeline (when not using CodePipeline)
   - Permissions to deploy to AWS
   - Access to ECR, ECS, S3, etc.

Example GitHub Actions role trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::account-id:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:aybiza/aybiza-platform:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

### CodePipeline Integration with GitHub

1. **AybizaGitHubWebhookRole**:
   - Role for GitHub webhook integration with CodePipeline
   - Permissions to trigger pipeline executions

Example GitHub webhook integration policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codepipeline:StartPipelineExecution"
      ],
      "Resource": "arn:aws:codepipeline:*:*:pipeline/aybiza-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:github-oauth-token*"
    }
  ]
}
```

### Deployment Permissions

1. **Development Environment**:
   - Broader permissions for rapid iteration
   - Access to development resources only

2. **Staging Environment**:
   - More restricted permissions
   - Deployment via approved pipelines only

3. **Production Environment**:
   - Highly restricted permissions
   - Deployment via approved pipelines only
   - Manual approval steps

Example GitHub Actions configuration:
```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy-dev:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::DEV-ACCOUNT-ID:role/AybizaGitHubRole
          role-session-name: github-actions-deployment
          aws-region: us-east-1
      - run: aws sts get-caller-identity

  deploy-staging:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::STAGING-ACCOUNT-ID:role/AybizaGitHubRole
          role-session-name: github-actions-deployment
          aws-region: us-east-1

  deploy-production:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::PROD-ACCOUNT-ID:role/AybizaGitHubRole
          role-session-name: github-actions-deployment
          aws-region: us-east-1
```

## Monitoring and Audit

### IAM Activity Monitoring

1. **CloudTrail Configuration**:
   - Enable CloudTrail in all accounts
   - Configure for all regions
   - Enable log file validation
   - Set up alerts for sensitive actions

AWS CLI command:
```bash
aws cloudtrail create-trail \
  --name aybiza-cloudtrail \
  --s3-bucket-name aybiza-cloudtrail-logs \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name aybiza-cloudtrail
```

2. **CloudWatch Alarms**:
   - Alerts for root account usage
   - Alerts for IAM policy changes
   - Alerts for failed authentication attempts
   - Alerts for changes to security groups

Example CloudWatch alarm for root account usage:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name RootAccountUsageAlarm \
  --alarm-description "Alarm when root account is used" \
  --metric-name RootAccountUsage \
  --namespace CloudTrailMetrics \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:region:account-id:SecurityAlerts
```

### Security Hub Integration

1. **AWS Security Hub**:
   - Enable in all accounts
   - Configure CIS AWS Foundations benchmark
   - Set up compliance standards
   - Regular review of findings

AWS CLI command:
```bash
aws securityhub enable-security-hub \
  --enable-default-standards \
  --tags Key=Application,Value=aybiza
```

### Access Logging and Analysis

1. **S3 Access Logs**:
   - Enable access logging for all S3 buckets
   - Analyze for unauthorized access attempts

2. **CloudFront Logs**:
   - Enable detailed access logging
   - Configure for security analysis

3. **RDS Logs**:
   - Enable audit logging
   - Capture all database actions

4. **WAF Logs**:
   - Enable full logging for WAF
   - Analyze for attack patterns

## Implementation Checklist

Use this checklist to ensure all IAM components are properly configured:

- [ ] **Account Structure**
  - [ ] Create AWS Organization
  - [ ] Set up multi-account structure
  - [ ] Configure Service Control Policies
  - [ ] Set up consolidated billing

- [ ] **User Management**
  - [ ] Create IAM groups
  - [ ] Create administrative users
  - [ ] Configure password policy
  - [ ] Enforce MFA for all users

- [ ] **Service Roles**
  - [ ] Create CodePipeline service roles
  - [ ] Create CodeBuild service roles
  - [ ] Create CodeDeploy service roles
  - [ ] Create ECS task execution roles
  - [ ] Create Lambda execution roles
  - [ ] Create database access roles
  - [ ] Create API Gateway roles
  - [ ] Create Bedrock access roles

- [ ] **Policies**
  - [ ] Create standard policies
  - [ ] Create custom component policies
  - [ ] Configure permission boundaries
  - [ ] Apply resource tags

- [ ] **Cross-Account Access**
  - [ ] Create roles for cross-account access
  - [ ] Configure trust relationships
  - [ ] Set up deployment pipeline roles

- [ ] **Security Controls**
  - [ ] Enable IAM Access Analyzer
  - [ ] Configure CloudTrail
  - [ ] Set up Security Hub
  - [ ] Configure access log analysis

- [ ] **Authentication**
  - [ ] Set up identity federation
  - [ ] Configure Cognito user pools
  - [ ] Set up application authentication

- [ ] **CI/CD Integration**
  - [ ] Create CodePipeline integration roles
  - [ ] Create GitHub Actions roles (alternative approach)
  - [ ] Configure deployment permissions for both pipelines
  - [ ] Set up OIDC identity provider for GitHub Actions
  - [ ] Configure EventBridge rules for pipeline triggers

- [ ] **Monitoring**
  - [ ] Configure CloudWatch alarms
  - [ ] Set up notification channels
  - [ ] Enable compliance reporting

- [ ] **Documentation**
  - [ ] Document IAM architecture
  - [ ] Create user management procedures
  - [ ] Document emergency access process
  - [ ] Create access review templates

This comprehensive IAM configuration provides a secure foundation for the AYBIZA AI Voice Agent Platform, ensuring proper access controls while enabling necessary functionality. Regular reviews and updates to this configuration should be performed as the platform evolves.