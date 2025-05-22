# AYBIZA Observability and Monitoring Guide (Updated)

This guide outlines the observability and monitoring strategy for the AYBIZA platform's hybrid Cloudflare+AWS architecture, enabling proactive identification and resolution of issues across edge and cloud components.

## Enhanced Monitoring Stack

The AYBIZA platform uses a comprehensive monitoring stack optimized for hybrid architecture:

- **Metrics**: Prometheus with Grafana for visualization + Cloudflare Analytics
- **Logging**: Loki for log aggregation and querying + Cloudflare Logs
- **Tracing**: OpenTelemetry with AWS X-Ray + Cloudflare distributed tracing
- **Alerting**: AlertManager with PagerDuty integration + Cloudflare Health Checks
- **Dashboards**: Grafana for unified visualization + custom hybrid dashboards
- **Edge Monitoring**: Cloudflare Workers Analytics and Real User Monitoring (RUM)
- **Cost Monitoring**: AWS Cost Explorer + Cloudflare billing integration

## Key Metrics to Monitor

### System Metrics

| Metric | Description | Normal Range | Alert Threshold |
|--------|-------------|--------------|-----------------|
| CPU Usage | CPU usage per container/node | 0-70% | >85% for 5min |
| Memory Usage | Memory usage per container/node | 0-70% | >85% for 5min |
| Network I/O | Network traffic in/out | Varies | >90% bandwidth |
| Disk Usage | Disk space usage | 0-70% | >85% |
| BEAM Process Count | Number of Erlang processes | 0-1M | >2M |
| BEAM Memory Usage | Memory used by BEAM VM | 0-70% | >85% for 5min |

### Application Metrics

| Metric | Description | Normal Range | Alert Threshold |
|--------|-------------|--------------|-----------------|
| Call Rate | Calls per minute | 0-10K | >15K or <10 |
| Call Duration | Average call duration | 30s-5min | >10min |
| Call Success Rate | Percentage of successful calls | 98-100% | <95% |
| Call Error Rate | Percentage of failed calls | 0-2% | >5% |
| Webhook Response Time | Time to respond to Twilio webhooks | 0-100ms | >200ms |
| DB Query Time | Database query execution time | 0-50ms | >100ms |
| API Response Time | External API response time | Varies | >500ms |

### Voice Pipeline Metrics (Enhanced for Hybrid Architecture)

| Metric | Description | Normal Range (Edge) | Normal Range (Cloud) | Alert Threshold |
|--------|-------------|--------------------|--------------------|-----------------|
| STT Latency (Nova-3) | Time for speech-to-text conversion | 15-25ms | 20-35ms | >50ms |
| LLM Latency (Claude 3.7 Sonnet) | Time for LLM response generation | N/A | 120-180ms | >300ms |
| LLM Latency (Claude 3.5 Sonnet) | Time for LLM response generation | 80-120ms | 100-150ms | >250ms |
| LLM Latency (Claude 3.5 Haiku) | Time for LLM response generation | 40-60ms | 60-90ms | >150ms |
| TTS Latency (Aura-2) | Time for text-to-speech conversion | 15-25ms | 20-30ms | >50ms |
| **Total Pipeline Latency** | **End-to-end voice processing time** | **90-140ms** | **150-230ms** | **>350ms** |
| Edge Processing Rate | Percentage of requests processed at edge | 60-80% | N/A | <40% |
| Handoff Success Rate | Edge-to-cloud handoff success rate | N/A | 98-100% | <95% |
| Transcription Confidence | Average confidence score (Nova-3) | 0.85-1.0 | 0.85-1.0 | <0.7 |
| Audio Quality | Measured audio quality score | 4.0-5.0 | 4.0-5.0 | <3.5 |
| Cost Optimization Rate | Percentage of cost-optimized requests | 70-90% | 70-90% | <50% |

## Telemetry Implementation

AYBIZA uses `:telemetry` for metrics instrumentation:

```elixir
# Example telemetry event for STT latency
:telemetry.execute(
  [:aybiza, :voice_pipeline, :stt, :complete],
  %{latency: latency_ms},
  %{call_id: call_id, tenant_id: tenant_id}
)
```

### Enhanced Telemetry Events (Hybrid Architecture)

| Event | Measurements | Metadata |
|-------|--------------|----------|
| `[:aybiza, :call, :start]` | `%{timestamp: timestamp}` | `%{call_id: call_id, tenant_id: tenant_id, agent_id: agent_id, edge_region: edge_region, processing_mode: processing_mode}` |
| `[:aybiza, :call, :end]` | `%{duration: duration_ms, cost: cost_usd}` | `%{call_id: call_id, tenant_id: tenant_id, agent_id: agent_id, status: status, cost_optimized: boolean}` |
| `[:aybiza, :voice_pipeline, :stt, :complete]` | `%{latency: latency_ms, confidence: confidence}` | `%{call_id: call_id, model: "nova-3", processing_location: location}` |
| `[:aybiza, :voice_pipeline, :llm, :complete]` | `%{latency: latency_ms, tokens: token_count, thinking_tokens: thinking_tokens}` | `%{call_id: call_id, model: model_id, processing_location: location, thinking_enabled: boolean}` |
| `[:aybiza, :voice_pipeline, :tts, :complete]` | `%{latency: latency_ms, audio_length: length_ms}` | `%{call_id: call_id, voice: voice_id, processing_location: location}` |
| `[:aybiza, :hybrid, :handoff]` | `%{handoff_time: time_ms}` | `%{call_id: call_id, reason: reason, from: from_location, to: to_location}` |
| `[:aybiza, :edge, :processing]` | `%{processing_time: time_ms}` | `%{edge_region: region, complexity: complexity, success: boolean}` |
| `[:aybiza, :cost, :optimization]` | `%{savings: savings_usd, percentage: percentage}` | `%{strategy: strategy, model_selected: model}` |
| `[:aybiza, :db, :query]` | `%{query_time: query_time_ms}` | `%{query: query, table: table, connection_pool: pool_id}` |
| `[:aybiza, :api, :request]` | `%{latency: latency_ms}` | `%{service: service, endpoint: endpoint, status: status, region: region}` |

## Logging Strategy

### Log Levels

- **debug**: Detailed information for debugging
- **info**: General information about system operation
- **warn**: Potential issues that don't affect operation
- **error**: Errors that affect specific operations
- **critical**: Critical errors that affect system operation

### Structured Logging

All logs are structured as JSON:

```json
{
  "timestamp": "2025-05-16T12:34:56.789Z",
  "level": "info",
  "message": "Call completed successfully",
  "call_id": "CA123456789",
  "tenant_id": "tenant-123",
  "agent_id": "agent-456",
  "duration_ms": 65000,
  "status": "completed",
  "component": "call_manager"
}
```

### Log Aggregation

Logs are aggregated in Loki and can be queried via Grafana:

- **Development**: Logs are output to console and file
- **Testing**: Logs are output to console and file
- **Staging/Production**: Logs are sent to Loki via Promtail

### Sensitive Data Handling

Sensitive data is redacted from logs:

- PII (names, addresses, phone numbers, etc.)
- Authentication credentials
- Financial information
- Health information

## Distributed Tracing

OpenTelemetry is used for distributed tracing:

```elixir
# Example tracing implementation
defmodule Aybiza.Tracing do
  def start_span(name, parent_ctx \\ nil) do
    ctx = OpenTelemetry.Ctx.get_current()
    tracer = OpenTelemetry.tracer(:aybiza_tracer)
    
    span_opts = if parent_ctx do
      [links: [OpenTelemetry.link(parent_ctx)]]
    else
      []
    end
    
    span_ctx = :otel_tracer.start_span(tracer, name, span_opts)
    {ctx, span_ctx}
  end
  
  def end_span({ctx, span_ctx}) do
    :otel_span.end_span(span_ctx)
    OpenTelemetry.Ctx.set_current(ctx)
  end
  
  def add_event(span_ctx, name, attributes \\ %{}) do
    :otel_span.add_event(span_ctx, name, attributes)
  end
  
  def set_attribute(span_ctx, key, value) do
    :otel_span.set_attribute(span_ctx, key, value)
  end
end
```

### Key Spans to Trace

- Call processing end-to-end
- Voice pipeline stages (STT, LLM, TTS)
- Database queries
- External API calls
- WebSocket connections

## Monitoring Dashboards

### Main Dashboard

- Active calls
- Call success/failure rate
- Voice pipeline latency
- System resource usage
- Error rate

### Voice Pipeline Dashboard

- STT latency and confidence
- LLM latency and token usage
- TTS latency and quality
- End-to-end latency
- Component-specific metrics

### Database Dashboard

- Query performance
- Connection pool usage
- Transaction rate
- Database size and growth
- Slow queries

### External API Dashboard

- API call rate
- API latency
- API error rate
- API availability
- Rate limiting status

## Alert Configuration

### Alert Severity Levels

- **P1**: Critical, requires immediate attention (24/7)
- **P2**: High, requires attention within 1 hour
- **P3**: Medium, requires attention within 4 hours
- **P4**: Low, requires attention within 24 hours

### Enhanced Alerts (Hybrid Architecture)

| Alert | Condition | Severity | Response |
|-------|-----------|----------|----------|
| High Error Rate | Error rate > 5% for 5min | P1 | Investigate immediately, potential service degradation |
| Ultra-High Latency | Pipeline latency > 350ms for 5min | P1 | Check edge processing, potential handoff issues |
| Edge Processing Failure | Edge processing rate < 40% for 10min | P2 | Investigate edge capacity, route to cloud |
| Claude Model Latency | Claude 3.7 Sonnet > 300ms or 3.5 Sonnet > 250ms | P2 | Check model performance, consider switching |
| Handoff Failure Rate | Edge-to-cloud handoff failure > 5% | P2 | Investigate hybrid coordination issues |
| Cost Optimization Failure | Cost optimization rate < 50% for 30min | P2 | Review model selection algorithm |
| System Resource Usage | CPU/Memory > 85% for 5min | P2 | Scale up resources or optimize usage |
| Database Connection Pool | Pool usage > 90% for 5min | P2 | Increase pool size or optimize queries |
| Deepgram API Issues | STT/TTS latency > 50ms or confidence < 0.7 | P2 | Check Deepgram service status |
| Edge Region Unavailable | Edge region responses > 100ms for 15min | P3 | Route traffic to alternative edge regions |
| Cost Budget Exceeded | Daily cost > budget threshold | P3 | Review usage patterns and optimization |
| Low Call Volume | Call volume < 10% of normal for 15min | P3 | Investigate potential ingress issues |

### Alert Routing

Alerts are routed based on severity and component:

- P1: On-call engineer (24/7)
- P2: On-call engineer (business hours) or on-call backup (non-business hours)
- P3: Team lead during business hours
- P4: Team queue for next business day

## Health Checks

### Application Health Checks

Endpoints for health checking:

- `/api/health` - Basic application health
- `/api/health/detailed` - Detailed component health
- `/api/health/db` - Database health
- `/api/health/redis` - Redis health
- `/api/health/dependencies` - External dependencies health

Example health check response:

```json
{
  "status": "healthy",
  "version": "1.2.3",
  "components": {
    "database": {"status": "healthy", "latency_ms": 5},
    "redis": {"status": "healthy", "latency_ms": 2},
    "deepgram": {"status": "healthy", "latency_ms": 120},
    "twilio": {"status": "healthy", "latency_ms": 85},
    "aws_bedrock": {"status": "healthy", "latency_ms": 150}
  },
  "uptime_seconds": 86400,
  "environment": "production"
}
```

### Kubernetes Probe Configuration

For Kubernetes deployments:

```yaml
livenessProbe:
  httpGet:
    path: /api/health
    port: 4000
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /api/health/detailed
    port: 4000
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

## Incident Response

### Incident Detection

Incidents are detected through:

- Automated alerts
- User reports
- Proactive monitoring

### Incident Classification

| Severity | Impact | Response Time | Resolution Time |
|----------|--------|---------------|-----------------|
| Critical | Service unavailable | Immediate | <1 hour |
| High | Major functionality impacted | <15 minutes | <4 hours |
| Medium | Limited functionality impacted | <1 hour | <8 hours |
| Low | Minor issues | <4 hours | <24 hours |

### Incident Response Process

1. **Detection**: Alert triggered or issue reported
2. **Triage**: Assess severity and impact
3. **Notification**: Notify appropriate team members
4. **Investigation**: Determine root cause
5. **Mitigation**: Implement immediate fix or workaround
6. **Resolution**: Implement permanent fix
7. **Postmortem**: Document incident and prevention measures

### Postmortem Template

```markdown
# Incident Postmortem: [Incident Title]

## Summary
Brief description of the incident

## Timeline
- **HH:MM** - Initial detection
- **HH:MM** - Triage started
- **HH:MM** - Root cause identified
- **HH:MM** - Mitigation implemented
- **HH:MM** - Resolution confirmed

## Root Cause
Detailed explanation of what caused the incident

## Impact
- Duration of impact
- Number of affected users/calls
- Business impact

## Resolution
How the issue was resolved

## Lessons Learned
What we learned from this incident

## Action Items
- [ ] Action 1 (Owner, Deadline)
- [ ] Action 2 (Owner, Deadline)
- [ ] Action 3 (Owner, Deadline)
```

## Capacity Planning

### Resource Monitoring

Monitor resources to plan for capacity needs:

- CPU usage trends
- Memory usage trends
- Network traffic trends
- Database size growth
- Call volume trends

### Scaling Triggers

Automated scaling based on:

- CPU usage > 70% for 5 minutes
- Memory usage > 70% for 5 minutes
- Active calls > 80% of capacity
- Call queue depth > 10

### Capacity Planning Calendar

Regular capacity planning reviews:

- Weekly: Short-term capacity adjustments
- Monthly: Medium-term capacity planning
- Quarterly: Long-term capacity forecasting

## Production Dashboard

### Real-Time Operations Dashboard

The main production dashboard provides real-time visibility into system health:

```elixir
defmodule Aybiza.Monitoring.Dashboard do
  @moduledoc """
  Production dashboard configuration
  """
  
  def main_dashboard_config do
    %{
      name: "AYBIZA Operations Dashboard",
      refresh_interval: "5s",
      time_range: "last_15_minutes",
      
      panels: [
        # System Health Overview
        %{
          title: "System Health",
          type: :stat,
          grid: {x: 0, y: 0, w: 6, h: 4},
          queries: [
            "up{job='aybiza'}",
            "avg(1 - rate(node_cpu_seconds_total{mode='idle'}[5m]))"
          ],
          thresholds: %{
            green: {0, 0.7},
            yellow: {0.7, 0.85},
            red: {0.85, 1}
          }
        },
        
        # Active Calls
        %{
          title: "Active Calls",
          type: :gauge,
          grid: {x: 6, y: 0, w: 6, h: 4},
          query: "aybiza_active_calls_total",
          max_value: 10000,
          thresholds: %{
            green: {0, 7000},
            yellow: {7000, 8500},
            red: {8500, 10000}
          }
        },
        
        # Voice Pipeline Latency Heatmap
        %{
          title: "Voice Pipeline Latency",
          type: :heatmap,
          grid: {x: 12, y: 0, w: 12, h: 4},
          query: "histogram_quantile(0.95, aybiza_voice_pipeline_latency_bucket)",
          unit: "ms",
          color_scheme: :cool_warm
        },
        
        # Call Success Rate
        %{
          title: "Call Success Rate",
          type: :time_series,
          grid: {x: 0, y: 4, w: 12, h: 6},
          queries: [
            "rate(aybiza_calls_completed_total[5m]) / rate(aybiza_calls_total[5m]) * 100"
          ],
          y_axis: {min: 0, max: 100, unit: "%"},
          thresholds: [
            {value: 95, color: :red, label: "SLA Target"}
          ]
        },
        
        # Error Rate by Component
        %{
          title: "Error Rate by Component",
          type: :bar_chart,
          grid: {x: 12, y: 4, w: 12, h: 6},
          query: "sum by (component) (rate(aybiza_errors_total[5m]))",
          stack: true,
          legend: :right
        },
        
        # Global Call Distribution Map
        %{
          title: "Global Call Distribution",
          type: :geo_map,
          grid: {x: 0, y: 10, w: 16, h: 8},
          query: "sum by (country) (aybiza_calls_by_location)",
          map_type: :world,
          color_mode: :heat
        },
        
        # Component Health Matrix
        %{
          title: "Component Health",
          type: :table,
          grid: {x: 16, y: 10, w: 8, h: 8},
          queries: %{
            component: "aybiza_component_info",
            status: "aybiza_component_status",
            latency: "avg by (component) (aybiza_component_latency)",
            errors: "sum by (component) (rate(aybiza_component_errors[5m]))"
          },
          columns: [:component, :status, :latency, :errors],
          color_coding: true
        }
      ]
    }
  end
  
  def voice_agent_dashboard_config do
    %{
      name: "Voice Agent Performance",
      panels: [
        %{
          title: "Agent Response Time Distribution",
          type: :histogram,
          query: "histogram_quantile(0.99, aybiza_agent_response_time_bucket)"
        },
        
        %{
          title: "Model Performance Comparison",
          type: :multi_line,
          queries: %{
            claude_3_7: "avg(aybiza_llm_latency{model='claude-3-7-sonnet'})",
            claude_3_5: "avg(aybiza_llm_latency{model='claude-3-5-sonnet'})",
            claude_haiku: "avg(aybiza_llm_latency{model='claude-3-haiku'})"
          }
        },
        
        %{
          title: "Deepgram STT/TTS Performance",
          type: :dual_axis,
          left_query: "avg(aybiza_deepgram_stt_latency)",
          right_query: "avg(aybiza_deepgram_tts_latency)",
          left_unit: "ms",
          right_unit: "ms"
        }
      ]
    }
  end
  
  def automation_dashboard_config do
    %{
      name: "Automation Engine",
      panels: [
        %{
          title: "Active Workflows",
          type: :stat,
          query: "aybiza_active_workflows_total"
        },
        
        %{
          title: "Workflow Execution Time",
          type: :box_plot,
          query: "aybiza_workflow_execution_duration"
        },
        
        %{
          title: "Node Execution Success Rate",
          type: :stacked_bar,
          queries: %{
            success: "sum by (node_type) (aybiza_node_executions_success)",
            failure: "sum by (node_type) (aybiza_node_executions_failure)"
          }
        }
      ]
    }
  end
  
  def alerts_dashboard_config do
    %{
      name: "Alerts & Incidents",
      panels: [
        %{
          title: "Active Alerts",
          type: :alert_list,
          datasource: :alertmanager,
          filter: {state: :firing}
        },
        
        %{
          title: "Alert History",
          type: :timeline,
          query: "ALERTS",
          group_by: :alertname
        },
        
        %{
          title: "MTTR by Component",
          type: :bar_chart,
          query: "avg by (component) (aybiza_incident_resolution_time)"
        }
      ]
    }
  end
  
  def cost_dashboard_config do
    %{
      name: "Cost Management",
      panels: [
        %{
          title: "AWS Service Costs",
          type: :pie_chart,
          queries: %{
            bedrock: "aws_cost_by_service{service='bedrock'}",
            ecs: "aws_cost_by_service{service='ecs'}",
            rds: "aws_cost_by_service{service='rds'}",
            other: "aws_cost_by_service{service!~'bedrock|ecs|rds'}"
          }
        },
        
        %{
          title: "Cost per Call",
          type: :trend_line,
          query: "sum(aws_total_cost) / sum(aybiza_calls_total)"
        },
        
        %{
          title: "Deepgram API Usage",
          type: :dual_axis,
          left_query: "sum(deepgram_stt_minutes_used)",
          right_query: "sum(deepgram_tts_characters_used)",
          left_unit: "minutes",
          right_unit: "characters"
        }
      ]
    }
  end
end
```

### Dashboard Implementation

```elixir
defmodule Aybiza.Monitoring.DashboardServer do
  use GenServer
  
  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end
  
  @impl true
  def init(_) do
    # Initialize dashboards
    dashboards = [
      Dashboard.main_dashboard_config(),
      Dashboard.voice_agent_dashboard_config(),
      Dashboard.automation_dashboard_config(),
      Dashboard.alerts_dashboard_config(),
      Dashboard.cost_dashboard_config()
    ]
    
    # Set up auto-refresh
    Process.send_after(self(), :refresh_dashboards, 5_000)
    
    {:ok, %{dashboards: dashboards, metrics: %{}}}
  end
  
  @impl true
  def handle_info(:refresh_dashboards, state) do
    # Fetch latest metrics
    metrics = fetch_all_metrics()
    
    # Broadcast updates to connected clients
    Phoenix.PubSub.broadcast(Aybiza.PubSub, "dashboard:updates", {:metrics_update, metrics})
    
    # Schedule next refresh
    Process.send_after(self(), :refresh_dashboards, 5_000)
    
    {:noreply, %{state | metrics: metrics}}
  end
  
  defp fetch_all_metrics do
    %{
      system_health: fetch_system_health(),
      active_calls: fetch_active_calls(),
      pipeline_latency: fetch_pipeline_latency(),
      error_rates: fetch_error_rates(),
      global_distribution: fetch_global_distribution()
    }
  end
end
```

### Custom Metrics Implementation

```elixir
defmodule Aybiza.Monitoring.Metrics do
  use Prometheus.Metric
  
  def setup do
    # Counter metrics
    Counter.declare(
      name: :aybiza_calls_total,
      help: "Total number of calls processed",
      labels: [:region, :agent_id]
    )
    
    Counter.declare(
      name: :aybiza_errors_total,
      help: "Total number of errors",
      labels: [:component, :error_type]
    )
    
    # Gauge metrics
    Gauge.declare(
      name: :aybiza_active_calls,
      help: "Number of currently active calls",
      labels: [:region]
    )
    
    Gauge.declare(
      name: :aybiza_active_workflows,
      help: "Number of currently active workflows"
    )
    
    # Histogram metrics
    Histogram.declare(
      name: :aybiza_voice_pipeline_latency,
      help: "Voice pipeline processing latency",
      labels: [:component],
      buckets: [10, 25, 50, 100, 250, 500, 1000, 2500, 5000]
    )
    
    Histogram.declare(
      name: :aybiza_llm_latency,
      help: "LLM response latency",
      labels: [:model, :region],
      buckets: [50, 100, 200, 500, 1000, 2000, 5000]
    )
    
    # Summary metrics
    Summary.declare(
      name: :aybiza_call_duration,
      help: "Call duration in seconds",
      labels: [:agent_id]
    )
  end
  
  # Helper functions for metric recording
  def record_call_started(region, agent_id) do
    Counter.inc(name: :aybiza_calls_total, labels: [region, agent_id])
    Gauge.inc(name: :aybiza_active_calls, labels: [region])
  end
  
  def record_call_ended(region, agent_id, duration) do
    Gauge.dec(name: :aybiza_active_calls, labels: [region])
    Summary.observe(
      [name: :aybiza_call_duration, labels: [agent_id]], 
      duration
    )
  end
  
  def record_pipeline_latency(component, latency) do
    Histogram.observe(
      [name: :aybiza_voice_pipeline_latency, labels: [component]], 
      latency
    )
  end
  
  def record_error(component, error_type) do
    Counter.inc(
      name: :aybiza_errors_total, 
      labels: [component, error_type]
    )
  end
end
```

### Mobile Dashboard Support

```elixir
defmodule AybizaWeb.MobileDashboardController do
  use AybizaWeb, :controller
  
  def index(conn, _params) do
    # Simplified mobile dashboard
    metrics = %{
      active_calls: get_metric(:active_calls),
      success_rate: get_metric(:success_rate),
      avg_latency: get_metric(:avg_latency),
      error_rate: get_metric(:error_rate)
    }
    
    conn
    |> put_resp_content_type("application/json")
    |> json(%{metrics: metrics, timestamp: DateTime.utc_now()})
  end
  
  def websocket(conn, _params) do
    # WebSocket connection for real-time updates
    {:ok, _} = MobileDashboardSocket.start_link(conn)
    conn
  end
end
```

## Implementation Checklist

- [ ] Set up Prometheus and Grafana
- [ ] Configure Loki for log aggregation
- [ ] Implement OpenTelemetry tracing
- [ ] Set up AlertManager and alert rules
- [ ] Create dashboards for all key metrics
- [ ] Configure health check endpoints
- [ ] Set up PagerDuty integration
- [ ] Document incident response procedures
- [ ] Train team on monitoring tools and procedures
- [ ] Conduct regular capacity planning reviews
- [ ] Deploy production dashboard with real-time updates
- [ ] Implement custom metrics collection
- [ ] Set up mobile dashboard support
- [ ] Configure dashboard access controls
- [ ] Create dashboard templates for common views