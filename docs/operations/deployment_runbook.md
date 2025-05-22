# AYBIZA Production Deployment Runbook

## Overview
This runbook provides step-by-step procedures for deploying the AYBIZA AI voice agent platform to production, covering the hybrid Cloudflare+AWS architecture.

## Pre-Deployment Checklist

### 1. Code Quality Gates
- [ ] All tests pass (unit, integration, E2E)
- [ ] Code coverage > 90% for critical components
- [ ] Security scan passes with no critical vulnerabilities
- [ ] Performance benchmarks meet requirements
- [ ] Documentation updated
- [ ] Changelog updated

### 2. Infrastructure Readiness
- [ ] AWS resources provisioned via Terraform
- [ ] Cloudflare Workers deployed to staging
- [ ] Database migrations tested
- [ ] Environment variables configured
- [ ] SSL certificates valid
- [ ] Monitoring and alerting configured

### 3. External Services
- [ ] Twilio webhook endpoints updated
- [ ] Deepgram API keys rotated
- [ ] AWS Bedrock model access verified
- [ ] DNS records configured
- [ ] CDN cache rules updated

## Deployment Environments

### Environment Hierarchy
```
Development → Staging → Production
     ↓          ↓         ↓
   dev-*     staging    api.aybiza.com
```

### Environment-Specific Configuration
```yaml
# Environment Variables by Stage
development:
  DATABASE_URL: "postgresql://localhost:5432/aybiza_dev"
  REDIS_URL: "redis://localhost:6379/0"
  LOG_LEVEL: "debug"
  
staging:
  DATABASE_URL: "postgresql://staging-db.aybiza.com:5432/aybiza"
  REDIS_URL: "redis://staging-redis.aybiza.com:6379/0"
  LOG_LEVEL: "info"
  
production:
  DATABASE_URL: "postgresql://prod-db.aybiza.com:5432/aybiza"
  REDIS_URL: "redis://prod-redis.aybiza.com:6379/0"
  LOG_LEVEL: "warn"
```

## Deployment Process

### 1. Pre-Deployment Steps

#### A. Create Release Branch
```bash
# Create release branch from main
git checkout main
git pull origin main
git checkout -b release/v$(cat VERSION)

# Verify version consistency
echo "Current version: $(cat VERSION)"
echo "Git tag: $(git describe --tags --abbrev=0)"
```

#### B. Run Pre-Deployment Tests
```bash
# Full test suite
mix test --include integration --include e2e

# Security scan
mix deps.audit
mix sobelow --config

# Performance benchmarks
mix benchmark.run
```

#### C. Build and Tag Docker Images
```bash
# Build production images
docker build -t aybiza/app:$(cat VERSION) .
docker build -t aybiza/nginx:$(cat VERSION) ./nginx

# Tag as latest
docker tag aybiza/app:$(cat VERSION) aybiza/app:latest
docker tag aybiza/nginx:$(cat VERSION) aybiza/nginx:latest

# Push to registry
docker push aybiza/app:$(cat VERSION)
docker push aybiza/app:latest
docker push aybiza/nginx:$(cat VERSION)
docker push aybiza/nginx:latest
```

### 2. Infrastructure Deployment

#### A. Deploy Infrastructure Updates
```bash
# Navigate to terraform directory
cd terraform/

# Plan infrastructure changes
terraform plan -var-file="environments/production.tfvars"

# Apply infrastructure changes
terraform apply -var-file="environments/production.tfvars"

# Verify infrastructure health
terraform output | grep -E "(database_endpoint|redis_endpoint|load_balancer_dns)"
```

#### B. Database Migrations
```bash
# Run database migrations
kubectl exec -it deployment/aybiza-app -- mix ecto.migrate

# Verify migration status
kubectl exec -it deployment/aybiza-app -- mix ecto.migrations
```

### 3. Edge Deployment (Cloudflare Workers)

#### A. Deploy Cloudflare Workers
```bash
cd cloudflare-workers/

# Deploy voice router worker
wrangler publish --env production --name voice-router

# Deploy cache manager worker  
wrangler publish --env production --name cache-manager

# Deploy auth validator worker
wrangler publish --env production --name auth-validator

# Verify deployment
wrangler deployments list
```

#### B. Update DNS and Routing
```bash
# Update DNS records
curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "A",
    "name": "api",
    "content": "'$NEW_ORIGIN_IP'",
    "ttl": 300
  }'

# Update load balancer pools
wrangler r2 put aybiza-config/load-balancer.json --file ./config/load-balancer-prod.json
```

### 4. Backend Deployment (AWS ECS)

#### A. Update ECS Service
```bash
# Update task definition
aws ecs register-task-definition \
  --cli-input-json file://ecs-task-definition.json

# Update service with new task definition
aws ecs update-service \
  --cluster aybiza-production \
  --service aybiza-app \
  --task-definition aybiza-app:$(cat VERSION | tr '.' '-')

# Wait for deployment to complete
aws ecs wait services-stable \
  --cluster aybiza-production \
  --services aybiza-app
```

#### B. Verify Service Health
```bash
# Check service status
aws ecs describe-services \
  --cluster aybiza-production \
  --services aybiza-app \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount}'

# Check application health
curl -f https://api.aybiza.com/health
curl -f https://api.aybiza.com/voice/health
```

### 5. Post-Deployment Verification

#### A. Smoke Tests
```bash
# Run smoke test suite
mix test test/smoke/

# Test critical endpoints
curl -f https://api.aybiza.com/api/v1/agents
curl -f https://api.aybiza.com/api/v1/phone-numbers

# Test voice pipeline
curl -X POST https://api.aybiza.com/voice/test \
  -H "Content-Type: application/json" \
  -d '{"test_audio": "base64_encoded_audio"}'
```

#### B. Performance Verification
```bash
# Run performance tests
artillery run test/performance/load-test.yml

# Check response times
curl -w "Response time: %{time_total}s\n" https://api.aybiza.com/health

# Verify latency requirements
mix benchmark.voice_latency
```

#### C. Security Verification
```bash
# SSL/TLS verification
curl -vI https://api.aybiza.com 2>&1 | grep -E "(SSL|TLS)"

# Security headers check
curl -I https://api.aybiza.com | grep -E "(X-Frame-Options|X-Content-Type-Options|Strict-Transport-Security)"

# WAF verification
curl -H "User-Agent: <script>alert('xss')</script>" https://api.aybiza.com/
# Should return 403 or block the request
```

## Rollback Procedures

### 1. Emergency Rollback (< 5 minutes)

#### A. Cloudflare Workers Rollback
```bash
# List recent deployments
wrangler deployments list --name voice-router

# Rollback to previous version
wrangler rollback --deployment-id $PREVIOUS_DEPLOYMENT_ID
```

#### B. ECS Service Rollback
```bash
# Rollback to previous task definition
aws ecs update-service \
  --cluster aybiza-production \
  --service aybiza-app \
  --task-definition aybiza-app:$PREVIOUS_VERSION

# Force new deployment
aws ecs update-service \
  --cluster aybiza-production \
  --service aybiza-app \
  --force-new-deployment
```

### 2. Database Rollback (if needed)
```bash
# Rollback migrations (CAUTION: Data loss possible)
kubectl exec -it deployment/aybiza-app -- mix ecto.rollback --to $PREVIOUS_MIGRATION

# Restore from backup (if necessary)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier aybiza-prod-rollback \
  --db-snapshot-identifier aybiza-prod-backup-$(date -d "1 hour ago" +%Y%m%d%H)
```

## Monitoring and Alerting

### 1. Deployment Monitoring
```bash
# Monitor deployment progress
watch 'aws ecs describe-services --cluster aybiza-production --services aybiza-app --query "services[0].events[0:5]"'

# Monitor Cloudflare Workers
curl -H "Authorization: Bearer $CF_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/scripts/voice-router/metrics"

# Monitor application metrics
curl https://api.aybiza.com/metrics | grep -E "(http_requests_total|voice_calls_active)"
```

### 2. Post-Deployment Alerts
```yaml
# CloudWatch Alarms
HTTPErrorRate:
  threshold: 5%
  period: 300 seconds
  
VoiceProcessingLatency:
  threshold: 100ms
  period: 60 seconds
  
DatabaseConnections:
  threshold: 80%
  period: 300 seconds
```

## Blue-Green Deployment Strategy

### 1. Blue-Green Setup
```bash
# Deploy to green environment
aws ecs create-service \
  --cluster aybiza-production \
  --service-name aybiza-app-green \
  --task-definition aybiza-app:$(cat VERSION | tr '.' '-') \
  --desired-count 3

# Wait for green environment to be healthy
aws ecs wait services-stable \
  --cluster aybiza-production \
  --services aybiza-app-green
```

### 2. Traffic Switching
```bash
# Switch load balancer target
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,TargetGroupArn=$GREEN_TARGET_GROUP_ARN

# Verify traffic switch
curl -H "X-Blue-Green-Test: true" https://api.aybiza.com/health
```

### 3. Cleanup Old Environment
```bash
# After successful verification, remove blue environment
aws ecs update-service \
  --cluster aybiza-production \
  --service aybiza-app-blue \
  --desired-count 0

aws ecs delete-service \
  --cluster aybiza-production \
  --service aybiza-app-blue
```

## Environment-Specific Procedures

### Production Deployment Approval
1. **Technical Review**: Architecture and code review
2. **Security Review**: Security team approval
3. **Business Review**: Product owner approval
4. **Change Control**: Change management board approval (for major releases)

### Staging Deployment (Automated)
```yaml
# GitHub Actions Workflow
name: Deploy to Staging
on:
  push:
    branches: [ main ]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster aybiza-staging \
            --service aybiza-app \
            --force-new-deployment
```

## Disaster Recovery

### 1. Region Failover
```bash
# Switch to backup region
aws route53 change-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --change-batch file://failover-changeset.json

# Update Cloudflare load balancer
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/load_balancers/$LB_ID" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  --data '{"default_pools": ["us-west-2-pool"]}'
```

### 2. Database Failover
```bash
# Promote read replica to primary
aws rds promote-read-replica \
  --db-instance-identifier aybiza-prod-replica-west

# Update connection strings
kubectl create secret generic database-config \
  --from-literal=DATABASE_URL=$NEW_DATABASE_URL \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Communication Plan

### 1. Deployment Notifications
- **Start**: Notify #engineering-alerts channel
- **Issues**: Escalate to #incident-response
- **Complete**: Post summary to #deployments

### 2. Stakeholder Communication
```bash
# Deployment announcement template
echo "🚀 DEPLOYMENT STARTING
Version: $(cat VERSION)
Components: Backend, Edge Workers, Database
Expected Duration: 15 minutes
Monitoring: https://grafana.aybiza.com/deployment-dashboard"
```

## Post-Deployment Tasks

### 1. Performance Validation
- Monitor response times for 1 hour post-deployment
- Verify latency metrics meet SLA requirements
- Check error rates and success metrics

### 2. Documentation Updates
- Update deployment changelog
- Document any issues encountered
- Update runbook based on lessons learned

### 3. Metrics Collection
- Deployment duration and success rate
- Rollback frequency and reasons
- Performance impact analysis

## Emergency Contacts

### On-Call Rotation
- **Primary**: DevOps Engineer on-call
- **Secondary**: Platform Engineer
- **Escalation**: Engineering Manager
- **Executive**: CTO (for business-critical issues)

### External Contacts
- **AWS Support**: [Support case portal]
- **Cloudflare Support**: [Enterprise support line]
- **Twilio Support**: [Priority support]
- **Deepgram Support**: [Technical support]

## Appendices

### A. Environment Variables Reference
### B. Service Dependencies Map
### C. Troubleshooting Common Issues
### D. Performance Baseline Metrics
### E. Security Configuration Checklist