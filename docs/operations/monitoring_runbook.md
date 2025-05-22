# AYBIZA Monitoring and Observability Runbook

## Overview
This runbook defines monitoring strategies, alerting procedures, and troubleshooting guides for the AYBIZA AI voice agent platform's hybrid Cloudflare+AWS architecture.

## Monitoring Architecture

### 1. Multi-Layer Monitoring Stack
```
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   Edge Layer    │ │  Backend Layer  │ │ Business Layer  │
│  (Cloudflare)   │ │     (AWS)       │ │   (Analytics)   │
├─────────────────┤ ├─────────────────┤ ├─────────────────┤
│ • Analytics     │ │ • CloudWatch    │ │ • Call Metrics  │
│ • RUM           │ │ • X-Ray         │ │ • Agent KPIs    │
│ • Logs          │ │ • Container     │ │ • Revenue       │
│ • Security      │ │   Insights      │ │ • Quality       │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### 2. Data Flow
```
Application Metrics → CloudWatch → Grafana Dashboard
Edge Metrics → Cloudflare Analytics → Custom Dashboard  
Business Metrics → TimescaleDB → Analytics Dashboard
Security Events → SIEM → Security Dashboard
```

## Key Performance Indicators (KPIs)

### 1. Technical KPIs

#### Voice Processing Metrics
- **End-to-End Latency**: < 100ms (95th percentile)
- **STT Processing Time**: < 50ms (95th percentile)  
- **LLM Response Time**: < 200ms (95th percentile)
- **TTS Generation Time**: < 50ms (95th percentile)
- **Audio Quality Score**: > 4.0/5.0

#### System Performance Metrics
- **API Response Time**: < 200ms (95th percentile)
- **Database Query Time**: < 50ms (95th percentile)
- **Cache Hit Ratio**: > 90%
- **Error Rate**: < 1%
- **Availability**: 99.99%

#### Infrastructure Metrics
- **CPU Utilization**: < 70% average
- **Memory Utilization**: < 80% average
- **Network Latency**: < 10ms between services
- **Disk I/O**: < 80% utilization
- **Connection Pool**: < 80% utilization

### 2. Business KPIs

#### Call Quality Metrics
- **Call Success Rate**: > 98%
- **Call Completion Rate**: > 95%
- **Customer Satisfaction**: > 4.5/5.0
- **Agent Response Accuracy**: > 90%
- **Conversation Flow Score**: > 4.0/5.0

#### Operational Metrics
- **Concurrent Call Capacity**: 10,000+ per region
- **Daily Call Volume**: Track trends and capacity
- **Revenue per Call**: Track business impact
- **Cost per Call**: Monitor efficiency
- **Agent Utilization**: Track resource usage

## Monitoring Implementation

### 1. CloudWatch Configuration

#### Custom Metrics Collection
```python
# Voice Pipeline Metrics
import boto3

cloudwatch = boto3.client('cloudwatch')

def publish_voice_latency(latency_ms, call_id):
    cloudwatch.put_metric_data(
        Namespace='AYBIZA/Voice',
        MetricData=[
            {
                'MetricName': 'ProcessingLatency',
                'Dimensions': [
                    {
                        'Name': 'CallId',
                        'Value': call_id
                    }
                ],
                'Value': latency_ms,
                'Unit': 'Milliseconds'
            }
        ]
    )
```

#### Elixir Telemetry Integration
```elixir
# Telemetry Events
defmodule Aybiza.Telemetry do
  def setup do
    events = [
      [:aybiza, :voice_pipeline, :processing, :duration],
      [:aybiza, :database, :query, :duration],
      [:aybiza, :external_api, :request, :duration],
      [:aybiza, :call, :started],
      [:aybiza, :call, :completed],
      [:aybiza, :call, :failed]
    ]

    :telemetry.attach_many(
      "aybiza-telemetry-handler",
      events,
      &__MODULE__.handle_event/4,
      nil
    )
  end

  def handle_event([:aybiza, :voice_pipeline, :processing, :duration], measurements, metadata, _config) do
    # Send to CloudWatch
    AWS.CloudWatch.put_metric_data(%{
      namespace: "AYBIZA/Voice",
      metric_data: [
        %{
          metric_name: "ProcessingDuration",
          value: measurements.duration / 1_000_000, # Convert to milliseconds
          unit: "Milliseconds",
          dimensions: [
            %{name: "Agent", value: metadata.agent_id},
            %{name: "Region", value: metadata.region}
          ]
        }
      ]
    })
  end
end
```

### 2. Cloudflare Analytics Setup

#### Custom Analytics Events
```javascript
// Cloudflare Worker Analytics
export default {
  async fetch(request, env, ctx) {
    const startTime = Date.now();
    
    try {
      const response = await handleRequest(request, env);
      
      // Track successful request
      ctx.waitUntil(
        env.ANALYTICS.writeDataPoint({
          blobs: [
            request.cf.country,
            request.headers.get('user-agent'),
            response.status.toString()
          ],
          doubles: [
            Date.now() - startTime, // Response time
            response.headers.get('content-length') || 0
          ],
          indexes: [request.cf.colo] // Cloudflare data center
        })
      );
      
      return response;
    } catch (error) {
      // Track error
      ctx.waitUntil(
        env.ANALYTICS.writeDataPoint({
          blobs: ['error', error.message, request.url],
          doubles: [Date.now() - startTime]
        })
      );
      
      throw error;
    }
  }
};
```

### 3. Application Performance Monitoring (APM)

#### Distributed Tracing with X-Ray
```elixir
# X-Ray Tracing Middleware
defmodule AybizaWeb.XRayPlug do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    trace_id = generate_trace_id()
    
    conn
    |> put_resp_header("x-trace-id", trace_id)
    |> assign(:trace_id, trace_id)
    |> start_xray_segment(trace_id)
  end
  
  defp start_xray_segment(conn, trace_id) do
    segment = %{
      id: generate_segment_id(),
      name: "aybiza-web-request",
      start_time: System.system_time(:seconds),
      trace_id: trace_id,
      http: %{
        request: %{
          method: conn.method,
          url: request_url(conn)
        }
      }
    }
    
    # Send to X-Ray
    AWS.XRay.put_trace_segments([segment])
    
    conn
  end
end
```

## Alerting Rules

### 1. Critical Alerts (PagerDuty)

#### System Health Alerts
```yaml
# CloudWatch Alarms
VoiceProcessingLatencyHigh:
  metric: AYBIZA/Voice/ProcessingLatency
  statistic: Average
  threshold: 100
  period: 60
  evaluation_periods: 2
  comparison: GreaterThanThreshold
  actions:
    - !Ref PagerDutyTopic

ErrorRateHigh:
  metric: AWS/ApplicationELB/HTTPCode_ELB_5XX_Count
  statistic: Sum
  threshold: 10
  period: 300
  evaluation_periods: 1
  comparison: GreaterThanThreshold

DatabaseConnectionsHigh:
  metric: AWS/RDS/DatabaseConnections
  statistic: Average
  threshold: 80
  period: 300
  evaluation_periods: 2
  comparison: GreaterThanThreshold
```

#### Edge Performance Alerts
```javascript
// Cloudflare Analytics Alert Rules
const ALERT_RULES = {
  errorRateHigh: {
    metric: 'requests',
    filter: 'statusCode >= 500',
    threshold: 0.05, // 5% error rate
    window: '5m'
  },
  
  latencyHigh: {
    metric: 'responseTime',
    percentile: 95,
    threshold: 500, // 500ms
    window: '5m'
  },
  
  cacheHitRateLow: {
    metric: 'cacheHitRatio',
    threshold: 0.85, // 85%
    window: '10m'
  }
};
```

### 2. Warning Alerts (Slack)

#### Performance Degradation
```elixir
# Performance Monitoring
defmodule Aybiza.Monitoring.PerformanceWatcher do
  use GenServer
  
  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end
  
  def init(state) do
    # Check performance every minute
    :timer.send_interval(60_000, :check_performance)
    {:ok, state}
  end
  
  def handle_info(:check_performance, state) do
    metrics = collect_performance_metrics()
    
    # Check thresholds
    if metrics.avg_latency > 80 do
      alert(:warning, "Voice processing latency elevated: #{metrics.avg_latency}ms")
    end
    
    if metrics.memory_usage > 75 do
      alert(:warning, "Memory usage high: #{metrics.memory_usage}%")
    end
    
    {:noreply, state}
  end
  
  defp alert(level, message) do
    Slack.send_message("#alerts", "#{level}: #{message}")
    Logger.warn("Performance alert: #{message}")
  end
end
```

### 3. Business Alerts

#### Revenue Impact Monitoring
```sql
-- Daily Revenue Tracking
CREATE OR REPLACE FUNCTION check_revenue_impact()
RETURNS void AS $$
DECLARE
  current_revenue numeric;
  yesterday_revenue numeric;
  variance_pct numeric;
BEGIN
  -- Calculate today's revenue so far
  SELECT SUM(call_cost) INTO current_revenue
  FROM calls 
  WHERE created_at >= CURRENT_DATE;
  
  -- Calculate yesterday's revenue at same time
  SELECT SUM(call_cost) INTO yesterday_revenue
  FROM calls 
  WHERE created_at >= CURRENT_DATE - INTERVAL '1 day'
    AND created_at < CURRENT_DATE + (CURRENT_TIME - INTERVAL '24 hours');
  
  -- Calculate variance
  variance_pct := ((current_revenue - yesterday_revenue) / yesterday_revenue) * 100;
  
  -- Alert if significant variance
  IF ABS(variance_pct) > 20 THEN
    PERFORM pg_notify('revenue_alert', 
      'Revenue variance: ' || variance_pct::text || '%');
  END IF;
END;
$$ LANGUAGE plpgsql;
```

## Dashboard Configuration

### 1. Executive Dashboard
```json
{
  "dashboard": "AYBIZA Executive Overview",
  "panels": [
    {
      "title": "Active Calls",
      "type": "stat",
      "targets": [
        {
          "query": "sum(aybiza_active_calls)"
        }
      ]
    },
    {
      "title": "Daily Revenue",
      "type": "stat", 
      "targets": [
        {
          "query": "sum(increase(aybiza_call_revenue[24h]))"
        }
      ]
    },
    {
      "title": "System Health",
      "type": "gauge",
      "targets": [
        {
          "query": "(1 - rate(aybiza_errors_total[5m])) * 100"
        }
      ]
    },
    {
      "title": "Voice Quality Score",
      "type": "gauge",
      "targets": [
        {
          "query": "avg(aybiza_voice_quality_score)"
        }
      ]
    }
  ]
}
```

### 2. Technical Operations Dashboard
```json
{
  "dashboard": "AYBIZA Technical Operations",
  "panels": [
    {
      "title": "Voice Processing Latency",
      "type": "graph",
      "targets": [
        {
          "query": "histogram_quantile(0.95, aybiza_voice_processing_duration_seconds_bucket)"
        }
      ]
    },
    {
      "title": "Database Performance",
      "type": "graph",
      "targets": [
        {
          "query": "avg(aws_rds_database_connections_average)",
          "legendFormat": "Connections"
        },
        {
          "query": "avg(aws_rds_cpu_utilization_average)",
          "legendFormat": "CPU %"
        }
      ]
    },
    {
      "title": "Error Rate by Service",
      "type": "graph",
      "targets": [
        {
          "query": "rate(aybiza_http_requests_total{status=~\"5..\"}[5m]) by (service)"
        }
      ]
    }
  ]
}
```

### 3. Business Analytics Dashboard
```json
{
  "dashboard": "AYBIZA Business Analytics",
  "panels": [
    {
      "title": "Call Volume Trends",
      "type": "graph",
      "targets": [
        {
          "query": "sum(increase(aybiza_calls_total[1h])) by (agent_type)"
        }
      ]
    },
    {
      "title": "Customer Satisfaction",
      "type": "stat",
      "targets": [
        {
          "query": "avg(aybiza_customer_satisfaction_score)"
        }
      ]
    },
    {
      "title": "Revenue by Region",
      "type": "pie",
      "targets": [
        {
          "query": "sum(aybiza_revenue) by (region)"
        }
      ]
    }
  ]
}
```

## Troubleshooting Procedures

### 1. High Latency Investigation

#### Step 1: Identify Bottleneck
```bash
# Check voice processing pipeline latency
curl -s https://api.aybiza.com/metrics | grep voice_processing_duration

# Check database query performance
aws logs filter-log-events \
  --log-group-name /aws/ecs/aybiza-app \
  --filter-pattern "SLOW_QUERY" \
  --start-time $(date -d "1 hour ago" +%s)000

# Check external service latency
curl -s https://api.aybiza.com/metrics | grep external_api_duration
```

#### Step 2: Drill Down Analysis
```elixir
# Enable detailed tracing
mix telemetry.enable --detailed --duration 300

# Check specific components
iex> Aybiza.Monitoring.get_component_metrics(:voice_pipeline)
%{
  avg_latency: 85.2,
  p95_latency: 124.5,
  error_rate: 0.02,
  throughput: 1250
}
```

#### Step 3: Remediation Actions
```bash
# Scale up ECS service if CPU/memory bound
aws ecs update-service \
  --cluster aybiza-production \
  --service aybiza-app \
  --desired-count 6

# Clear Redis cache if cache-related
redis-cli -h prod-redis.aybiza.com FLUSHDB

# Restart services if memory leak suspected
kubectl rollout restart deployment/aybiza-app
```

### 2. Error Rate Investigation

#### Step 1: Error Analysis
```bash
# Get error breakdown by type
aws logs insights start-query \
  --log-group-name /aws/ecs/aybiza-app \
  --start-time $(date -d "1 hour ago" +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, level, message, error_type
    | filter level = "ERROR"
    | stats count() by error_type
    | sort count desc
  '
```

#### Step 2: Root Cause Analysis
```elixir
# Check for common error patterns
iex> Aybiza.ErrorAnalyzer.get_error_patterns(last: :hour)
[
  %{
    error_type: "timeout",
    count: 45,
    component: "deepgram_client",
    trend: :increasing
  },
  %{
    error_type: "connection_pool_timeout", 
    count: 23,
    component: "database",
    trend: :stable
  }
]
```

### 3. Capacity Planning

#### Current Utilization Assessment
```bash
# Check current resource utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=aybiza-app \
  --start-time $(date -d "24 hours ago" --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics Average,Maximum

# Check call volume trends
aws timestream-query query \
  --query-string "
    SELECT time, COUNT(*) as call_count
    FROM aybiza.calls 
    WHERE time > ago(7d)
    GROUP BY bin(time, 1h)
    ORDER BY time DESC
  "
```

#### Predictive Scaling
```python
# Capacity prediction model
import pandas as pd
from sklearn.linear_model import LinearRegression

def predict_capacity_needs():
    # Load historical metrics
    df = load_call_metrics(days=30)
    
    # Create features
    df['hour'] = pd.to_datetime(df['timestamp']).dt.hour
    df['day_of_week'] = pd.to_datetime(df['timestamp']).dt.dayofweek
    
    # Train model
    X = df[['hour', 'day_of_week', 'trend']]
    y = df['call_count']
    
    model = LinearRegression()
    model.fit(X, y)
    
    # Predict next week
    future_X = generate_future_features(days=7)
    predictions = model.predict(future_X)
    
    # Calculate required capacity
    max_predicted_calls = max(predictions)
    required_instances = math.ceil(max_predicted_calls / CALLS_PER_INSTANCE)
    
    return {
        'current_instances': get_current_instance_count(),
        'required_instances': required_instances,
        'scale_up_needed': required_instances > get_current_instance_count()
    }
```

## Incident Response Procedures

### 1. Incident Classification
- **P1 (Critical)**: System down, major functionality unavailable
- **P2 (High)**: Degraded performance, partial functionality affected  
- **P3 (Medium)**: Minor issues, workarounds available
- **P4 (Low)**: Cosmetic issues, minimal impact

### 2. Response Timeline
- **P1**: Response within 15 minutes, resolution target 1 hour
- **P2**: Response within 30 minutes, resolution target 4 hours
- **P3**: Response within 2 hours, resolution target 24 hours
- **P4**: Response within 24 hours, resolution target 1 week

### 3. Communication Procedures
```bash
# Incident announcement template
cat << EOF
🚨 INCIDENT ALERT - P1
Title: Voice processing system degraded
Impact: 30% of calls experiencing high latency
Started: $(date)
Commander: @john.doe
Status Page: https://status.aybiza.com
EOF
```

## Maintenance Windows

### 1. Scheduled Maintenance
- **Weekly**: Database maintenance (Sundays 2-4 AM UTC)
- **Monthly**: Security updates (First Saturday 1-3 AM UTC)
- **Quarterly**: Infrastructure updates (TBD with advance notice)

### 2. Maintenance Procedures
```bash
# Pre-maintenance checklist
./scripts/pre-maintenance-check.sh

# Enable maintenance mode
curl -X POST https://api.aybiza.com/admin/maintenance-mode \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"enabled": true, "message": "Scheduled maintenance in progress"}'

# Perform maintenance tasks
./scripts/database-maintenance.sh
./scripts/security-updates.sh

# Disable maintenance mode
curl -X POST https://api.aybiza.com/admin/maintenance-mode \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"enabled": false}'

# Post-maintenance verification
./scripts/post-maintenance-check.sh
```

## Contact Information

### On-Call Rotation
- **Primary**: Platform Engineer (24/7)
- **Secondary**: Senior DevOps Engineer
- **Escalation**: Engineering Manager
- **Executive**: CTO (P1 incidents only)

### External Support
- **AWS Support**: Enterprise support case
- **Cloudflare Support**: Enterprise support line
- **PagerDuty**: Incident management platform

## Appendices

### A. Metric Definitions and Calculations
### B. Alert Runbook Templates  
### C. Dashboard JSON Configurations
### D. Troubleshooting Decision Trees
### E. Historical Incident Analysis