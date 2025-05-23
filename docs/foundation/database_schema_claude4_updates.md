# Database Schema Updates for Claude 4 Implementation
> Schema Update Document v1.0 - Based on Claude 4 Architecture Documents

## Overview

This document outlines the database schema updates required to support the Claude 4 implementation based on the new architecture documents. The existing schema is comprehensive but needs enhancements for the learning system, advanced memory management, and Claude 4-specific features.

## New Tables Required

### Schema: learning

#### Table: conversation_patterns
```sql
CREATE TABLE conversation_patterns (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pattern_id VARCHAR(100) UNIQUE NOT NULL,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  pattern_type VARCHAR(50) NOT NULL CHECK (pattern_type IN ('success', 'failure', 'objection_handling', 'resolution', 'escalation')),
  category VARCHAR(100) NOT NULL,
  trigger_conditions JSONB NOT NULL, -- Pattern matching rules
  successful_responses JSONB DEFAULT '[]', -- Array of response templates
  success_rate DECIMAL(5,2) DEFAULT 0.00,
  confidence_score DECIMAL(3,2) DEFAULT 0.50,
  usage_count INTEGER DEFAULT 0,
  last_used_at TIMESTAMP(6) WITH TIME ZONE,
  discovered_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  metadata JSONB DEFAULT '{}',
  is_active BOOLEAN DEFAULT TRUE,
  version INTEGER DEFAULT 1
);

CREATE INDEX idx_patterns_tenant_type ON conversation_patterns(tenant_id, pattern_type);
CREATE INDEX idx_patterns_success_rate ON conversation_patterns(success_rate DESC);
```

#### Table: learning_artifacts
```sql
CREATE TABLE learning_artifacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  artifact_id VARCHAR(100) UNIQUE NOT NULL,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  artifact_type VARCHAR(50) NOT NULL CHECK (artifact_type IN ('behavioral_improvement', 'tool_optimization', 'language_enhancement', 'error_prevention')),
  description TEXT NOT NULL,
  current_performance JSONB NOT NULL, -- Current metrics
  improvement_strategy JSONB NOT NULL, -- Proposed changes
  expected_impact JSONB NOT NULL, -- Predicted improvements
  actual_impact JSONB DEFAULT '{}', -- Measured after deployment
  status VARCHAR(50) DEFAULT 'proposed' CHECK (status IN ('proposed', 'testing', 'deployed', 'rejected', 'archived')),
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  deployed_at TIMESTAMP(6) WITH TIME ZONE,
  created_by VARCHAR(50) DEFAULT 'learning_engine',
  confidence_level DECIMAL(3,2) DEFAULT 0.50,
  rollout_percentage DECIMAL(5,2) DEFAULT 0.00,
  metadata JSONB DEFAULT '{}'
);

CREATE INDEX idx_artifacts_status ON learning_artifacts(status, tenant_id);
```

#### Table: improvement_experiments
```sql
CREATE TABLE improvement_experiments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  experiment_id VARCHAR(100) UNIQUE NOT NULL,
  learning_artifact_id UUID NOT NULL REFERENCES learning_artifacts(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  hypothesis TEXT NOT NULL,
  control_variant JSONB NOT NULL, -- Current behavior
  test_variants JSONB NOT NULL DEFAULT '[]', -- Array of test behaviors
  allocation_strategy VARCHAR(50) DEFAULT 'random' CHECK (allocation_strategy IN ('random', 'weighted', 'targeted')),
  target_sample_size INTEGER NOT NULL,
  current_sample_size INTEGER DEFAULT 0,
  metrics JSONB NOT NULL, -- Metrics to track
  results JSONB DEFAULT '{}', -- Accumulated results
  statistical_significance DECIMAL(5,4), -- p-value
  status VARCHAR(50) DEFAULT 'planning' CHECK (status IN ('planning', 'running', 'completed', 'stopped')),
  started_at TIMESTAMP(6) WITH TIME ZONE,
  completed_at TIMESTAMP(6) WITH TIME ZONE,
  winner_variant VARCHAR(50),
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_experiments_status ON improvement_experiments(status, tenant_id);
```

#### Table: error_catalog
```sql
CREATE TABLE error_catalog (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  error_fingerprint VARCHAR(64) UNIQUE NOT NULL, -- Hash of error characteristics
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  error_type VARCHAR(100) NOT NULL,
  error_category VARCHAR(50) NOT NULL CHECK (error_category IN ('tool_failure', 'comprehension', 'execution', 'timeout', 'permission', 'integration')),
  occurrence_count INTEGER DEFAULT 1,
  first_seen_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  last_seen_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  error_context JSONB NOT NULL, -- Context when error occurs
  prevention_strategy JSONB DEFAULT '{}', -- How to prevent
  resolution_strategy JSONB DEFAULT '{}', -- How to resolve
  is_resolved BOOLEAN DEFAULT FALSE,
  resolved_at TIMESTAMP(6) WITH TIME ZONE,
  impact_severity VARCHAR(20) DEFAULT 'medium' CHECK (impact_severity IN ('low', 'medium', 'high', 'critical')),
  affected_features JSONB DEFAULT '[]'
);

CREATE INDEX idx_errors_fingerprint ON error_catalog(error_fingerprint);
CREATE INDEX idx_errors_unresolved ON error_catalog(is_resolved, tenant_id) WHERE is_resolved = FALSE;
```

#### Table: collective_knowledge
```sql
CREATE TABLE collective_knowledge (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  knowledge_id VARCHAR(100) UNIQUE NOT NULL,
  knowledge_type VARCHAR(50) NOT NULL CHECK (knowledge_type IN ('best_practice', 'common_solution', 'industry_pattern', 'optimization')),
  category VARCHAR(100) NOT NULL,
  title VARCHAR(255) NOT NULL,
  description TEXT NOT NULL,
  applicable_contexts JSONB NOT NULL, -- When to apply this knowledge
  implementation JSONB NOT NULL, -- How to implement
  success_metrics JSONB DEFAULT '{}', -- How to measure success
  source_tenant_ids UUID[] DEFAULT '{}', -- Tenants that contributed
  adoption_count INTEGER DEFAULT 0,
  average_improvement DECIMAL(5,2), -- Percentage improvement
  confidence_score DECIMAL(3,2) DEFAULT 0.50,
  is_verified BOOLEAN DEFAULT FALSE,
  verified_by UUID REFERENCES users(id),
  verified_at TIMESTAMP(6) WITH TIME ZONE,
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_knowledge_type_category ON collective_knowledge(knowledge_type, category);
CREATE INDEX idx_knowledge_confidence ON collective_knowledge(confidence_score DESC);
```

### Schema: memory_enhanced

#### Table: memory_synthesis_results
```sql
CREATE TABLE memory_synthesis_results (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  synthesis_id VARCHAR(100) UNIQUE NOT NULL,
  synthesis_type VARCHAR(50) NOT NULL CHECK (synthesis_type IN ('daily', 'weekly', 'on_demand', 'real_time')),
  time_range_start TIMESTAMP(6) WITH TIME ZONE NOT NULL,
  time_range_end TIMESTAMP(6) WITH TIME ZONE NOT NULL,
  conversations_analyzed INTEGER NOT NULL,
  patterns_extracted JSONB DEFAULT '[]',
  knowledge_graph_updates JSONB DEFAULT '{}',
  relationship_updates JSONB DEFAULT '{}',
  key_insights JSONB DEFAULT '[]',
  recommendations JSONB DEFAULT '[]',
  synthesis_duration_ms INTEGER,
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  metadata JSONB DEFAULT '{}'
);

CREATE INDEX idx_synthesis_tenant_time ON memory_synthesis_results(tenant_id, time_range_end DESC);
```

#### Table: memory_versions
```sql
CREATE TABLE memory_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  memory_id UUID NOT NULL, -- References agent_memories
  version INTEGER NOT NULL,
  content TEXT NOT NULL,
  embedding VECTOR(1536),
  metadata JSONB DEFAULT '{}',
  change_type VARCHAR(50) NOT NULL CHECK (change_type IN ('create', 'update', 'merge', 'expire')),
  change_reason TEXT,
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  created_by VARCHAR(50) DEFAULT 'system', -- 'system', 'user', 'learning_engine'
  UNIQUE(memory_id, version)
);

CREATE INDEX idx_memory_versions ON memory_versions(memory_id, version DESC);
```

### Schema: claude4_features

#### Table: thinking_sessions
```sql
CREATE TABLE thinking_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id VARCHAR(100) UNIQUE NOT NULL,
  call_id UUID REFERENCES calls(id),
  agent_id UUID NOT NULL REFERENCES agents(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  thinking_mode VARCHAR(50) NOT NULL CHECK (thinking_mode IN ('standard', 'extended', 'deep')),
  thinking_budget INTEGER NOT NULL, -- Token budget (1024-64000)
  tokens_used INTEGER,
  problem_description TEXT NOT NULL,
  thinking_process JSONB DEFAULT '{}', -- Structured thinking steps
  solution JSONB DEFAULT '{}', -- Final solution
  tools_considered JSONB DEFAULT '[]', -- Tools evaluated during thinking
  tools_used JSONB DEFAULT '[]', -- Tools actually used
  confidence_score DECIMAL(3,2),
  duration_ms INTEGER,
  status VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'completed', 'interrupted', 'failed')),
  started_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMP(6) WITH TIME ZONE,
  metadata JSONB DEFAULT '{}'
);

CREATE INDEX idx_thinking_sessions_call ON thinking_sessions(call_id);
CREATE INDEX idx_thinking_sessions_status ON thinking_sessions(status, tenant_id);
```

#### Table: prompt_cache
```sql
CREATE TABLE prompt_cache (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cache_key VARCHAR(64) UNIQUE NOT NULL, -- SHA-256 hash
  prompt_type VARCHAR(50) NOT NULL CHECK (prompt_type IN ('system', 'template', 'dynamic')),
  prompt_content TEXT NOT NULL,
  tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE, -- NULL for system prompts
  agent_id UUID REFERENCES agents(id) ON DELETE CASCADE, -- NULL for tenant-wide
  token_count INTEGER NOT NULL,
  usage_count INTEGER DEFAULT 0,
  last_used_at TIMESTAMP(6) WITH TIME ZONE,
  cache_hit_rate DECIMAL(5,2) DEFAULT 0.00,
  estimated_savings_usd DECIMAL(10,4) DEFAULT 0.00,
  expires_at TIMESTAMP(6) WITH TIME ZONE,
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  metadata JSONB DEFAULT '{}'
);

CREATE INDEX idx_prompt_cache_key ON prompt_cache(cache_key);
CREATE INDEX idx_prompt_cache_usage ON prompt_cache(usage_count DESC);
```

#### Table: parallel_execution_plans
```sql
CREATE TABLE parallel_execution_plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id VARCHAR(100) UNIQUE NOT NULL,
  call_id UUID REFERENCES calls(id),
  agent_id UUID NOT NULL REFERENCES agents(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  requested_tools JSONB NOT NULL, -- Original tool requests
  dependency_graph JSONB NOT NULL, -- DAG of dependencies
  execution_groups JSONB NOT NULL, -- Parallel execution groups
  estimated_duration_ms INTEGER,
  actual_duration_ms INTEGER,
  parallelism_degree INTEGER, -- Max tools in parallel
  execution_strategy VARCHAR(50) DEFAULT 'optimal' CHECK (execution_strategy IN ('optimal', 'conservative', 'aggressive')),
  status VARCHAR(50) DEFAULT 'planned' CHECK (status IN ('planned', 'executing', 'completed', 'failed')),
  results JSONB DEFAULT '{}',
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  executed_at TIMESTAMP(6) WITH TIME ZONE,
  completed_at TIMESTAMP(6) WITH TIME ZONE
);

CREATE INDEX idx_parallel_plans_call ON parallel_execution_plans(call_id);
```

### Schema: tool_registry_enhanced

#### Table: tool_credentials
```sql
CREATE TABLE tool_credentials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  credential_id VARCHAR(100) UNIQUE NOT NULL,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  tool_id UUID NOT NULL REFERENCES tools(id),
  credential_name VARCHAR(255) NOT NULL,
  credential_type VARCHAR(50) NOT NULL CHECK (credential_type IN ('api_key', 'oauth2', 'basic_auth', 'custom')),
  encrypted_credentials JSONB NOT NULL, -- KMS encrypted
  kms_key_id VARCHAR(255) NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  last_rotated_at TIMESTAMP(6) WITH TIME ZONE,
  rotation_period_days INTEGER, -- NULL means no rotation
  next_rotation_at TIMESTAMP(6) WITH TIME ZONE,
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  created_by UUID NOT NULL REFERENCES users(id),
  metadata JSONB DEFAULT '{}'
);

CREATE INDEX idx_tool_credentials_tenant ON tool_credentials(tenant_id, tool_id);
CREATE INDEX idx_tool_credentials_rotation ON tool_credentials(next_rotation_at) WHERE next_rotation_at IS NOT NULL;
```

#### Table: webhook_configurations
```sql
CREATE TABLE webhook_configurations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  webhook_id VARCHAR(100) UNIQUE NOT NULL,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  endpoint_url VARCHAR(1000) NOT NULL,
  http_method VARCHAR(10) NOT NULL CHECK (http_method IN ('GET', 'POST', 'PUT', 'DELETE', 'PATCH')),
  headers JSONB DEFAULT '{}', -- Custom headers
  authentication JSONB DEFAULT '{}', -- Auth config
  retry_config JSONB DEFAULT '{
    "max_attempts": 3,
    "backoff_strategy": "exponential",
    "initial_delay_ms": 1000
  }',
  timeout_ms INTEGER DEFAULT 30000,
  signature_config JSONB DEFAULT '{}', -- For webhook signature verification
  is_active BOOLEAN DEFAULT TRUE,
  last_triggered_at TIMESTAMP(6) WITH TIME ZONE,
  success_count INTEGER DEFAULT 0,
  failure_count INTEGER DEFAULT 0,
  average_response_time_ms INTEGER,
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhooks_tenant ON webhook_configurations(tenant_id, is_active);
```

### Schema: analytics_enhanced

#### Table: dashboard_configurations
```sql
CREATE TABLE dashboard_configurations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  config_id VARCHAR(100) UNIQUE NOT NULL,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id),
  dashboard_name VARCHAR(255) NOT NULL,
  layout_config JSONB NOT NULL, -- Grid layout, widget positions
  widgets JSONB NOT NULL, -- Widget configurations
  refresh_interval_seconds INTEGER DEFAULT 30,
  time_range VARCHAR(50) DEFAULT '24h',
  filters JSONB DEFAULT '{}',
  is_default BOOLEAN DEFAULT FALSE,
  is_shared BOOLEAN DEFAULT FALSE,
  shared_with_roles UUID[] DEFAULT '{}',
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_dashboard_configs_user ON dashboard_configurations(user_id, tenant_id);
```

#### Table: alert_rules
```sql
CREATE TABLE alert_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  rule_id VARCHAR(100) UNIQUE NOT NULL,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  rule_name VARCHAR(255) NOT NULL,
  rule_type VARCHAR(50) NOT NULL CHECK (rule_type IN ('threshold', 'anomaly', 'trend', 'composite')),
  metric_name VARCHAR(255) NOT NULL,
  condition JSONB NOT NULL, -- Rule conditions
  threshold_value DECIMAL(20,4),
  time_window_minutes INTEGER DEFAULT 5,
  severity VARCHAR(20) NOT NULL CHECK (severity IN ('info', 'warning', 'error', 'critical')),
  notification_channels JSONB NOT NULL DEFAULT '[]', -- Email, SMS, Slack, etc.
  cooldown_minutes INTEGER DEFAULT 60,
  is_active BOOLEAN DEFAULT TRUE,
  last_triggered_at TIMESTAMP(6) WITH TIME ZONE,
  trigger_count INTEGER DEFAULT 0,
  created_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW(),
  created_by UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_alert_rules_active ON alert_rules(tenant_id, is_active);
```

## Updates to Existing Tables

### Table: agent_memories (Enhanced)
```sql
-- Add new columns to existing agent_memories table
ALTER TABLE agent_memories 
ADD COLUMN memory_version INTEGER DEFAULT 1,
ADD COLUMN synthesis_eligible BOOLEAN DEFAULT TRUE,
ADD COLUMN related_memory_ids UUID[] DEFAULT '{}',
ADD COLUMN decay_factor DECIMAL(3,2) DEFAULT 0.95, -- For temporal decay
ADD COLUMN reinforcement_count INTEGER DEFAULT 0, -- Times reinforced
ADD COLUMN last_reinforced_at TIMESTAMP(6) WITH TIME ZONE;

-- Add indexes for performance
CREATE INDEX idx_memories_synthesis ON agent_memories(synthesis_eligible, tenant_id) WHERE synthesis_eligible = TRUE;
CREATE INDEX idx_memories_importance ON agent_memories(importance_score DESC, tenant_id);
```

### Table: tools (Enhanced)
```sql
-- Add new columns for parallel execution support
ALTER TABLE tools
ADD COLUMN parallel_safe BOOLEAN DEFAULT TRUE,
ADD COLUMN max_parallel_instances INTEGER DEFAULT 10,
ADD COLUMN rate_limit_per_minute INTEGER,
ADD COLUMN estimated_duration_ms INTEGER,
ADD COLUMN dependency_tools UUID[] DEFAULT '{}', -- Tools that must run before this
ADD COLUMN webhook_config_id UUID REFERENCES webhook_configurations(id);
```

### Table: agents (Enhanced)
```sql
-- Update model_config to support Claude 4 features
ALTER TABLE agents
ADD COLUMN learning_enabled BOOLEAN DEFAULT TRUE,
ADD COLUMN memory_config JSONB DEFAULT '{
  "short_term_ttl_seconds": 300,
  "long_term_enabled": true,
  "synthesis_frequency": "daily",
  "max_memories_per_type": 1000
}',
ADD COLUMN thinking_config JSONB DEFAULT '{
  "enabled": true,
  "default_mode": "standard",
  "max_budget": 4096,
  "auto_escalate": true
}';
```

## Redis Schema for Short-term Memory

```lua
-- Redis key patterns for short-term memory
-- Pattern: memory:short:{tenant_id}:{customer_id}:{timestamp}
-- TTL: 300 seconds (5 minutes)

-- Conversation context
-- Key: context:{call_id}
-- Type: Hash
-- Fields: customer_id, agent_id, start_time, current_state, turn_count

-- Active tool executions
-- Key: tools:active:{call_id}
-- Type: Sorted Set
-- Score: timestamp, Member: tool_execution_id

-- Recent patterns
-- Key: patterns:recent:{agent_id}
-- Type: List
-- Values: pattern_ids (last 20 patterns used)
```

## DynamoDB Schema Updates

```yaml
# Enhanced agent_sessions table structure
Table: agent_sessions_enhanced
  PartitionKey: tenant_id
  SortKey: session_id
  Attributes:
    - agent_id
    - customer_id
    - call_id
    - session_state (active/completed/failed)
    - context_snapshot (compressed)
    - active_tools (list)
    - thinking_budget_used
    - memory_references (list of memory IDs accessed)
    - learning_opportunities (patterns to extract)
    - created_at
    - updated_at
    - ttl (for automatic cleanup)
  
  GSI1:
    PartitionKey: agent_id
    SortKey: created_at
  
  GSI2:
    PartitionKey: customer_id
    SortKey: created_at
```

## Migration Steps

1. **Create new schemas**
   ```sql
   CREATE SCHEMA IF NOT EXISTS learning;
   CREATE SCHEMA IF NOT EXISTS memory_enhanced;
   CREATE SCHEMA IF NOT EXISTS claude4_features;
   CREATE SCHEMA IF NOT EXISTS tool_registry_enhanced;
   CREATE SCHEMA IF NOT EXISTS analytics_enhanced;
   ```

2. **Run table creation scripts** in order (foreign key dependencies)

3. **Update existing tables** with ALTER statements

4. **Create TimescaleDB hypertables** for time-series data:
   ```sql
   SELECT create_hypertable('thinking_sessions', 'started_at');
   SELECT create_hypertable('improvement_experiments', 'created_at');
   ```

5. **Set up Citus distribution** for new tables:
   ```sql
   SELECT create_distributed_table('conversation_patterns', 'tenant_id');
   SELECT create_distributed_table('learning_artifacts', 'tenant_id');
   SELECT create_distributed_table('thinking_sessions', 'tenant_id');
   ```

## Summary

These schema updates provide:
- ✅ Complete self-learning system with pattern recognition and A/B testing
- ✅ Enhanced memory architecture with synthesis and versioning
- ✅ Claude 4 specific features (thinking sessions, prompt caching)
- ✅ Advanced tool registry with credentials and webhooks
- ✅ Real-time analytics dashboard configuration
- ✅ Parallel execution planning and tracking
- ✅ Error prevention and collective intelligence

The schema is designed to scale to billions of operations while maintaining the flexibility to integrate with future AWS Bedrock features as they become available.