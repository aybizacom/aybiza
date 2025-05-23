# Real-time Agent Analytics Dashboard Specification
> Specification #005 - v1.0

## High-Level Objective

- Create a comprehensive real-time analytics dashboard using Phoenix LiveView that provides instant visibility into voice agent performance, conversation quality, learning patterns, and business metrics

## Mid-Level Objectives

- Build real-time dashboard with sub-second updates via WebSocket
- Implement agent performance tracking and quality metrics
- Create conversation analytics with sentiment and outcome tracking
- Visualize learning system effectiveness and improvements
- Design cost analytics and optimization insights
- Ensure dashboard scales to monitor 10,000+ concurrent agents

## Implementation Notes

- Use Phoenix LiveView for real-time updates without JavaScript
- Implement Telemetry.Metrics for data collection
- Use TimescaleDB for time-series analytics data
- Leverage Phoenix PubSub for distributed updates
- Create responsive design for desktop and mobile
- Follow AYBIZA's established UI/UX patterns
- Ensure all metrics respect data privacy requirements

## Context

### Beginning Context
- `apps/analytics_web/lib/analytics_web/live/dashboard_live.ex` (new)
- `apps/analytics/lib/analytics/metrics_collector.ex` (new)
- `apps/analytics/lib/analytics/aggregator.ex` (new)

### Ending Context
- `apps/analytics_web/lib/analytics_web/live/dashboard_live.ex` (created)
- `apps/analytics_web/lib/analytics_web/live/components/` (created with components)
- `apps/analytics/lib/analytics/metrics_collector.ex` (created)
- `apps/analytics/lib/analytics/aggregator.ex` (created)
- `apps/analytics/lib/analytics/time_series.ex` (created)
- `apps/analytics/lib/analytics/alerting.ex` (created)
- `apps/analytics_web/assets/css/dashboard.css` (created)
- `apps/analytics/test/analytics/metrics_collector_test.exs` (created)

## Low-Level Tasks
> Ordered from start to finish

1. Create metrics collector infrastructure
```claude
CREATE apps/analytics/lib/analytics/metrics_collector.ex:
    IMPLEMENT GenServer for metrics collection:
        Subscribe to telemetry events from all components
        Buffer metrics for batch processing
        Implement sampling for high-volume metrics
        Add metric validation and sanitization
        Create metric routing to appropriate stores
        Implement backpressure handling
```

2. Build time-series data management
```claude
CREATE apps/analytics/lib/analytics/time_series.ex:
    IMPLEMENT TimescaleDB integration:
        Design hypertables for different metric types
        Create continuous aggregates for performance
        Implement data retention policies
        Add compression for historical data
        Create efficient query patterns
        Build data export capabilities
```

3. Create metrics aggregator
```claude
CREATE apps/analytics/lib/analytics/aggregator.ex:
    IMPLEMENT real-time aggregation:
        Calculate moving averages and percentiles
        Aggregate by agent, customer, region
        Implement sliding window calculations
        Create derived metrics (efficiency, quality)
        Build anomaly detection algorithms
        Cache frequently accessed aggregates
```

4. Build main dashboard LiveView
```claude
CREATE apps/analytics_web/lib/analytics_web/live/dashboard_live.ex:
    IMPLEMENT Phoenix LiveView dashboard:
        Create multi-tab interface structure
        Implement real-time data subscriptions
        Add user preference persistence
        Create responsive grid layout
        Implement dashboard customization
        Add export and sharing capabilities
```

5. Create agent performance components
```claude
CREATE apps/analytics_web/lib/analytics_web/live/components/agent_metrics.ex:
    IMPLEMENT agent tracking components:
        Active agents counter with regional breakdown
        Call volume and duration charts
        Success rate and escalation metrics
        Agent utilization heatmap
        Top performing agents leaderboard
        Agent learning progress tracker
```

6. Build conversation analytics components
```claude
CREATE apps/analytics_web/lib/analytics_web/live/components/conversation_metrics.ex:
    IMPLEMENT conversation analysis:
        Real-time sentiment tracking gauge
        Conversation flow visualization
        Topic distribution charts
        Resolution time histograms
        Customer satisfaction scores
        Intervention rate tracking
```

7. Create cost analytics components
```claude
CREATE apps/analytics_web/lib/analytics_web/live/components/cost_metrics.ex:
    IMPLEMENT cost tracking:
        Real-time cost per call calculator
        Model usage breakdown (Opus vs Sonnet)
        Token consumption charts
        Cost optimization recommendations
        Budget tracking and alerts
        ROI calculations
```

8. Build learning system dashboard
```claude
CREATE apps/analytics_web/lib/analytics_web/live/components/learning_metrics.ex:
    IMPLEMENT learning analytics:
        Pattern discovery visualization
        Improvement rate over time
        Error reduction tracking
        A/B test performance comparison
        Knowledge graph growth
        Success pattern replication metrics
```

9. Implement alerting system
```claude
CREATE apps/analytics/lib/analytics/alerting.ex:
    IMPLEMENT intelligent alerting:
        Define alert thresholds and rules
        Create alert routing (email, SMS, Slack)
        Implement alert suppression logic
        Add predictive alerting
        Create alert analytics
        Build alert acknowledgment system
```

10. Create comprehensive test suite
```claude
CREATE apps/analytics/test/analytics/metrics_collector_test.exs:
    IMPLEMENT analytics tests:
        Test high-volume metric ingestion
        Verify aggregation accuracy
        Test real-time update latency
        Validate alert triggering
        Test dashboard responsiveness
        Verify data privacy compliance
```

## Dashboard UI Components

```elixir
# Main dashboard structure
defmodule AnalyticsWeb.DashboardLive do
  use AnalyticsWeb, :live_view

  @impl true
  def render(assigns) do
    ~H"""
    <div class="dashboard-container">
      <.header>
        <h1>AYBIZA Voice Agent Analytics</h1>
        <.date_range_picker />
        <.refresh_controls />
      </.header>

      <.tabs>
        <.tab name="overview" label="Overview" active={@active_tab == "overview"}>
          <.metric_grid>
            <.metric_card title="Active Agents" value={@active_agents} trend="+12%" />
            <.metric_card title="Calls Today" value={@calls_today} trend="+5%" />
            <.metric_card title="Avg Resolution Time" value={@avg_resolution} trend="-8%" />
            <.metric_card title="Success Rate" value={@success_rate} trend="+2%" />
          </.metric_grid>
          
          <.chart_grid>
            <.time_series_chart data={@call_volume_data} title="Call Volume" />
            <.sentiment_gauge value={@avg_sentiment} title="Customer Sentiment" />
            <.geographic_map data={@regional_data} title="Global Activity" />
          </.chart_grid>
        </.tab>

        <.tab name="agents" label="Agent Performance">
          <.agent_performance_dashboard agents={@agents} />
        </.tab>

        <.tab name="learning" label="Learning & Improvement">
          <.learning_dashboard patterns={@patterns} improvements={@improvements} />
        </.tab>

        <.tab name="costs" label="Cost Analytics">
          <.cost_dashboard costs={@costs} optimizations={@optimizations} />
        </.tab>
      </.tabs>
    </div>
    """
  end
end

# Real-time metric updates
def handle_info({:metric_update, metric}, socket) do
  {:noreply, update_metric(socket, metric)}
end
```

## Visualization Examples

```yaml
key_visualizations:
  agent_activity_heatmap:
    type: heatmap
    dimensions: [time_of_day, day_of_week]
    metric: agent_utilization
    color_scale: green_to_red

  conversation_flow_sankey:
    type: sankey_diagram
    nodes: [greeting, intent_detection, resolution, farewell]
    flows: conversation_transitions
    
  learning_improvement_timeline:
    type: timeline_chart
    x_axis: date
    y_axis: improvement_percentage
    series: [error_rate, success_rate, efficiency]

  cost_breakdown_treemap:
    type: treemap
    hierarchy: [region, agent_type, model]
    size: cost
    color: efficiency_score
```

## Testing Criteria

- Dashboard must update within 100ms of metric emission
- Must handle 100,000+ metrics per second
- All visualizations must render in <500ms
- Dashboard must remain responsive with 50+ concurrent users
- Data accuracy must be 99.9%+
- Must work on mobile devices
- All sensitive data must be properly masked

## Success Metrics

- Real-time update latency: <100ms
- Dashboard load time: <2 seconds
- Metric ingestion rate: 100K+/second
- User satisfaction: >90%
- Data accuracy: 99.9%+
- Mobile responsiveness: 100%