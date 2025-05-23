# AYBIZA AI Voice Agent Platform - Database Schema (Billion-Scale)

## Hybrid Database Architecture for Billion-Scale Operations

### Primary Database Stack
- **PostgreSQL**: 16.9+ with Citus 12.1+ for horizontal sharding
- **DynamoDB**: For hot data (agent state, real-time sessions)
- **Pinecone/Weaviate**: Vector DB for semantic memory and embeddings
- **Redis**: 7.2+ for session cache and rate limiting
- **S3**: Call recordings, transcripts, Files API storage
- **TimescaleDB**: 2.20.0+ for time-series analytics

### PostgreSQL Configuration with Citus
- **Sharding Strategy**: Distributed by tenant_id across 128 shards
- **Connection Pooling**: PgBouncer with 10,000+ connection capacity
- **Replication**: Multi-region active-active with conflict resolution
- **Backup**: Continuous archiving with indefinite retention for compliance

### Schema: agent_manager

#### Table: organizations
- `id`: UUID (PK)
- `account_id`: VARCHAR(20) UNIQUE NOT NULL -- Format: AYB-XXXX-XXXX-XXXX
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
- `user_id`: VARCHAR(20) UNIQUE NOT NULL -- Format: USR-XXXXXXXXXX
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
  "opus_4": "anthropic.claude-opus-4-20250514-v1:0",
  "sonnet_4": "anthropic.claude-sonnet-4-20250514-v1:0",
  "selection_strategy": "auto",
  "extended_thinking": false,
  "prompt_caching_enabled": true,
  "cache_ttl_seconds": 3600
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
  "max_tool_calls": 10,
  "parallel_tool_execution": true,
  "async_workflows": true,
  "timeout_ms": 30000,
  "mcp_enabled": false,
  "code_execution_enabled": false,
  "files_api_enabled": false,
  "web_search_enabled": true
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

### Schema: agent_intelligence (NEW - Claude 4 Capabilities)

#### Table: agent_sessions (DynamoDB Primary)
- `agent_session_id`: UUID (PK)
- `agent_id`: UUID NOT NULL
- `tenant_id`: UUID NOT NULL
- `session_state`: JSONB -- Real-time agent state
- `active_tools`: JSONB DEFAULT '[]' -- Currently executing tools
- `memory_context`: JSONB DEFAULT '{}' -- Working memory
- `conversation_buffer`: JSONB DEFAULT '[]' -- Recent turns
- `model_in_use`: VARCHAR(100) -- Current Claude model
- `thinking_state`: VARCHAR(50) CHECK (thinking_state IN ('idle', 'thinking', 'extended_thinking'))
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `ttl`: INTEGER DEFAULT 3600 -- DynamoDB TTL

#### Table: agent_configurations
- `id`: UUID (PK)
- `agent_id`: UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `claude_4_config`: JSONB DEFAULT '{
  "opus_enabled": true,
  "sonnet_enabled": true,
  "extended_thinking_allowed": false,
  "max_thinking_time_ms": 30000,
  "parallel_tools_max": 5,
  "code_execution_allowed": false,
  "mcp_connectors": [],
  "files_api_enabled": false,
  "web_search_enabled": true,
  "prompt_cache_strategy": "adaptive"
}'
- `tool_permissions`: JSONB DEFAULT '{}' -- Granular tool access
- `memory_config`: JSONB DEFAULT '{
  "vector_db_enabled": true,
  "max_memory_items": 1000,
  "memory_ttl_days": 365
}'
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `version`: INTEGER DEFAULT 1

#### Table: agent_memories (Vector DB Primary)
- `id`: UUID (PK)
- `agent_id`: UUID NOT NULL
- `tenant_id`: UUID NOT NULL
- `memory_type`: VARCHAR(50) CHECK (memory_type IN ('fact', 'preference', 'context', 'skill'))
- `content`: TEXT NOT NULL
- `embedding`: VECTOR(1536) -- For semantic search
- `metadata`: JSONB DEFAULT '{}'
- `importance_score`: DECIMAL(3,2) DEFAULT 0.50
- `access_count`: INTEGER DEFAULT 0
- `last_accessed`: TIMESTAMP(6) WITH TIME ZONE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `expires_at`: TIMESTAMP(6) WITH TIME ZONE

#### Table: mcp_connectors
- `id`: UUID (PK)
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `name`: VARCHAR(255) NOT NULL
- `connector_type`: VARCHAR(100) NOT NULL -- 'hubspot', 'postgresql', 'google_drive', etc.
- `configuration`: JSONB NOT NULL -- Encrypted connection details
- `permissions`: JSONB DEFAULT '{}'
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'error'))
- `last_health_check`: TIMESTAMP(6) WITH TIME ZONE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()

#### Table: agent_files (S3 + Metadata)
- `id`: UUID (PK)
- `file_key`: VARCHAR(500) UNIQUE NOT NULL -- S3 key
- `agent_id`: UUID NOT NULL
- `tenant_id`: UUID NOT NULL
- `file_name`: VARCHAR(255) NOT NULL
- `file_type`: VARCHAR(100) NOT NULL
- `file_size_bytes`: BIGINT NOT NULL
- `content_hash`: VARCHAR(64) NOT NULL -- SHA-256
- `metadata`: JSONB DEFAULT '{}'
- `access_permissions`: JSONB DEFAULT '{}'
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `last_accessed`: TIMESTAMP(6) WITH TIME ZONE
- `access_count`: INTEGER DEFAULT 0
- `expires_at`: TIMESTAMP(6) WITH TIME ZONE

#### Table: tool_executions (TimescaleDB Hypertable)
- `id`: UUID (PK)
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `agent_id`: UUID NOT NULL
- `tenant_id`: UUID NOT NULL
- `call_sid`: VARCHAR(255)
- `tool_name`: VARCHAR(255) NOT NULL
- `tool_type`: VARCHAR(100) NOT NULL
- `execution_mode`: VARCHAR(50) CHECK (execution_mode IN ('sequential', 'parallel'))
- `input_data`: JSONB NOT NULL
- `output_data`: JSONB
- `status`: VARCHAR(50) CHECK (status IN ('pending', 'running', 'completed', 'failed'))
- `started_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL
- `completed_at`: TIMESTAMP(6) WITH TIME ZONE
- `duration_ms`: INTEGER
- `error_details`: JSONB
- `cost_usd`: DECIMAL(10,6) DEFAULT 0.00

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

### Schema: compliance (NEW - Consent & Call Recording Management)

#### Table: consent_records
- `id`: UUID (PK)
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `phone_number`: VARCHAR(20) NOT NULL
- `consent_type`: VARCHAR(50) NOT NULL CHECK (consent_type IN ('marketing', 'service', 'emergency'))
- `consent_status`: VARCHAR(50) NOT NULL CHECK (consent_status IN ('opted_in', 'opted_out', 'expired'))
- `opted_in_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL
- `opted_out_at`: TIMESTAMP(6) WITH TIME ZONE
- `expires_at`: TIMESTAMP(6) WITH TIME ZONE
- `verification_method`: VARCHAR(50) NOT NULL CHECK (verification_method IN ('checkbox', 'sms', 'voice', 'email', 'trusted_form'))
- `verification_details`: JSONB DEFAULT '{}'
- `ip_address`: INET
- `user_agent`: TEXT
- `source`: VARCHAR(100) -- 'web_form', 'api', 'import', etc.
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID REFERENCES users(id)
- INDEX idx_consent_phone_type (phone_number, consent_type, consent_status)
- INDEX idx_consent_tenant_status (tenant_id, consent_status, expires_at)

#### Table: call_recordings
- `id`: UUID (PK)
- `call_id`: UUID NOT NULL REFERENCES calls(id) ON DELETE CASCADE
- `call_sid`: VARCHAR(255) NOT NULL
- `tenant_id`: UUID NOT NULL
- `recording_url`: VARCHAR(500) NOT NULL -- S3 URL
- `recording_key`: VARCHAR(500) NOT NULL -- S3 key
- `duration_seconds`: INTEGER NOT NULL
- `file_size_bytes`: BIGINT NOT NULL
- `format`: VARCHAR(20) DEFAULT 'mp3' CHECK (format IN ('mp3', 'wav', 'opus'))
- `storage_tier`: VARCHAR(50) DEFAULT 'standard' CHECK (storage_tier IN ('standard', 'infrequent_access', 'glacier'))
- `encryption_key_id`: VARCHAR(255) -- KMS key ID
- `retention_policy`: VARCHAR(50) NOT NULL -- 'us_7_years', 'uk_6_years', 'default_3_years'
- `scheduled_deletion`: TIMESTAMP(6) WITH TIME ZONE
- `transcript_available`: BOOLEAN DEFAULT FALSE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `accessed_at`: TIMESTAMP(6) WITH TIME ZONE
- `access_count`: INTEGER DEFAULT 0
- INDEX idx_recordings_tenant_time (tenant_id, created_at DESC)
- INDEX idx_recordings_deletion (scheduled_deletion) WHERE scheduled_deletion IS NOT NULL

#### Table: inter_agent_communications
- `id`: UUID (PK)
- `from_agent_id`: UUID NOT NULL
- `to_agent_id`: UUID NOT NULL
- `tenant_id`: UUID NOT NULL
- `communication_type`: VARCHAR(50) CHECK (communication_type IN ('transfer', 'collaboration', 'escalation'))
- `context_packet`: JSONB NOT NULL -- Conversation history, entities, intent
- `status`: VARCHAR(50) CHECK (status IN ('initiated', 'accepted', 'completed', 'failed'))
- `initiated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `completed_at`: TIMESTAMP(6) WITH TIME ZONE
- `metadata`: JSONB DEFAULT '{}'

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

#### Table: customer_credentials (NEW - KMS Encrypted)
- `id`: UUID (PK)
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `credential_type`: VARCHAR(100) NOT NULL -- 'api_key', 'oauth_token', 'database', 'mcp_connector'
- `credential_name`: VARCHAR(255) NOT NULL
- `encrypted_data`: TEXT NOT NULL -- Base64 encoded encrypted credential
- `data_key_encrypted`: TEXT NOT NULL -- KMS encrypted data key
- `encryption_context`: JSONB NOT NULL -- Additional AAD for decryption
- `kms_key_alias`: VARCHAR(255) NOT NULL -- Tenant-specific KMS key
- `metadata`: JSONB DEFAULT '{}' -- Non-sensitive metadata
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'expired', 'rotating'))
- `expires_at`: TIMESTAMP(6) WITH TIME ZONE
- `last_rotated_at`: TIMESTAMP(6) WITH TIME ZONE
- `last_accessed_at`: TIMESTAMP(6) WITH TIME ZONE
- `access_count`: INTEGER DEFAULT 0
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by_id`: UUID NOT NULL REFERENCES users(id)
- INDEX idx_credentials_tenant_type (tenant_id, credential_type, status)
- INDEX idx_credentials_expires (expires_at) WHERE expires_at IS NOT NULL

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

## Billion-Scale Architecture with Citus

### Citus Distributed Tables Configuration
```sql
-- Distribute primary tables by tenant_id for horizontal scaling
SELECT create_distributed_table('tenants', 'id');
SELECT create_distributed_table('users', 'tenant_id');
SELECT create_distributed_table('agents', 'tenant_id');
SELECT create_distributed_table('agent_configurations', 'tenant_id');
SELECT create_distributed_table('phone_numbers', 'tenant_id');
SELECT create_distributed_table('calls', 'tenant_id');
SELECT create_distributed_table('call_transcripts', 'tenant_id');
SELECT create_distributed_table('workflows', 'tenant_id');
SELECT create_distributed_table('tools', 'tenant_id');
SELECT create_distributed_table('consent_records', 'tenant_id');

-- Co-locate related tables for efficient joins
SELECT create_distributed_table('agent_versions', 'tenant_id', colocate_with => 'agents');
SELECT create_distributed_table('conversation_nodes', 'tenant_id', colocate_with => 'agents');
SELECT create_distributed_table('workflow_versions', 'tenant_id', colocate_with => 'workflows');
SELECT create_distributed_table('workflow_nodes', 'tenant_id', colocate_with => 'workflows');

-- Reference tables (replicated to all nodes)
SELECT create_reference_table('organizations');
SELECT create_reference_table('roles');
```

### DynamoDB Tables for Hot Data
```yaml
AgentSessions:
  PartitionKey: agent_session_id
  SortKey: timestamp
  GlobalSecondaryIndexes:
    - Name: TenantAgentIndex
      PartitionKey: tenant_id
      SortKey: agent_id
  BillingMode: ON_DEMAND
  StreamEnabled: true
  TTL:
    AttributeName: ttl
    Enabled: true

ActiveToolExecutions:
  PartitionKey: agent_id
  SortKey: execution_id
  GlobalSecondaryIndexes:
    - Name: TenantToolIndex
      PartitionKey: tenant_id
      SortKey: timestamp
  BillingMode: ON_DEMAND
  StreamEnabled: true

RealtimeCallState:
  PartitionKey: call_sid
  SortKey: timestamp
  BillingMode: ON_DEMAND
  TTL:
    AttributeName: ttl
    Enabled: true
```

### Vector Database Schema (Pinecone/Weaviate)
```json
{
  "class": "AgentMemory",
  "description": "Long-term memory storage for agents",
  "vectorIndexType": "hnsw",
  "vectorizer": "text2vec-openai",
  "properties": [
    {
      "name": "content",
      "dataType": ["text"],
      "description": "The memory content"
    },
    {
      "name": "agentId",
      "dataType": ["string"],
      "description": "Agent UUID"
    },
    {
      "name": "tenantId",
      "dataType": ["string"],
      "description": "Tenant UUID for isolation"
    },
    {
      "name": "memoryType",
      "dataType": ["string"],
      "description": "Type of memory: fact, preference, context, skill"
    },
    {
      "name": "importanceScore",
      "dataType": ["number"],
      "description": "Relevance score 0-1"
    },
    {
      "name": "metadata",
      "dataType": ["object"],
      "description": "Additional context"
    }
  ]
}
```

### Data Distribution Strategy

#### Sharding Key Selection
- **Primary shard key**: `tenant_id` (ensures data locality)
- **Secondary considerations**: Geographic distribution
- **Shard count**: 128 initial shards (allows 100M+ tenants)

#### Cross-Database Query Patterns
```elixir
defmodule Aybiza.Database.BillionScale do
  @doc """
  Fetch agent state from hybrid storage
  """
  def get_agent_state(agent_id, tenant_id) do
    # 1. Check DynamoDB for hot state
    case DynamoDB.get_item("AgentSessions", %{agent_session_id: agent_id}) do
      {:ok, session} when not is_nil(session) ->
        {:ok, :hot, session}
      
      _ ->
        # 2. Fall back to PostgreSQL
        case Repo.get_by(AgentConfiguration, agent_id: agent_id, tenant_id: tenant_id) do
          nil -> {:error, :not_found}
          config -> {:ok, :cold, config}
        end
    end
  end

  @doc """
  Store tool execution across databases
  """
  def record_tool_execution(execution_data) do
    # 1. Write to DynamoDB for real-time access
    DynamoDB.put_item("ActiveToolExecutions", execution_data)
    
    # 2. Write to TimescaleDB for analytics
    %ToolExecution{}
    |> ToolExecution.changeset(execution_data)
    |> Repo.insert()
    
    # 3. Update cost metrics
    update_usage_metrics(execution_data)
  end
end
```

### Cost Optimization at Scale

#### Storage Tiering Strategy
1. **Hot (0-7 days)**: DynamoDB + S3 Standard
2. **Warm (7-90 days)**: PostgreSQL + S3 IA
3. **Cold (90+ days)**: Compressed PostgreSQL + S3 Glacier
4. **Archive (1+ year)**: S3 Glacier Deep Archive

#### Query Optimization
```sql
-- Partial indexes for common queries
CREATE INDEX CONCURRENTLY idx_active_agents ON agents(tenant_id, status) 
  WHERE status = 'active';

CREATE INDEX CONCURRENTLY idx_recent_calls ON calls(tenant_id, start_time) 
  WHERE start_time > NOW() - INTERVAL '7 days';

-- Materialized views for expensive aggregations
CREATE MATERIALIZED VIEW tenant_usage_daily AS
SELECT 
  tenant_id,
  DATE(start_time) as usage_date,
  COUNT(*) as total_calls,
  SUM(duration_seconds) as total_duration,
  SUM(cost_usd) as total_cost,
  COUNT(DISTINCT agent_id) as active_agents
FROM calls
WHERE start_time > NOW() - INTERVAL '30 days'
GROUP BY tenant_id, DATE(start_time);

CREATE UNIQUE INDEX ON tenant_usage_daily(tenant_id, usage_date);
```

### Security at Scale

#### Customer Credential Storage (AWS KMS)
```elixir
defmodule Aybiza.Security.CredentialVault do
  @kms_client ExAws.KMS
  
  def store_customer_credential(tenant_id, credential_type, credential_data) do
    # Generate tenant-specific data key
    {:ok, %{ciphertext_blob: data_key_encrypted, plaintext: data_key}} = 
      @kms_client.generate_data_key(
        key_id: "alias/aybiza-tenant-#{tenant_id}",
        key_spec: "AES_256"
      )
      |> ExAws.request()
    
    # Encrypt credential with data key
    encrypted_credential = :crypto.crypto_one_time(
      :aes_256_gcm, 
      data_key, 
      generate_iv(), 
      credential_data, 
      true
    )
    
    # Store in database
    %CustomerCredential{
      tenant_id: tenant_id,
      credential_type: credential_type,
      encrypted_data: Base.encode64(encrypted_credential),
      data_key_encrypted: Base.encode64(data_key_encrypted),
      encryption_context: %{
        "tenant_id" => tenant_id,
        "timestamp" => DateTime.utc_now()
      }
    }
    |> Repo.insert()
  end
end
```

### Schema: billing

#### Table: billing_accounts
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `account_number`: VARCHAR(20) UNIQUE NOT NULL -- Format: BILL-XXXX-XXXX
- `billing_type`: VARCHAR(50) DEFAULT 'postpaid' CHECK (billing_type IN ('prepaid', 'postpaid', 'enterprise_contract'))
- `payment_terms`: VARCHAR(50) DEFAULT 'net_30' CHECK (payment_terms IN ('due_on_receipt', 'net_15', 'net_30', 'net_60', 'custom'))
- `credit_limit`: DECIMAL(10,2) DEFAULT 10000.00
- `current_balance`: DECIMAL(10,2) DEFAULT 0.00
- `auto_recharge_enabled`: BOOLEAN DEFAULT FALSE
- `auto_recharge_amount`: DECIMAL(10,2)
- `auto_recharge_threshold`: DECIMAL(10,2)
- `billing_contact`: JSONB NOT NULL -- Name, email, phone
- `billing_address`: JSONB NOT NULL
- `tax_exempt`: BOOLEAN DEFAULT FALSE
- `tax_exempt_certificate`: TEXT -- Encrypted
- `currency`: VARCHAR(3) DEFAULT 'USD'
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'delinquent', 'closed'))
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()

#### Table: payment_methods
- `id`: UUID (PK)
- `billing_account_id`: UUID NOT NULL REFERENCES billing_accounts(id) ON DELETE CASCADE
- `type`: VARCHAR(50) NOT NULL CHECK (type IN ('credit_card', 'ach', 'wire_transfer', 'invoice'))
- `is_primary`: BOOLEAN DEFAULT FALSE
- `last_four`: VARCHAR(4) -- Last 4 digits of card/account
- `brand`: VARCHAR(50) -- Visa, Mastercard, etc.
- `expiry_month`: INTEGER
- `expiry_year`: INTEGER
- `bank_name`: VARCHAR(100)
- `account_holder_name`: VARCHAR(255) -- Encrypted
- `billing_address`: JSONB
- `stripe_payment_method_id`: VARCHAR(255) -- External provider reference
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'expired', 'failed', 'removed'))
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()

#### Table: usage_metering (TimescaleDB Hypertable)
- `id`: UUID (PK)
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `organization_id`: UUID NOT NULL
- `tenant_id`: UUID NOT NULL
- `resource_type`: VARCHAR(100) NOT NULL -- 'call_minutes', 'agent_sessions', 'api_calls', 'storage_gb', 'tool_executions'
- `resource_subtype`: VARCHAR(100) -- 'inbound', 'outbound', 'claude_opus', 'claude_sonnet'
- `quantity`: DECIMAL(20,6) NOT NULL
- `unit`: VARCHAR(50) NOT NULL -- 'minutes', 'sessions', 'calls', 'gb', 'executions'
- `unit_price`: DECIMAL(10,6) NOT NULL
- `total_cost`: DECIMAL(10,6) NOT NULL
- `metadata`: JSONB DEFAULT '{}'
- `billing_period`: DATE NOT NULL
- `invoice_id`: UUID REFERENCES invoices(id)

#### Table: invoices
- `id`: UUID (PK)
- `invoice_number`: VARCHAR(50) UNIQUE NOT NULL -- Format: INV-YYYY-MM-XXXXXX
- `billing_account_id`: UUID NOT NULL REFERENCES billing_accounts(id)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id)
- `billing_period_start`: DATE NOT NULL
- `billing_period_end`: DATE NOT NULL
- `status`: VARCHAR(50) DEFAULT 'draft' CHECK (status IN ('draft', 'pending', 'sent', 'paid', 'overdue', 'cancelled', 'refunded'))
- `subtotal`: DECIMAL(10,2) NOT NULL
- `tax_amount`: DECIMAL(10,2) DEFAULT 0.00
- `discount_amount`: DECIMAL(10,2) DEFAULT 0.00
- `total_amount`: DECIMAL(10,2) NOT NULL
- `currency`: VARCHAR(3) DEFAULT 'USD'
- `due_date`: DATE NOT NULL
- `paid_date`: DATE
- `payment_method_id`: UUID REFERENCES payment_methods(id)
- `stripe_invoice_id`: VARCHAR(255)
- `pdf_url`: VARCHAR(500) -- S3 URL
- `notes`: TEXT
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `sent_at`: TIMESTAMP(6) WITH TIME ZONE
- `metadata`: JSONB DEFAULT '{}'

### Schema: identity (Enhanced SSO & Sessions)

#### Table: sso_configurations
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `provider`: VARCHAR(50) NOT NULL CHECK (provider IN ('saml', 'oidc', 'azure_ad', 'okta', 'google_workspace', 'custom'))
- `enabled`: BOOLEAN DEFAULT TRUE
- `idp_entity_id`: VARCHAR(500) NOT NULL
- `sso_url`: VARCHAR(500) NOT NULL
- `certificate`: TEXT -- X.509 certificate for SAML
- `client_id`: VARCHAR(255) -- For OIDC
- `client_secret`: TEXT -- Encrypted, for OIDC
- `metadata_url`: VARCHAR(500)
- `attribute_mapping`: JSONB NOT NULL DEFAULT '{}'
- `default_role_id`: UUID REFERENCES roles(id)
- `auto_provision_users`: BOOLEAN DEFAULT TRUE
- `jit_provisioning`: BOOLEAN DEFAULT TRUE
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'testing'))
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()

#### Table: user_sessions
- `id`: UUID (PK)
- `user_id`: UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
- `session_token`: VARCHAR(255) UNIQUE NOT NULL -- Hashed
- `refresh_token`: VARCHAR(255) UNIQUE -- Hashed
- `ip_address`: INET NOT NULL
- `user_agent`: TEXT
- `device_fingerprint`: VARCHAR(255)
- `location`: JSONB -- GeoIP data
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `expires_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL
- `last_activity`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `revoked_at`: TIMESTAMP(6) WITH TIME ZONE
- `revoked_reason`: VARCHAR(100)

#### Table: login_history (TimescaleDB Hypertable)
- `id`: UUID (PK)
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `user_id`: UUID NOT NULL REFERENCES users(id)
- `success`: BOOLEAN NOT NULL
- `failure_reason`: VARCHAR(100) -- 'invalid_password', 'account_locked', 'mfa_failed'
- `ip_address`: INET NOT NULL
- `user_agent`: TEXT
- `device_fingerprint`: VARCHAR(255)
- `location`: JSONB
- `authentication_method`: VARCHAR(50) -- 'password', 'sso', 'api_key', 'mfa'
- `risk_score`: DECIMAL(3,2)
- `suspicious_indicators`: JSONB DEFAULT '[]'

### Schema: compliance (Enhanced KYC & Business Verification)

#### Table: kyc_verifications
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `verification_type`: VARCHAR(50) NOT NULL CHECK (verification_type IN ('business', 'individual', 'enhanced'))
- `status`: VARCHAR(50) DEFAULT 'pending' CHECK (status IN ('pending', 'in_review', 'approved', 'rejected', 'expired'))
- `risk_level`: VARCHAR(20) CHECK (risk_level IN ('low', 'medium', 'high', 'unacceptable'))
- `submitted_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `reviewed_at`: TIMESTAMP(6) WITH TIME ZONE
- `reviewed_by`: VARCHAR(255) -- Can be system or human reviewer
- `expiry_date`: DATE
- `rejection_reasons`: JSONB DEFAULT '[]'
- `verification_data`: JSONB NOT NULL -- Encrypted sensitive data
- `provider`: VARCHAR(50) -- 'jumio', 'onfido', 'trulioo', 'manual'
- `provider_reference`: VARCHAR(255)
- `documents`: JSONB DEFAULT '[]' -- Array of document references
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()

#### Table: compliance_documents
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `document_type`: VARCHAR(100) NOT NULL -- 'business_license', 'tax_certificate', 'incorporation', 'bank_statement'
- `document_name`: VARCHAR(255) NOT NULL
- `file_key`: VARCHAR(500) NOT NULL -- S3 key
- `file_hash`: VARCHAR(64) NOT NULL -- SHA-256
- `mime_type`: VARCHAR(100) NOT NULL
- `file_size_bytes`: BIGINT NOT NULL
- `status`: VARCHAR(50) DEFAULT 'pending' CHECK (status IN ('pending', 'verified', 'rejected', 'expired'))
- `expiry_date`: DATE
- `verified_at`: TIMESTAMP(6) WITH TIME ZONE
- `verified_by`: UUID REFERENCES users(id)
- `metadata`: JSONB DEFAULT '{}'
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()

#### Table: sanctions_screening
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id)
- `screening_type`: VARCHAR(50) NOT NULL CHECK (screening_type IN ('pep', 'sanctions', 'adverse_media', 'watchlist'))
- `status`: VARCHAR(50) NOT NULL CHECK (status IN ('clear', 'potential_match', 'confirmed_match', 'false_positive'))
- `provider`: VARCHAR(50) NOT NULL -- 'dow_jones', 'refinitiv', 'complyadvantage'
- `provider_reference`: VARCHAR(255)
- `matches`: JSONB DEFAULT '[]'
- `risk_score`: DECIMAL(3,2)
- `screened_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `reviewed_at`: TIMESTAMP(6) WITH TIME ZONE
- `reviewed_by`: UUID REFERENCES users(id)
- `review_notes`: TEXT
- `next_screening_date`: DATE

### Schema: quality (Call Quality & Analytics)

#### Table: call_quality_scores
- `id`: UUID (PK)
- `call_id`: UUID NOT NULL REFERENCES calls(id) ON DELETE CASCADE
- `tenant_id`: UUID NOT NULL
- `agent_id`: UUID NOT NULL
- `overall_score`: DECIMAL(3,2) NOT NULL -- 0.00 to 1.00
- `audio_quality_score`: DECIMAL(3,2)
- `agent_performance_score`: DECIMAL(3,2)
- `customer_satisfaction_score`: DECIMAL(3,2)
- `compliance_score`: DECIMAL(3,2)
- `scoring_method`: VARCHAR(50) DEFAULT 'ai' CHECK (scoring_method IN ('ai', 'manual', 'hybrid'))
- `scoring_model_version`: VARCHAR(50)
- `issues_detected`: JSONB DEFAULT '[]'
- `highlights`: JSONB DEFAULT '[]'
- `recommendations`: JSONB DEFAULT '[]'
- `scored_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `reviewed_by`: UUID REFERENCES users(id)
- `reviewed_at`: TIMESTAMP(6) WITH TIME ZONE
- `review_notes`: TEXT

#### Table: agent_performance_metrics (TimescaleDB Hypertable)
- `id`: UUID (PK)
- `timestamp`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `agent_id`: UUID NOT NULL REFERENCES agents(id)
- `tenant_id`: UUID NOT NULL
- `metric_date`: DATE NOT NULL
- `total_calls`: INTEGER DEFAULT 0
- `successful_calls`: INTEGER DEFAULT 0
- `average_call_duration`: DECIMAL(10,2)
- `average_response_time_ms`: DECIMAL(10,2)
- `customer_satisfaction_avg`: DECIMAL(3,2)
- `first_call_resolution_rate`: DECIMAL(5,2)
- `tool_usage_stats`: JSONB DEFAULT '{}'
- `error_rate`: DECIMAL(5,2)
- `escalation_rate`: DECIMAL(5,2)
- `compliance_violations`: INTEGER DEFAULT 0
- UNIQUE (agent_id, metric_date)

#### Table: customer_feedback
- `id`: UUID (PK)
- `call_id`: UUID NOT NULL REFERENCES calls(id)
- `tenant_id`: UUID NOT NULL
- `feedback_type`: VARCHAR(50) NOT NULL CHECK (feedback_type IN ('survey', 'rating', 'comment', 'complaint'))
- `rating`: INTEGER CHECK (rating >= 1 AND rating <= 5)
- `nps_score`: INTEGER CHECK (nps_score >= 0 AND nps_score <= 10)
- `comment`: TEXT
- `categories`: JSONB DEFAULT '[]' -- ['helpful', 'fast', 'accurate', 'friendly']
- `sentiment`: VARCHAR(20) CHECK (sentiment IN ('positive', 'neutral', 'negative'))
- `collected_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `collection_method`: VARCHAR(50) -- 'post_call_ivr', 'sms', 'email', 'in_call'
- `metadata`: JSONB DEFAULT '{}'

### Schema: features (Feature Management & Configuration)

#### Table: feature_flags
- `id`: UUID (PK)
- `name`: VARCHAR(100) UNIQUE NOT NULL
- `description`: TEXT
- `flag_type`: VARCHAR(50) DEFAULT 'release' CHECK (flag_type IN ('release', 'experiment', 'operational', 'permission'))
- `default_enabled`: BOOLEAN DEFAULT FALSE
- `rollout_percentage`: DECIMAL(5,2) DEFAULT 0.00
- `targeting_rules`: JSONB DEFAULT '{}'
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by`: UUID REFERENCES users(id)

#### Table: tenant_feature_overrides
- `id`: UUID (PK)
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
- `feature_flag_id`: UUID NOT NULL REFERENCES feature_flags(id)
- `enabled`: BOOLEAN NOT NULL
- `configuration`: JSONB DEFAULT '{}'
- `reason`: TEXT
- `expires_at`: TIMESTAMP(6) WITH TIME ZONE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by`: UUID REFERENCES users(id)
- UNIQUE (tenant_id, feature_flag_id)

#### Table: white_label_configurations
- `id`: UUID (PK)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
- `brand_name`: VARCHAR(255) NOT NULL
- `logo_url`: VARCHAR(500)
- `favicon_url`: VARCHAR(500)
- `primary_color`: VARCHAR(7) -- Hex color
- `secondary_color`: VARCHAR(7)
- `custom_domain`: VARCHAR(255) UNIQUE
- `ssl_certificate_id`: VARCHAR(255) -- AWS ACM certificate
- `email_from_name`: VARCHAR(255)
- `email_from_address`: VARCHAR(255)
- `support_email`: VARCHAR(255)
- `support_phone`: VARCHAR(20)
- `custom_css`: TEXT
- `custom_js`: TEXT
- `metadata`: JSONB DEFAULT '{}'
- `status`: VARCHAR(50) DEFAULT 'active'
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()

### Schema: operations (Operational Excellence)

#### Table: scheduled_maintenance
- `id`: UUID (PK)
- `title`: VARCHAR(255) NOT NULL
- `description`: TEXT
- `maintenance_type`: VARCHAR(50) NOT NULL CHECK (maintenance_type IN ('planned', 'emergency', 'security_patch'))
- `affected_services`: JSONB NOT NULL DEFAULT '[]'
- `affected_regions`: JSONB NOT NULL DEFAULT '[]'
- `affected_tenants`: JSONB DEFAULT '[]' -- Empty means all tenants
- `start_time`: TIMESTAMP(6) WITH TIME ZONE NOT NULL
- `end_time`: TIMESTAMP(6) WITH TIME ZONE NOT NULL
- `status`: VARCHAR(50) DEFAULT 'scheduled' CHECK (status IN ('scheduled', 'in_progress', 'completed', 'cancelled'))
- `notification_sent`: BOOLEAN DEFAULT FALSE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by`: UUID REFERENCES users(id)
- `completed_at`: TIMESTAMP(6) WITH TIME ZONE
- `impact_summary`: TEXT

#### Table: rate_limit_configurations
- `id`: UUID (PK)
- `scope`: VARCHAR(50) NOT NULL CHECK (scope IN ('global', 'organization', 'tenant', 'user'))
- `scope_id`: UUID -- NULL for global
- `resource`: VARCHAR(100) NOT NULL -- 'api_calls', 'concurrent_calls', 'tool_executions'
- `limit_value`: INTEGER NOT NULL
- `time_window_seconds`: INTEGER NOT NULL
- `burst_allowed`: BOOLEAN DEFAULT TRUE
- `burst_multiplier`: DECIMAL(3,2) DEFAULT 1.5
- `enforcement_action`: VARCHAR(50) DEFAULT 'throttle' CHECK (enforcement_action IN ('throttle', 'reject', 'queue'))
- `alert_threshold_percentage`: DECIMAL(5,2) DEFAULT 80.00
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()

#### Table: sla_tracking
- `id`: UUID (PK)
- `tenant_id`: UUID NOT NULL REFERENCES tenants(id)
- `month`: DATE NOT NULL
- `uptime_percentage`: DECIMAL(5,2) NOT NULL
- `api_availability_percentage`: DECIMAL(5,2) NOT NULL
- `voice_quality_percentage`: DECIMAL(5,2) NOT NULL
- `support_response_time_avg_minutes`: INTEGER
- `incidents_count`: INTEGER DEFAULT 0
- `sla_breaches`: JSONB DEFAULT '[]'
- `credits_issued`: DECIMAL(10,2) DEFAULT 0.00
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- UNIQUE (tenant_id, month)

### Schema: partners (Partner & Reseller Ecosystem)

#### Table: partners
- `id`: UUID (PK)
- `partner_code`: VARCHAR(20) UNIQUE NOT NULL -- Format: PTR-XXXX-XXXX
- `company_name`: VARCHAR(255) NOT NULL
- `partner_type`: VARCHAR(50) NOT NULL CHECK (partner_type IN ('reseller', 'referral', 'technology', 'integration'))
- `tier`: VARCHAR(50) DEFAULT 'bronze' CHECK (tier IN ('bronze', 'silver', 'gold', 'platinum'))
- `commission_rate`: DECIMAL(5,2) DEFAULT 20.00
- `discount_rate`: DECIMAL(5,2) DEFAULT 0.00
- `billing_model`: VARCHAR(50) DEFAULT 'revenue_share' CHECK (billing_model IN ('revenue_share', 'fixed_fee', 'tiered', 'custom'))
- `contract_start`: DATE NOT NULL
- `contract_end`: DATE
- `status`: VARCHAR(50) DEFAULT 'active' CHECK (status IN ('pending', 'active', 'suspended', 'terminated'))
- `primary_contact`: JSONB NOT NULL
- `technical_contact`: JSONB
- `billing_contact`: JSONB
- `api_access_enabled`: BOOLEAN DEFAULT TRUE
- `white_label_enabled`: BOOLEAN DEFAULT FALSE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()

#### Table: partner_organizations
- `id`: UUID (PK)
- `partner_id`: UUID NOT NULL REFERENCES partners(id)
- `organization_id`: UUID NOT NULL REFERENCES organizations(id)
- `relationship_type`: VARCHAR(50) DEFAULT 'managed' CHECK (relationship_type IN ('managed', 'referred', 'supported'))
- `commission_override`: DECIMAL(5,2) -- Override partner default
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `metadata`: JSONB DEFAULT '{}'
- UNIQUE (partner_id, organization_id)

### Schema: security (Enhanced Threat Intelligence)

#### Table: threat_intelligence
- `id`: UUID (PK)
- `threat_type`: VARCHAR(100) NOT NULL -- 'voice_cloning', 'number_spoofing', 'prompt_injection'
- `threat_signature`: TEXT NOT NULL
- `severity`: VARCHAR(20) NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical'))
- `detection_method`: VARCHAR(100) NOT NULL
- `active`: BOOLEAN DEFAULT TRUE
- `false_positive_rate`: DECIMAL(5,2)
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `source`: VARCHAR(100) -- 'internal', 'vendor', 'community'
- `metadata`: JSONB DEFAULT '{}'

#### Table: security_policies
- `id`: UUID (PK)
- `organization_id`: UUID REFERENCES organizations(id) -- NULL for global policies
- `tenant_id`: UUID REFERENCES tenants(id) -- NULL for org-wide policies
- `policy_type`: VARCHAR(100) NOT NULL -- 'password', 'mfa', 'api_access', 'data_retention'
- `policy_name`: VARCHAR(255) NOT NULL
- `requirements`: JSONB NOT NULL
- `enforcement_level`: VARCHAR(50) DEFAULT 'required' CHECK (enforcement_level IN ('optional', 'recommended', 'required', 'strict'))
- `grace_period_days`: INTEGER DEFAULT 0
- `exceptions`: JSONB DEFAULT '[]'
- `active`: BOOLEAN DEFAULT TRUE
- `created_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `updated_at`: TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT NOW()
- `created_by`: UUID REFERENCES users(id)

## Additional Indexes for Enterprise Tables

### Billing & Financial Indexes
```sql
-- Billing Accounts
CREATE INDEX CONCURRENTLY idx_billing_accounts_org ON billing_accounts(organization_id, status);
CREATE INDEX CONCURRENTLY idx_billing_accounts_balance ON billing_accounts(current_balance, credit_limit);

-- Payment Methods
CREATE INDEX CONCURRENTLY idx_payment_methods_account ON payment_methods(billing_account_id, is_primary);
CREATE INDEX CONCURRENTLY idx_payment_methods_expiry ON payment_methods(expiry_year, expiry_month, status);

-- Usage Metering (TimescaleDB optimized)
CREATE INDEX CONCURRENTLY idx_usage_metering_tenant_time ON usage_metering(tenant_id, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_usage_metering_resource ON usage_metering(resource_type, resource_subtype, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_usage_metering_billing ON usage_metering(billing_period, invoice_id);

-- Invoices
CREATE INDEX CONCURRENTLY idx_invoices_org_period ON invoices(organization_id, billing_period_start DESC);
CREATE INDEX CONCURRENTLY idx_invoices_status_due ON invoices(status, due_date) WHERE status IN ('pending', 'sent', 'overdue');
```

### Identity & Security Indexes
```sql
-- SSO Configurations
CREATE INDEX CONCURRENTLY idx_sso_configs_org ON sso_configurations(organization_id, enabled);

-- User Sessions
CREATE INDEX CONCURRENTLY idx_user_sessions_user ON user_sessions(user_id, expires_at);
CREATE INDEX CONCURRENTLY idx_user_sessions_activity ON user_sessions(last_activity) WHERE revoked_at IS NULL;

-- Login History (TimescaleDB optimized)
CREATE INDEX CONCURRENTLY idx_login_history_user_time ON login_history(user_id, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_login_history_failures ON login_history(timestamp DESC) WHERE success = FALSE;
CREATE INDEX CONCURRENTLY idx_login_history_risk ON login_history(risk_score DESC, timestamp DESC) WHERE risk_score > 0.7;
```

### Quality & Analytics Indexes
```sql
-- Call Quality Scores
CREATE INDEX CONCURRENTLY idx_quality_scores_call ON call_quality_scores(call_id);
CREATE INDEX CONCURRENTLY idx_quality_scores_agent ON call_quality_scores(agent_id, scored_at DESC);
CREATE INDEX CONCURRENTLY idx_quality_scores_low ON call_quality_scores(overall_score, scored_at DESC) WHERE overall_score < 0.7;

-- Agent Performance Metrics (TimescaleDB optimized)
CREATE INDEX CONCURRENTLY idx_agent_metrics_time ON agent_performance_metrics(agent_id, timestamp DESC);
CREATE INDEX CONCURRENTLY idx_agent_metrics_date ON agent_performance_metrics(agent_id, metric_date DESC);

-- Customer Feedback
CREATE INDEX CONCURRENTLY idx_feedback_call ON customer_feedback(call_id);
CREATE INDEX CONCURRENTLY idx_feedback_sentiment ON customer_feedback(sentiment, collected_at DESC);
CREATE INDEX CONCURRENTLY idx_feedback_nps ON customer_feedback(nps_score, collected_at DESC) WHERE nps_score IS NOT NULL;
```

### Feature Management Indexes
```sql
-- Feature Flags
CREATE INDEX CONCURRENTLY idx_feature_flags_type ON feature_flags(flag_type, default_enabled);

-- Tenant Feature Overrides
CREATE INDEX CONCURRENTLY idx_feature_overrides_tenant ON tenant_feature_overrides(tenant_id);
CREATE INDEX CONCURRENTLY idx_feature_overrides_expires ON tenant_feature_overrides(expires_at) WHERE expires_at IS NOT NULL;
```

### Partner Management Indexes
```sql
-- Partners
CREATE INDEX CONCURRENTLY idx_partners_type_status ON partners(partner_type, status);
CREATE INDEX CONCURRENTLY idx_partners_tier ON partners(tier, status) WHERE status = 'active';

-- Partner Organizations
CREATE INDEX CONCURRENTLY idx_partner_orgs_partner ON partner_organizations(partner_id);
CREATE INDEX CONCURRENTLY idx_partner_orgs_org ON partner_organizations(organization_id);
```

## TimescaleDB Configuration for New Tables

### New Hypertables
```sql
-- Create hypertables for time-series data
SELECT create_hypertable('usage_metering', 'timestamp', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('login_history', 'timestamp', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('agent_performance_metrics', 'timestamp', chunk_time_interval => INTERVAL '1 day');

-- Add space dimensions for better performance
SELECT add_dimension('usage_metering', 'tenant_id', number_partitions => 4);
SELECT add_dimension('login_history', 'user_id', number_partitions => 4);
SELECT add_dimension('agent_performance_metrics', 'tenant_id', number_partitions => 4);
```

### Continuous Aggregates for Enterprise Metrics
```sql
-- Hourly usage aggregations
CREATE MATERIALIZED VIEW usage_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', timestamp) AS hour,
       tenant_id,
       resource_type,
       SUM(quantity) AS total_quantity,
       SUM(total_cost) AS total_cost,
       COUNT(*) AS transaction_count
FROM usage_metering
GROUP BY hour, tenant_id, resource_type;

-- Daily billing aggregations
CREATE MATERIALIZED VIEW billing_daily
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 day', timestamp) AS day,
       organization_id,
       SUM(total_cost) AS daily_cost,
       COUNT(DISTINCT tenant_id) AS active_tenants,
       COUNT(DISTINCT resource_type) AS resource_types_used
FROM usage_metering
GROUP BY day, organization_id;
```

## Citus Distribution for Enterprise Tables
```sql
-- Distribute enterprise tables by appropriate keys
SELECT create_distributed_table('billing_accounts', 'organization_id');
SELECT create_distributed_table('payment_methods', 'billing_account_id', colocate_with => 'billing_accounts');
SELECT create_distributed_table('invoices', 'organization_id');
SELECT create_distributed_table('kyc_verifications', 'organization_id');
SELECT create_distributed_table('compliance_documents', 'organization_id');
SELECT create_distributed_table('sso_configurations', 'organization_id');
SELECT create_distributed_table('user_sessions', 'user_id', colocate_with => 'users');
SELECT create_distributed_table('call_quality_scores', 'tenant_id', colocate_with => 'calls');
SELECT create_distributed_table('customer_feedback', 'tenant_id', colocate_with => 'calls');
SELECT create_distributed_table('feature_flags', 'id'); -- Reference table candidate
SELECT create_distributed_table('tenant_feature_overrides', 'tenant_id');
SELECT create_distributed_table('white_label_configurations', 'organization_id');
SELECT create_distributed_table('rate_limit_configurations', 'scope_id');
SELECT create_distributed_table('sla_tracking', 'tenant_id');

-- Reference tables (replicated to all nodes)
SELECT create_reference_table('partners');
SELECT create_reference_table('threat_intelligence');
SELECT create_reference_table('security_policies');
```

This enhanced database schema now includes:
- **Billion-scale operations** with Citus horizontal sharding
- **Claude 4 agent capabilities** including extended thinking, parallel tools, MCP, and Files API
- **Hybrid storage** with PostgreSQL, DynamoDB, TimescaleDB, and Vector DB
- **Comprehensive billing** with usage metering and invoicing
- **Enterprise identity** with SSO and session management
- **KYC/compliance** for business verification
- **Quality management** with AI-powered scoring
- **Feature flags** for controlled rollouts
- **Partner ecosystem** support
- **Advanced security** with threat intelligence
- **Account/User ID system** for easy support verification
- **Comprehensive consent management** for outbound calls
- **Secure credential storage** with tenant isolation
- **Indefinite conversation history** with intelligent tiering
- **Cost optimization** through caching and data lifecycle management