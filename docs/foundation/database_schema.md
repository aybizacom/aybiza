# AYBIZA AI Voice Agent Platform - Database Schema

## PostgreSQL 16.9+ with TimescaleDB 2.20.0+ Database Design

### Database Configuration
- **PostgreSQL Version**: 16.9+ with enhanced performance features
- **TimescaleDB Version**: 2.20.0+ with continuous aggregates and improved compression
- **Encoding**: UTF8 with ICU collation support
- **Connection Pooling**: PgBouncer with optimized pool settings
- **Replication**: Aurora Global Database with cross-region read replicas
- **Backup**: Point-in-time recovery with 30-day retention

### Schema: agent_manager

#### Table: organizations
- `id`: UUID (PK)
- `name`: VARCHAR(255) NOT NULL
- `slug`: VARCHAR(100) UNIQUE NOT NULL
- `settings`: JSONB DEFAULT '{}'
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended', 'pending'))
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)
- `version`: INTEGER DEFAULT 1
- `business_type`: VARCHAR(100)
- `registration_number`: VARCHAR(100)
- `tax_id`: VARCHAR(100)
- `address_line1`: VARCHAR(255)
- `address_line2`: VARCHAR(255)
- `city`: VARCHAR(100)
- `state`: VARCHAR(100)
- `postal_code`: VARCHAR(20)
- `country`: VARCHAR(100)
- `phone_number`: VARCHAR(20)
- `website`: VARCHAR(255)
- `industry`: VARCHAR(100)
- `compliance_status`: VARCHAR(50) DEFAULT 'pending' CHECK (compliance_status IN ('pending', 'verified', 'rejected', 'expired'))
- `compliance_verified_at`: TIMESTAMP(6) WITH TIME ZONE
- `compliance_documents`: JSONB DEFAULT '{}'
- `edge_regions`: JSONB DEFAULT '["us-east-1", "eu-west-1"]' -- Cloudflare edge optimization regions
- `data_residency_region`: VARCHAR(50) DEFAULT 'us-east-1' -- GDPR compliance
- `hybrid_config`: JSONB DEFAULT '{}' -- Cloudflare+AWS configuration

#### Table: tenants
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `name`: VARCHAR(255) NOT NULL
- `slug`: VARCHAR(100) UNIQUE NOT NULL
- `subdomain`: VARCHAR(255) UNIQUE NOT NULL
- `settings`: JSONB DEFAULT '{}'
- `plan_tier`: VARCHAR(50) DEFAULT 'standard' CHECK (plan_tier IN ('starter', 'standard', 'premium', 'enterprise'))
- `max_agents`: INTEGER DEFAULT 5
- `max_concurrent_calls`: INTEGER DEFAULT 10
- `max_phone_numbers`: INTEGER DEFAULT 1
- `features`: JSONB DEFAULT '{}' -- Feature flags and limits
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)
- `version`: INTEGER DEFAULT 1
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended', 'trial'))
- `trial_ends_at`: TIMESTAMP(6) WITH TIME ZONE
- `billing_cycle`: VARCHAR(20) DEFAULT 'monthly' CHECK (billing_cycle IN ('monthly', 'yearly'))

#### Table: roles
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `name`: VARCHAR(100) NOT NULL
- `description`: TEXT
- `permissions`: JSONB NOT NULL DEFAULT '{}'
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)
- `version`: INTEGER DEFAULT 1
- `is_system_role`: BOOLEAN DEFAULT FALSE
- `scope`: VARCHAR(50) DEFAULT 'tenant' CHECK (scope IN ('organization', 'tenant'))
- UNIQUE (organization_id, name)

#### Table: users
- `id`: UUID (PK)
- `email`: VARCHAR(255) UNIQUE NOT NULL
- `encrypted_password`: VARCHAR(255) NOT NULL
- `full_name`: VARCHAR(255) NOT NULL
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `tenant_id`: UUID REFERENCES tenants(id) ON DELETE SET NULL
- `role_id`: UUID NOT NULL REFERENCES roles(id)
- `settings`: JSONB DEFAULT '{}'
- `is_active`: BOOLEAN DEFAULT TRUE
- `last_login`: TIMESTAMP(6) WITH TIME ZONE
- `login_attempts`: INTEGER DEFAULT 0
- `locked_until`: TIMESTAMP(6) WITH TIME ZONE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)
- `version`: INTEGER DEFAULT 1
- `position`: VARCHAR(100)
- `department`: VARCHAR(100)
- `phone_number`: VARCHAR(20)
- `mfa_enabled`: BOOLEAN DEFAULT FALSE
- `mfa_secret`: TEXT -- Encrypted field
- `recovery_codes`: JSONB -- Encrypted field
- `password_changed_at`: TIMESTAMP(6) WITH TIME ZONE DEFAULT NOW()
- `must_change_password`: BOOLEAN DEFAULT FALSE

#### Table: agents
- `id`: UUID (PK)
- `name`: VARCHAR(255) NOT NULL
- `description`: TEXT
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)
- `system_prompt`: TEXT NOT NULL
- `voice_config`: JSONB DEFAULT '{
  "model": "aura-asteria-en",
  "style": "calm_informative",
  "natural_speech": true,
  "emotion_control": "moderate",
  "speed": 1.0,
  "pitch": 0.0
}'
- `language_config`: JSONB DEFAULT '{
  "primary": "en-US",
  "auto_detect": true,
  "supported": ["en-US"]
}'
- `model_config`: JSONB DEFAULT '{
  "default": "claude-3-5-sonnet-v2",
  "fast": "claude-3-5-haiku", 
  "complex": "claude-3-7-sonnet",
  "selection_strategy": "auto"
}'
- `settings`: JSONB DEFAULT '{}'
- `status`: VARCHAR(50) DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'inactive', 'archived'))
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `version`: INTEGER DEFAULT 1
- `latency_optimization_level`: VARCHAR(50) DEFAULT 'balanced' CHECK (latency_optimization_level IN ('ultra_low', 'low', 'balanced', 'quality'))
- `max_turn_time_ms`: INTEGER DEFAULT 1500
- `interruption_threshold`: DECIMAL(3,2) DEFAULT 0.7
- `context_window_size`: INTEGER DEFAULT 32000
- `temperature`: DECIMAL(3,2) DEFAULT 0.3
- `automation_config`: JSONB DEFAULT '{
  "tools_enabled": true,
  "max_tool_calls": 5,
  "async_workflows": true,
  "timeout_ms": 30000
}'
- `phone_integration_type`: VARCHAR(20) DEFAULT 'managed' CHECK (phone_integration_type IN ('managed', 'sip_trunk', 'twilio'))
- `edge_optimization`: JSONB DEFAULT '{
  "enabled": true,
  "cache_responses": true,
  "preferred_regions": ["us-east-1", "eu-west-1"]
}'

#### Table: agent_versions
- `id`: UUID (PK)
- `agent_id`: UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE
- `version`: INTEGER NOT NULL
- `system_prompt`: TEXT NOT NULL
- `voice_config`: JSONB NOT NULL
- `language_config`: JSONB NOT NULL
- `model_config`: JSONB NOT NULL
- `settings`: JSONB DEFAULT '{}'
- `status`: VARCHAR(50) DEFAULT 'draft' CHECK (status IN ('draft', 'testing', 'published', 'archived'))
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `is_published`: BOOLEAN DEFAULT FALSE
- `published_at`: TIMESTAMP(6) WITH TIME ZONE
- `published_by_id`: UUID REFERENCES users(id)
- `automation_config`: JSONB DEFAULT '{}'
- `performance_metrics`: JSONB DEFAULT '{}'
- `a_b_test_config`: JSONB DEFAULT '{}'
- UNIQUE (agent_id, version)

#### Table: conversation_nodes
- `id`: UUID (PK)
- `agent_id`: UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE
- `agent_version_id`: UUID NOT NULL REFERENCES agent_versions(id) ON DELETE CASCADE
- `name`: VARCHAR(255) NOT NULL
- `node_type`: VARCHAR(50) NOT NULL CHECK (node_type IN ('main', 'response', 'tool_call', 'condition', 'transfer', 'end'))
- `content`: TEXT
- `conditions`: JSONB DEFAULT '{}'
- `next_node_ids`: JSONB DEFAULT '[]' -- Array of possible next nodes
- `tool_config`: JSONB DEFAULT '{}'
- `position_x`: INTEGER DEFAULT 0
- `position_y`: INTEGER DEFAULT 0
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)

### Schema: phone_manager (Enhanced)

#### Table: phone_numbers
- `id`: UUID (PK)
- `number`: VARCHAR(20) UNIQUE NOT NULL
- `formatted_number`: VARCHAR(30) NOT NULL -- E.164 format
- `country_code`: VARCHAR(3) NOT NULL
- `area_code`: VARCHAR(10)
- `local_number`: VARCHAR(15) NOT NULL
- `acquisition_method`: VARCHAR(20) NOT NULL CHECK (acquisition_method IN ('managed', 'sip_trunk', 'twilio', 'ported'))
- `provider`: VARCHAR(50) NOT NULL -- 'aybiza', 'twilio', 'external_sip'
- `provider_id`: VARCHAR(255) -- External provider identifier
- `capabilities`: JSONB DEFAULT '{
  "voice": true,
  "sms": false,
  "fax": false,
  "mms": false
}'
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `agent_id`: UUID REFERENCES agents(id) ON DELETE SET NULL
- `settings`: JSONB DEFAULT '{}'
- `monthly_cost`: DECIMAL(10,4) DEFAULT 1.00
- `setup_cost`: DECIMAL(10,4) DEFAULT 0.00
- `currency`: VARCHAR(3) DEFAULT 'USD'
- `status`: VARCHAR(50) DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'inactive', 'suspended', 'porting', 'failed'))
- `region`: VARCHAR(50) NOT NULL -- Geographic region for optimization
- `compliance_verified`: BOOLEAN DEFAULT FALSE
- `compliance_documents`: JSONB DEFAULT '{}'
- `emergency_services_enabled`: BOOLEAN DEFAULT TRUE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)
- `version`: INTEGER DEFAULT 1
- `last_health_check`: TIMESTAMP(6) WITH TIME ZONE
- `health_status`: VARCHAR(20) DEFAULT 'unknown' CHECK (health_status IN ('healthy', 'degraded', 'unhealthy', 'unknown'))

#### Table: phone_number_reservations
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `requested_area_code`: VARCHAR(10)
- `requested_country`: VARCHAR(100) NOT NULL
- `requested_region`: VARCHAR(100)
- `acquisition_method`: VARCHAR(20) NOT NULL CHECK (acquisition_method IN ('managed', 'sip_trunk', 'twilio', 'porting'))
- `status`: VARCHAR(50) DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'available', 'reserved', 'completed', 'failed', 'expired'))
- `available_numbers`: JSONB DEFAULT '[]' -- Array of available numbers
- `selected_number`: VARCHAR(20)
- `reservation_expires_at`: TIMESTAMP(6) WITH TIME ZONE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `metadata`: JSONB DEFAULT '{}'

#### Table: porting_requests
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `phone_number`: VARCHAR(20) NOT NULL
- `current_provider`: VARCHAR(100) NOT NULL
- `account_number`: VARCHAR(100) NOT NULL
- `pin_password`: TEXT -- Encrypted field
- `authorized_contact`: JSONB NOT NULL -- Contact information
- `service_address`: JSONB NOT NULL -- Service address
- `billing_address`: JSONB -- Billing address if different
- `supporting_documents`: JSONB DEFAULT '[]' -- Array of document URLs
- `status`: VARCHAR(50) DEFAULT 'pending' CHECK (status IN ('pending', 'submitted', 'in_progress', 'approved', 'rejected', 'completed', 'cancelled'))
- `provider_response`: JSONB DEFAULT '{}'
- `estimated_completion`: TIMESTAMP(6) WITH TIME ZONE
- `actual_completion`: TIMESTAMP(6) WITH TIME ZONE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `processed_by_id`: UUID REFERENCES users(id)

#### Table: sip_trunks
- `id`: UUID (PK)
- `name`: VARCHAR(255) NOT NULL
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `provider_name`: VARCHAR(100) NOT NULL
- `termination_uri`: VARCHAR(500) NOT NULL
- `origination_uris`: JSONB NOT NULL DEFAULT '[]' -- Array of origination URIs
- `authentication_method`: VARCHAR(50) DEFAULT 'ip_acl' CHECK (authentication_method IN ('ip_acl', 'credential', 'certificate'))
- `credentials`: JSONB -- Encrypted credentials if needed
- `ip_access_control_list`: JSONB DEFAULT '[]' -- Array of allowed IP addresses
- `codec_preferences`: JSONB DEFAULT '["PCMU", "PCMA", "G722"]'
- `max_concurrent_calls`: INTEGER DEFAULT 10
- `settings`: JSONB DEFAULT '{}'
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'testing', 'failed'))
- `last_health_check`: TIMESTAMP(6) WITH TIME ZONE
- `health_status`: VARCHAR(20) DEFAULT 'unknown' CHECK (health_status IN ('healthy', 'degraded', 'unhealthy', 'unknown'))
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)

### Schema: call_analytics (Enhanced with TimescaleDB)

#### Table: calls
- `id`: UUID (PK)
- `call_sid`: VARCHAR(255) UNIQUE NOT NULL
- `direction`: VARCHAR(10) NOT NULL CHECK (direction IN ('inbound', 'outbound'))
- `status`: VARCHAR(50) NOT NULL CHECK (status IN ('initiated', 'ringing', 'answered', 'completed', 'busy', 'failed', 'no-answer', 'cancelled'))
- `from_number`: VARCHAR(20) NOT NULL
- `to_number`: VARCHAR(20) NOT NULL
- `duration_seconds`: INTEGER DEFAULT 0
- `billable_duration_seconds`: INTEGER DEFAULT 0
- `start_time`: TIMESTAMP(6) WITH TIME ZONE NOT NULL
- `answer_time`: TIMESTAMP(6) WITH TIME ZONE
- `end_time`: TIMESTAMP(6) WITH TIME ZONE
- `recording_url`: VARCHAR(500)
- `recording_duration_seconds`: INTEGER
- `transcript_id`: VARCHAR(255)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id)
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id)
- `agent_id`: UUID REFERENCES agents(id)
- `agent_version_id`: UUID REFERENCES agent_versions(id)
- `phone_number_id`: UUID REFERENCES phone_numbers(id)
- `metadata`: JSONB DEFAULT '{}'
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `edge_region`: VARCHAR(50) -- Cloudflare edge region that handled the call
- `aws_region`: VARCHAR(50) -- AWS region that processed the call
- `cost_usd`: DECIMAL(10,6) DEFAULT 0.00
- `quality_score`: DECIMAL(3,2) -- AI-generated call quality score (0.00-1.00)
- `sentiment_score`: DECIMAL(3,2) -- Overall sentiment (-1.00 to 1.00)

#### Table: call_transcripts
- `id`: UUID (PK)
- `call_id`: UUID NOT NULL REFERENCES calls(id) ON DELETE CASCADE
- `call_sid`: VARCHAR(255) NOT NULL
- `turns`: JSONB NOT NULL DEFAULT '[]' -- Array of conversation turns
- `full_text`: TEXT
- `word_count`: INTEGER DEFAULT 0
- `speaker_count`: INTEGER DEFAULT 2
- `language_detected`: VARCHAR(10)
- `confidence_score`: DECIMAL(3,2) DEFAULT 0.00
- `processing_time_ms`: INTEGER
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `redacted`: BOOLEAN DEFAULT FALSE
- `redacted_at`: TIMESTAMP(6) WITH TIME ZONE
- `redacted_by_id`: UUID REFERENCES users(id)
- `pii_detected`: JSONB DEFAULT '{}' -- Types and locations of detected PII
- `summary`: TEXT -- AI-generated summary
- `key_topics`: JSONB DEFAULT '[]' -- Extracted topics and keywords

#### Table: call_analytics
- `id`: UUID (PK)
- `call_sid`: VARCHAR(255) NOT NULL
- `organization_id`: UUID NOT NULL REFERENCES organizations(id)
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id)
- `agent_id`: UUID REFERENCES agents(id)
- `agent_version_id`: UUID REFERENCES agent_versions(id)
- `phone_number_id`: UUID REFERENCES phone_numbers(id)
- `stt_latency_ms`: INTEGER DEFAULT 0
- `llm_first_token_ms`: INTEGER DEFAULT 0
- `llm_total_latency_ms`: INTEGER DEFAULT 0
- `tts_first_audio_ms`: INTEGER DEFAULT 0
- `tts_total_latency_ms`: INTEGER DEFAULT 0
- `total_latency_ms`: INTEGER DEFAULT 0
- `edge_cache_hits`: INTEGER DEFAULT 0
- `edge_cache_misses`: INTEGER DEFAULT 0
- `turns_count`: INTEGER DEFAULT 0
- `interruptions_count`: INTEGER DEFAULT 0
- `tool_calls_count`: INTEGER DEFAULT 0
- `successful_tool_calls`: INTEGER DEFAULT 0
- `failed_tool_calls`: INTEGER DEFAULT 0
- `tokens_used`: INTEGER DEFAULT 0
- `model_used`: VARCHAR(100)
- `voice_model_used`: VARCHAR(100)
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `automation_usage`: JSONB DEFAULT '{}'
- `performance_metrics`: JSONB DEFAULT '{}'

#### Table: detailed_metrics (TimescaleDB Hypertable)
- `id`: BIGSERIAL
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL
- `call_sid`: VARCHAR(255) NOT NULL
- `organization_id`: UUID NOT NULL
- `tenant_id`: UUID NOT NULL
- `agent_id`: UUID
- `agent_version_id`: UUID
- `phone_number_id`: UUID
- `metric_type`: VARCHAR(50) NOT NULL CHECK (metric_type IN ('latency', 'quality', 'cost', 'usage', 'error', 'security'))
- `metric_name`: VARCHAR(100) NOT NULL
- `metric_value`: DOUBLE PRECISION NOT NULL
- `metric_unit`: VARCHAR(20) DEFAULT 'ms'
- `metadata`: JSONB DEFAULT '{}'
- `edge_region`: VARCHAR(50)
- `aws_region`: VARCHAR(50)

### Schema: automation_engine (Enhanced)

#### Table: workflows
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `name`: VARCHAR(255) NOT NULL
- `description`: TEXT
- `category`: VARCHAR(100) DEFAULT 'custom' CHECK (category IN ('crm', 'communication', 'analytics', 'integration', 'custom'))
- `trigger_type`: VARCHAR(50) NOT NULL CHECK (trigger_type IN ('voice_agent', 'webhook', 'scheduled', 'manual', 'event'))
- `trigger_config`: JSONB DEFAULT '{}'
- `status`: VARCHAR(50) DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'inactive', 'archived'))
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)
- `version`: INTEGER DEFAULT 1
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `is_published`: BOOLEAN DEFAULT FALSE
- `published_at`: TIMESTAMP(6) WITH TIME ZONE
- `published_by_id`: UUID REFERENCES users(id)
- `execution_count`: INTEGER DEFAULT 0
- `success_rate`: DECIMAL(5,2) DEFAULT 0.00
- `avg_execution_time_ms`: INTEGER DEFAULT 0
- `metadata`: JSONB DEFAULT '{}'
- `is_template`: BOOLEAN DEFAULT FALSE
- `template_category`: VARCHAR(100)

#### Table: workflow_versions
- `id`: UUID (PK)
- `workflow_id`: UUID NOT NULL REFERENCES workflows(id) ON DELETE CASCADE
- `version`: INTEGER NOT NULL
- `trigger_config`: JSONB NOT NULL
- `status`: VARCHAR(50) DEFAULT 'draft' CHECK (status IN ('draft', 'testing', 'published', 'archived'))
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `is_published`: BOOLEAN DEFAULT FALSE
- `published_at`: TIMESTAMP(6) WITH TIME ZONE
- `published_by_id`: UUID REFERENCES users(id)
- `changelog`: TEXT
- `metadata`: JSONB DEFAULT '{}'
- UNIQUE (workflow_id, version)

#### Table: workflow_nodes
- `id`: UUID (PK)
- `workflow_id`: UUID NOT NULL REFERENCES workflows(id) ON DELETE CASCADE
- `workflow_version_id`: UUID NOT NULL REFERENCES workflow_versions(id) ON DELETE CASCADE
- `name`: VARCHAR(255) NOT NULL
- `node_type`: VARCHAR(50) NOT NULL CHECK (node_type IN ('trigger', 'action', 'condition', 'loop', 'parallel', 'tool', 'integration', 'end'))
- `config`: JSONB NOT NULL DEFAULT '{}'
- `tool_id`: UUID REFERENCES tools(id)
- `position_x`: INTEGER DEFAULT 0
- `position_y`: INTEGER DEFAULT 0
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)
- `timeout_ms`: INTEGER DEFAULT 30000
- `retry_count`: INTEGER DEFAULT 0
- `retry_delay_ms`: INTEGER DEFAULT 1000

#### Table: workflow_connections
- `id`: UUID (PK)
- `workflow_id`: UUID NOT NULL REFERENCES workflows(id) ON DELETE CASCADE
- `workflow_version_id`: UUID NOT NULL REFERENCES workflow_versions(id) ON DELETE CASCADE
- `source_node_id`: UUID NOT NULL REFERENCES workflow_nodes(id) ON DELETE CASCADE
- `target_node_id`: UUID NOT NULL REFERENCES workflow_nodes(id) ON DELETE CASCADE
- `condition`: JSONB DEFAULT '{}'
- `label`: VARCHAR(255)
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)

#### Table: tools
- `id`: UUID (PK)
- `organization_id`: UUID REFERENCES organizations(id) ON DELETE CASCADE
- `tenant_id`: UUID REFERENCES tenants(id) ON DELETE CASCADE
- `name`: VARCHAR(255) NOT NULL
- `description`: TEXT
- `category`: VARCHAR(100) DEFAULT 'custom' CHECK (category IN ('crm', 'communication', 'database', 'analytics', 'integration', 'ai', 'custom'))
- `tool_type`: VARCHAR(50) NOT NULL CHECK (tool_type IN ('http_api', 'database', 'webhook', 'function', 'ai_service', 'file_operation'))
- `config`: JSONB NOT NULL DEFAULT '{}'
- `auth_config`: JSONB DEFAULT '{}' -- Encrypted
- `input_schema`: JSONB NOT NULL -- JSON Schema for input validation
- `output_schema`: JSONB NOT NULL -- JSON Schema for output validation
- `created_by_id`: UUID REFERENCES users(id)
- `updated_by_id`: UUID REFERENCES users(id)
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `version`: INTEGER DEFAULT 1
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'deprecated'))
- `is_system_tool`: BOOLEAN DEFAULT FALSE
- `is_verified`: BOOLEAN DEFAULT FALSE
- `usage_count`: INTEGER DEFAULT 0
- `success_rate`: DECIMAL(5,2) DEFAULT 0.00
- `avg_execution_time_ms`: INTEGER DEFAULT 0

#### Table: tool_versions
- `id`: UUID (PK)
- `tool_id`: UUID NOT NULL REFERENCES tools(id) ON DELETE CASCADE
- `version`: INTEGER NOT NULL
- `config`: JSONB NOT NULL
- `auth_config`: JSONB DEFAULT '{}'
- `input_schema`: JSONB NOT NULL
- `output_schema`: JSONB NOT NULL
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `is_published`: BOOLEAN DEFAULT FALSE
- `published_at`: TIMESTAMP(6) WITH TIME ZONE
- `published_by_id`: UUID REFERENCES users(id)
- `status`: VARCHAR(50) DEFAULT 'draft' CHECK (status IN ('draft', 'testing', 'published', 'deprecated'))
- `changelog`: TEXT
- UNIQUE (tool_id, version)

#### Table: execution_logs (TimescaleDB Hypertable)
- `id`: UUID (PK)
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `workflow_id`: UUID NOT NULL REFERENCES workflows(id)
- `workflow_version_id`: UUID NOT NULL REFERENCES workflow_versions(id)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id)
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id)
- `trigger_type`: VARCHAR(50) NOT NULL CHECK (trigger_type IN ('voice_agent', 'webhook', 'scheduled', 'manual', 'event'))
- `trigger_id`: UUID -- Could be call_id, agent_id, etc.
- `trigger_data`: JSONB DEFAULT '{}'
- `status`: VARCHAR(50) DEFAULT 'started' CHECK (status IN ('started', 'running', 'completed', 'failed', 'timeout', 'cancelled'))
- `started_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `completed_at`: TIMESTAMP(6) WITH TIME ZONE
- `duration_ms`: INTEGER
- `error_message`: TEXT
- `error_details`: JSONB DEFAULT '{}'
- `input_data`: JSONB DEFAULT '{}'
- `output_data`: JSONB DEFAULT '{}'
- `metadata`: JSONB DEFAULT '{}'
- `nodes_executed`: INTEGER DEFAULT 0
- `nodes_succeeded`: INTEGER DEFAULT 0
- `nodes_failed`: INTEGER DEFAULT 0
- `cost_usd`: DECIMAL(10,6) DEFAULT 0.00

#### Table: node_executions
- `id`: UUID (PK)
- `execution_log_id`: UUID NOT NULL REFERENCES execution_logs(id) ON DELETE CASCADE
- `node_id`: UUID NOT NULL REFERENCES workflow_nodes(id)
- `node_name`: VARCHAR(255) NOT NULL
- `node_type`: VARCHAR(50) NOT NULL
- `status`: VARCHAR(50) DEFAULT 'started' CHECK (status IN ('started', 'running', 'completed', 'failed', 'skipped', 'timeout'))
- `started_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `completed_at`: TIMESTAMP(6) WITH TIME ZONE
- `duration_ms`: INTEGER
- `input_data`: JSONB DEFAULT '{}'
- `output_data`: JSONB DEFAULT '{}'
- `error_message`: TEXT
- `error_details`: JSONB DEFAULT '{}'
- `retry_count`: INTEGER DEFAULT 0
- `cost_usd`: DECIMAL(10,6) DEFAULT 0.00

### Schema: security (Enhanced)

#### Table: api_keys
- `id`: UUID (PK)
- `key_hash`: VARCHAR(255) UNIQUE NOT NULL
- `name`: VARCHAR(255) NOT NULL
- `description`: TEXT
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `tenant_id`: UUID REFERENCES tenants(id) ON DELETE CASCADE
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- `permissions`: JSONB NOT NULL DEFAULT '{}'
- `rate_limit`: JSONB DEFAULT '{
  "requests_per_minute": 100,
  "requests_per_hour": 1000,
  "requests_per_day": 10000
}'
- `ip_whitelist`: JSONB DEFAULT '[]' -- Array of allowed IP addresses/CIDR
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'revoked', 'expired'))
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `expires_at`: TIMESTAMP(6) WITH TIME ZONE
- `last_used_at`: TIMESTAMP(6) WITH TIME ZONE
- `usage_count`: INTEGER DEFAULT 0
- `version`: INTEGER DEFAULT 1

#### Table: audit_logs (TimescaleDB Hypertable)
- `id`: UUID (PK)
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `organization_id`: UUID NOT NULL REFERENCES organizations(id)
- `tenant_id`: UUID REFERENCES tenants(id)
- `user_id`: UUID REFERENCES users(id)
- `api_key_id`: UUID REFERENCES api_keys(id)
- `action`: VARCHAR(100) NOT NULL
- `resource_type`: VARCHAR(100) NOT NULL
- `resource_id`: UUID
- `resource_name`: VARCHAR(255)
- `ip_address`: INET
- `user_agent`: TEXT
- `request_id`: UUID
- `session_id`: VARCHAR(255)
- `details`: JSONB DEFAULT '{}'
- `status`: VARCHAR(50) DEFAULT 'success' CHECK (status IN ('success', 'failure', 'error'))
- `error_message`: TEXT
- `duration_ms`: INTEGER
- `edge_region`: VARCHAR(50)
- `aws_region`: VARCHAR(50)

#### Table: security_events (TimescaleDB Hypertable)
- `id`: UUID (PK)
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `organization_id`: UUID NOT NULL REFERENCES organizations(id)
- `tenant_id`: UUID REFERENCES tenants(id)
- `event_type`: VARCHAR(100) NOT NULL CHECK (event_type IN ('login_failure', 'suspicious_activity', 'rate_limit_exceeded', 'unauthorized_access', 'data_breach', 'system_anomaly'))
- `severity`: VARCHAR(20) NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical'))
- `source`: VARCHAR(100) NOT NULL
- `ip_address`: INET
- `user_id`: UUID REFERENCES users(id)
- `api_key_id`: UUID REFERENCES api_keys(id)
- `details`: JSONB DEFAULT '{}'
- `resolved`: BOOLEAN DEFAULT FALSE
- `resolved_at`: TIMESTAMP(6) WITH TIME ZONE
- `resolved_by`: UUID REFERENCES users(id)
- `resolution_notes`: TEXT
- `automated_response`: JSONB DEFAULT '{}'
- `risk_score`: DECIMAL(3,2) DEFAULT 0.00

#### Table: prompt_injection_attempts (TimescaleDB Hypertable)
- `id`: UUID (PK)
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `organization_id`: UUID NOT NULL REFERENCES organizations(id)
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id)
- `agent_id`: UUID REFERENCES agents(id)
- `agent_version_id`: UUID REFERENCES agent_versions(id)
- `call_sid`: VARCHAR(255)
- `input_text`: TEXT NOT NULL
- `detection_method`: VARCHAR(100) NOT NULL CHECK (detection_method IN ('rule_based', 'ml_model', 'llm_detection', 'pattern_matching'))
- `confidence_score`: DECIMAL(3,2) NOT NULL
- `threat_level`: VARCHAR(20) DEFAULT 'medium' CHECK (threat_level IN ('low', 'medium', 'high', 'critical'))
- `action_taken`: VARCHAR(100) NOT NULL CHECK (action_taken IN ('blocked', 'sanitized', 'logged', 'escalated'))
- `metadata`: JSONB DEFAULT '{}'
- `false_positive`: BOOLEAN
- `reviewed_by`: UUID REFERENCES users(id)
- `reviewed_at`: TIMESTAMP(6) WITH TIME ZONE

#### Table: voice_security_events (TimescaleDB Hypertable)
- `id`: UUID (PK)
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `organization_id`: UUID NOT NULL REFERENCES organizations(id)
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id)
- `agent_id`: UUID REFERENCES agents(id)
- `agent_version_id`: UUID REFERENCES agent_versions(id)
- `call_sid`: VARCHAR(255) NOT NULL
- `event_type`: VARCHAR(100) NOT NULL CHECK (event_type IN ('voice_cloning', 'deepfake_audio', 'caller_spoofing', 'audio_anomaly', 'suspicious_patterns'))
- `detection_method`: VARCHAR(100) NOT NULL CHECK (detection_method IN ('ai_model', 'voice_biometrics', 'pattern_analysis', 'frequency_analysis'))
- `confidence_score`: DECIMAL(3,2) NOT NULL
- `threat_level`: VARCHAR(20) DEFAULT 'medium' CHECK (threat_level IN ('low', 'medium', 'high', 'critical'))
- `action_taken`: VARCHAR(100) NOT NULL CHECK (action_taken IN ('call_terminated', 'escalated', 'logged', 'user_notified'))
- `audio_sample_url`: VARCHAR(500) -- Encrypted storage location
- `metadata`: JSONB DEFAULT '{}'
- `false_positive`: BOOLEAN
- `reviewed_by`: UUID REFERENCES users(id)
- `reviewed_at`: TIMESTAMP(6) WITH TIME ZONE

#### Table: api_access_logs (TimescaleDB Hypertable)
- `id`: UUID (PK)
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `organization_id`: UUID REFERENCES organizations(id)
- `tenant_id`: UUID REFERENCES tenants(id)
- `user_id`: UUID REFERENCES users(id)
- `api_key_id`: UUID REFERENCES api_keys(id)
- `endpoint`: VARCHAR(255) NOT NULL
- `method`: VARCHAR(10) NOT NULL
- `status_code`: INTEGER NOT NULL
- `response_time_ms`: INTEGER NOT NULL
- `request_size_bytes`: INTEGER DEFAULT 0
- `response_size_bytes`: INTEGER DEFAULT 0
- `ip_address`: INET
- `user_agent`: TEXT
- `request_id`: UUID
- `request_body_hash`: VARCHAR(64) -- SHA-256 hash for sensitive data
- `rate_limited`: BOOLEAN DEFAULT FALSE
- `cached_response`: BOOLEAN DEFAULT FALSE
- `edge_region`: VARCHAR(50)
- `aws_region`: VARCHAR(50)
- `error_details`: JSONB DEFAULT '{}'

#### Table: compliance_reports
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `tenant_id`: UUID REFERENCES tenants(id) ON DELETE CASCADE
- `report_type`: VARCHAR(50) NOT NULL CHECK (report_type IN ('soc2', 'hipaa', 'gdpr', 'pci_dss', 'custom'))
- `reporting_period_start`: TIMESTAMP(6) WITH TIME ZONE NOT NULL
- `reporting_period_end`: TIMESTAMP(6) WITH TIME ZONE NOT NULL
- `status`: VARCHAR(50) DEFAULT 'generating' CHECK (status IN ('generating', 'completed', 'failed'))
- `report_data`: JSONB NOT NULL DEFAULT '{}'
- `findings`: JSONB DEFAULT '[]'
- `recommendations`: JSONB DEFAULT '[]'
- `generated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `generated_by_id`: UUID NOT NULL REFERENCES users(id)
- `report_file_url`: VARCHAR(500) -- Encrypted storage location
- `checksum`: VARCHAR(64) -- SHA-256 checksum for integrity

## Indexes (Optimized for PostgreSQL 16.9+)

### Core Entity Indexes
```sql
-- Organizations
CREATE UNIQUE INDEX CONCURRENTLY idx_organizations_slug ON organizations(slug);
CREATE INDEX CONCURRENTLY idx_organizations_status_created ON organizations(status, created_at DESC);
CREATE INDEX CONCURRENTLY idx_organizations_compliance ON organizations(compliance_status, compliance_verified_at);

-- Tenants
CREATE INDEX CONCURRENTLY idx_tenants_org_status ON tenants(organization_id, status);
CREATE UNIQUE INDEX CONCURRENTLY idx_tenants_subdomain ON tenants(subdomain);
CREATE INDEX CONCURRENTLY idx_tenants_plan_tier ON tenants(plan_tier, status);

-- Users
CREATE UNIQUE INDEX CONCURRENTLY idx_users_email_active ON users(email) WHERE is_active = true;
CREATE INDEX CONCURRENTLY idx_users_org_tenant ON users(organization_id, tenant_id, is_active);
CREATE INDEX CONCURRENTLY idx_users_last_login ON users(last_login DESC) WHERE is_active = true;
CREATE INDEX CONCURRENTLY idx_users_mfa_enabled ON users(mfa_enabled, is_active);

-- Agents
CREATE INDEX CONCURRENTLY idx_agents_tenant_status ON agents(tenant_id, status);
CREATE INDEX CONCURRENTLY idx_agents_integration_type ON agents(phone_integration_type, status);
CREATE INDEX CONCURRENTLY idx_agents_optimization ON agents(latency_optimization_level, status);

-- Agent Versions
CREATE INDEX CONCURRENTLY idx_agent_versions_published ON agent_versions(agent_id, is_published, version DESC);
CREATE INDEX CONCURRENTLY idx_agent_versions_status ON agent_versions(status, created_at DESC);
```

### Phone Manager Indexes
```sql
-- Phone Numbers
CREATE UNIQUE INDEX CONCURRENTLY idx_phone_numbers_number ON phone_numbers(number);
CREATE INDEX CONCURRENTLY idx_phone_numbers_tenant_status ON phone_numbers(tenant_id, status);
CREATE INDEX CONCURRENTLY idx_phone_numbers_acquisition ON phone_numbers(acquisition_method, status);
CREATE INDEX CONCURRENTLY idx_phone_numbers_region ON phone_numbers(region, status);
CREATE INDEX CONCURRENTLY idx_phone_numbers_health ON phone_numbers(health_status, last_health_check);

-- Phone Number Reservations
CREATE INDEX CONCURRENTLY idx_reservations_tenant_status ON phone_number_reservations(tenant_id, status);
CREATE INDEX CONCURRENTLY idx_reservations_expires ON phone_number_reservations(reservation_expires_at) WHERE status IN ('reserved', 'available');

-- Porting Requests
CREATE INDEX CONCURRENTLY idx_porting_tenant_status ON porting_requests(tenant_id, status);
CREATE INDEX CONCURRENTLY idx_porting_completion ON porting_requests(estimated_completion, status);

-- SIP Trunks
CREATE INDEX CONCURRENTLY idx_sip_trunks_tenant_status ON sip_trunks(tenant_id, status);
CREATE INDEX CONCURRENTLY idx_sip_trunks_health ON sip_trunks(health_status, last_health_check);
```

### Call Analytics Indexes (Optimized for TimescaleDB)
```sql
-- Calls
CREATE INDEX CONCURRENTLY idx_calls_tenant_time ON calls(tenant_id, start_time DESC);
CREATE INDEX CONCURRENTLY idx_calls_agent_time ON calls(agent_id, start_time DESC);
CREATE INDEX CONCURRENTLY idx_calls_status_time ON calls(status, start_time DESC);
CREATE INDEX CONCURRENTLY idx_calls_phone_number ON calls(phone_number_id, start_time DESC);
CREATE INDEX CONCURRENTLY idx_calls_duration ON calls(duration_seconds, status);
CREATE INDEX CONCURRENTLY idx_calls_cost ON calls(cost_usd, start_time DESC);

-- Call Analytics
CREATE INDEX CONCURRENTLY idx_call_analytics_tenant ON call_analytics(tenant_id, call_sid);
CREATE INDEX CONCURRENTLY idx_call_analytics_latency ON call_analytics(total_latency_ms, created_at DESC);
CREATE INDEX CONCURRENTLY idx_call_analytics_performance ON call_analytics(model_used, total_latency_ms);

-- Call Transcripts
CREATE INDEX CONCURRENTLY idx_transcripts_call ON call_transcripts(call_id);
CREATE INDEX CONCURRENTLY idx_transcripts_redacted ON call_transcripts(redacted, redacted_at);
CREATE INDEX CONCURRENTLY idx_transcripts_language ON call_transcripts(language_detected, confidence_score);
```

### Automation Engine Indexes
```sql
-- Workflows
CREATE INDEX CONCURRENTLY idx_workflows_tenant_status ON workflows(tenant_id, status);
CREATE INDEX CONCURRENTLY idx_workflows_category ON workflows(category, status);
CREATE INDEX CONCURRENTLY idx_workflows_template ON workflows(is_template, template_category);
CREATE INDEX CONCURRENTLY idx_workflows_performance ON workflows(success_rate DESC, execution_count DESC);

-- Tools
CREATE INDEX CONCURRENTLY idx_tools_tenant_category ON tools(tenant_id, category, status);
CREATE INDEX CONCURRENTLY idx_tools_system ON tools(is_system_tool, status);
CREATE INDEX CONCURRENTLY idx_tools_verified ON tools(is_verified, usage_count DESC);

-- Execution Logs (TimescaleDB optimized)
CREATE INDEX CONCURRENTLY idx_execution_logs_tenant_time ON execution_logs(tenant_id, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_execution_logs_workflow_time ON execution_logs(workflow_id, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_execution_logs_status ON execution_logs(status, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_execution_logs_trigger ON execution_logs(trigger_type, trigger_id, timestamp DESC);
```

### Security Indexes (TimescaleDB optimized)
```sql
-- Audit Logs
CREATE INDEX CONCURRENTLY idx_audit_logs_tenant_time ON audit_logs(tenant_id, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_audit_logs_user_action ON audit_logs(user_id, action, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_audit_logs_resource ON audit_logs(resource_type, resource_id, timestamp DESC);

-- Security Events
CREATE INDEX CONCURRENTLY idx_security_events_tenant_time ON security_events(tenant_id, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_security_events_severity ON security_events(severity, resolved, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_security_events_type ON security_events(event_type, timestamp DESC);

-- API Access Logs
CREATE INDEX CONCURRENTLY idx_api_logs_tenant_time ON api_access_logs(tenant_id, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_api_logs_endpoint_time ON api_access_logs(endpoint, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_api_logs_status_time ON api_access_logs(status_code, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_api_logs_performance ON api_access_logs(response_time_ms DESC, timestamp DESC);
```

## TimescaleDB Configuration (Enhanced for 2.20.0+)

### Hypertables with Optimized Chunking
```sql
-- Create hypertables with optimized chunk intervals

SELECT create_hypertable('detailed_metrics', 'timestamp', chunk_time_interval => INTERVAL '1 hour');
SELECT create_hypertable('execution_logs', 'timestamp', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('audit_logs', 'timestamp', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('security_events', 'timestamp', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('prompt_injection_attempts', 'timestamp', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('voice_security_events', 'timestamp', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('api_access_logs', 'timestamp', chunk_time_interval => INTERVAL '1 hour');

-- Add space dimensions for better performance
SELECT add_dimension('detailed_metrics', 'tenant_id', number_partitions => 4);
SELECT add_dimension('execution_logs', 'tenant_id', number_partitions => 4);
SELECT add_dimension('audit_logs', 'tenant_id', number_partitions => 4);
SELECT add_dimension('security_events', 'tenant_id', number_partitions => 4);
SELECT add_dimension('api_access_logs', 'tenant_id', number_partitions => 4);
```

### Continuous Aggregates (New in TimescaleDB 2.20.0)
```sql
-- Hourly call analytics aggregations
CREATE MATERIALIZED VIEW call_analytics_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', start_time) AS hour,
       tenant_id,
       agent_id,
       COUNT(*) AS total_calls,
       AVG(duration_seconds) AS avg_duration,
       AVG(quality_score) AS avg_quality,
       SUM(cost_usd) AS total_cost,
       COUNT(*) FILTER (WHERE status = 'completed') AS successful_calls
FROM calls
GROUP BY hour, tenant_id, agent_id;

-- Daily security events aggregations
CREATE MATERIALIZED VIEW security_events_daily
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 day', timestamp) AS day,
       tenant_id,
       event_type,
       severity,
       COUNT(*) AS event_count,
       COUNT(*) FILTER (WHERE resolved = true) AS resolved_count
FROM security_events
GROUP BY day, tenant_id, event_type, severity;

-- Hourly performance metrics
CREATE MATERIALIZED VIEW performance_metrics_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', timestamp) AS hour,
       tenant_id,
       metric_name,
       AVG(metric_value) AS avg_value,
       MIN(metric_value) AS min_value,
       MAX(metric_value) AS max_value,
       COUNT(*) AS sample_count
FROM detailed_metrics
WHERE metric_type = 'latency'
GROUP BY hour, tenant_id, metric_name;
```

### Compression Policies (Enhanced in 2.20.0)
```sql
-- Enable compression after 7 days for better storage efficiency
SELECT add_compression_policy('detailed_metrics', INTERVAL '7 days');
SELECT add_compression_policy('execution_logs', INTERVAL '7 days');
SELECT add_compression_policy('audit_logs', INTERVAL '7 days');
SELECT add_compression_policy('security_events', INTERVAL '7 days');
SELECT add_compression_policy('api_access_logs', INTERVAL '7 days');

-- Retention policies for automatic cleanup
SELECT add_retention_policy('detailed_metrics', INTERVAL '2 years');
SELECT add_retention_policy('execution_logs', INTERVAL '1 year');
SELECT add_retention_policy('audit_logs', INTERVAL '7 years'); -- Compliance requirement
SELECT add_retention_policy('security_events', INTERVAL '7 years'); -- Compliance requirement
SELECT add_retention_policy('api_access_logs', INTERVAL '1 year');
```

### Row-Level Security (RLS) for Multi-Tenancy
```sql
-- Enable RLS on all tenant-isolated tables
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE agents ENABLE ROW LEVEL SECURITY;
ALTER TABLE phone_numbers ENABLE ROW LEVEL SECURITY;
ALTER TABLE calls ENABLE ROW LEVEL SECURITY;
ALTER TABLE workflows ENABLE ROW LEVEL SECURITY;
ALTER TABLE tools ENABLE ROW LEVEL SECURITY;

-- Create policies for tenant isolation
CREATE POLICY tenant_isolation_users ON users
    FOR ALL TO application_role
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_agents ON agents
    FOR ALL TO application_role
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Organization-level policies for admin access
CREATE POLICY org_isolation_users ON users
    FOR ALL TO admin_role
    USING (organization_id = current_setting('app.current_organization_id')::UUID);
```

This updated database schema reflects the current AYBIZA platform with hybrid Cloudflare+AWS architecture, enhanced phone number management, latest PostgreSQL 16.9 features, TimescaleDB 2.20.0 capabilities, and comprehensive security and compliance requirements.