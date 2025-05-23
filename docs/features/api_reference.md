# AYBIZA API Reference

## Overview
This document provides comprehensive API reference for the AYBIZA AI voice agent platform, covering REST endpoints, WebSocket connections, and webhook integrations.

## Base URLs

### Production
- **API Base**: `https://api.aybiza.com`
- **WebSocket**: `wss://api.aybiza.com/ws`
- **Webhooks**: `https://webhooks.aybiza.com`

### Staging
- **API Base**: `https://staging-api.aybiza.com`
- **WebSocket**: `wss://staging-api.aybiza.com/ws`
- **Webhooks**: `https://staging-webhooks.aybiza.com`

## Authentication

### API Key Authentication
```http
Authorization: Bearer <your-api-key>
Content-Type: application/json
```

### JWT Token Authentication
```http
Authorization: Bearer <jwt-token>
Content-Type: application/json
```

### Webhook Signature Verification
```http
X-AYBIZA-Signature: sha256=<signature>
X-AYBIZA-Timestamp: <unix-timestamp>
```

## Core APIs

### 1. Agent Management API

#### Create Agent
```http
POST /api/v1/agents
```

**Request Body:**
```json
{
  "name": "Customer Support Agent",
  "description": "Handles customer inquiries and support requests",
  "system_prompt": "You are a helpful customer support agent for ACME Corp...",
  "voice": {
    "model": "aura-asteria-en",
    "style": "calm_informative",
    "natural_speech": true,
    "emotion_control": "moderate"
  },
  "language": {
    "primary": "en-US",
    "auto_detect": true,
    "supported": ["en-US", "es-ES", "fr-FR"]
  },
  "models": {
    "default": "claude-3-5-sonnet-v2",
    "fast": "claude-3-5-haiku",
    "complex": "claude-4-opus",
    "advanced": "claude-4-sonnet"
  },
  "parameters": {
    "max_response_time": 1500,
    "interruption_threshold": 0.7,
    "context_window": 32000,
    "temperature": 0.3
  },
  "automation": {
    "tools_enabled": true,
    "max_tool_calls": 5,
    "async_workflows": true
  }
}
```

**Response:**
```json
{
  "id": "agent_123456",
  "name": "Customer Support Agent",
  "status": "active",
  "version": "1.0.0",
  "created_at": "2025-01-15T10:30:00Z",
  "updated_at": "2025-01-15T10:30:00Z",
  "phone_numbers": [],
  "stats": {
    "total_calls": 0,
    "average_duration": 0,
    "satisfaction_score": 0
  }
}
```

#### Get Agent
```http
GET /api/v1/agents/{agent_id}
```

**Response:**
```json
{
  "id": "agent_123456",
  "name": "Customer Support Agent",
  "description": "Handles customer inquiries and support requests",
  "status": "active",
  "configuration": {
    "system_prompt": "You are a helpful customer support agent...",
    "voice": {
      "model": "aura-asteria-en",
      "style": "calm_informative"
    },
    "models": {
      "default": "claude-3-5-sonnet-v2",
      "complex": "claude-4-opus"
    }
  },
  "phone_numbers": [
    {
      "id": "phone_789012",
      "number": "+1234567890",
      "status": "active",
      "region": "us-east-1"
    }
  ],
  "created_at": "2025-01-15T10:30:00Z",
  "updated_at": "2025-01-15T10:30:00Z"
}
```

#### Update Agent
```http
PUT /api/v1/agents/{agent_id}
```

#### Delete Agent
```http
DELETE /api/v1/agents/{agent_id}
```

#### List Agents
```http
GET /api/v1/agents?limit=20&offset=0&status=active
```

### 2. Phone Number Management API

#### Acquire Phone Number
```http
POST /api/v1/phone-numbers
```

**Request Body:**
```json
{
  "type": "managed", // "managed", "sip_trunk", "twilio"
  "region": "us-east-1",
  "area_code": "415",
  "agent_id": "agent_123456",
  "configuration": {
    "call_routing": "direct",
    "fallback_agent": "agent_backup",
    "business_hours": {
      "timezone": "America/New_York",
      "schedule": {
        "monday": {"start": "09:00", "end": "17:00"},
        "tuesday": {"start": "09:00", "end": "17:00"},
        "wednesday": {"start": "09:00", "end": "17:00"},
        "thursday": {"start": "09:00", "end": "17:00"},
        "friday": {"start": "09:00", "end": "17:00"}
      }
    }
  }
}
```

**Response:**
```json
{
  "id": "phone_789012",
  "number": "+14155551234",
  "type": "managed",
  "status": "provisioning",
  "region": "us-east-1",
  "agent_id": "agent_123456",
  "webhook_url": "https://webhooks.aybiza.com/voice/inbound",
  "created_at": "2025-01-15T11:00:00Z",
  "estimated_activation": "2025-01-15T11:30:00Z"
}
```

#### Get Phone Number
```http
GET /api/v1/phone-numbers/{phone_id}
```

#### Update Phone Number Configuration
```http
PUT /api/v1/phone-numbers/{phone_id}
```

#### Release Phone Number
```http
DELETE /api/v1/phone-numbers/{phone_id}
```

#### List Phone Numbers
```http
GET /api/v1/phone-numbers?agent_id=agent_123456&status=active
```

### 3. Call Management API

#### Get Call Details
```http
GET /api/v1/calls/{call_id}
```

**Response:**
```json
{
  "id": "call_456789",
  "call_sid": "twilio_call_sid_123",
  "agent_id": "agent_123456",
  "phone_number": "+14155551234",
  "caller_number": "+19876543210",
  "status": "completed",
  "direction": "inbound",
  "started_at": "2025-01-15T14:30:00Z",
  "ended_at": "2025-01-15T14:33:45Z",
  "duration": 225,
  "transcript": [
    {
      "speaker": "caller",
      "text": "Hello, I need help with my account",
      "timestamp": "2025-01-15T14:30:15Z"
    },
    {
      "speaker": "agent",
      "text": "Hello! I'd be happy to help you with your account. Could you please provide your account number?",
      "timestamp": "2025-01-15T14:30:18Z"
    }
  ],
  "analytics": {
    "sentiment_score": 0.8,
    "satisfaction_score": 4.2,
    "resolution_status": "resolved",
    "agent_performance": 4.5,
    "call_quality": 4.8
  },
  "cost": {
    "voice_minutes": 3.75,
    "stt_processing": 0.12,
    "llm_tokens": 1847,
    "tts_characters": 456,
    "total_cost": 0.18
  }
}
```

#### List Calls
```http
GET /api/v1/calls?agent_id=agent_123456&status=completed&limit=50&offset=0
```

#### Get Call Recording
```http
GET /api/v1/calls/{call_id}/recording
```

#### Get Call Analytics
```http
GET /api/v1/calls/{call_id}/analytics
```

### 4. Real-time Call API

#### Initiate Outbound Call
```http
POST /api/v1/calls/outbound
```

**Request Body:**
```json
{
  "agent_id": "agent_123456",
  "to_number": "+19876543210",
  "from_number": "+14155551234",
  "context": {
    "customer_id": "cust_789",
    "call_reason": "follow_up",
    "previous_interactions": ["call_456789"]
  },
  "options": {
    "record_call": true,
    "enable_transcription": true,
    "max_duration": 1800
  }
}
```

### 5. Analytics API

#### Get Agent Performance
```http
GET /api/v1/analytics/agents/{agent_id}/performance?period=7d
```

**Response:**
```json
{
  "agent_id": "agent_123456",
  "period": "7d",
  "metrics": {
    "total_calls": 245,
    "total_duration": 18750,
    "average_duration": 76.5,
    "completion_rate": 0.96,
    "satisfaction_score": 4.3,
    "resolution_rate": 0.89
  },
  "trends": {
    "call_volume": [
      {"date": "2025-01-14", "calls": 38},
      {"date": "2025-01-15", "calls": 42},
      {"date": "2025-01-16", "calls": 35}
    ],
    "satisfaction": [
      {"date": "2025-01-14", "score": 4.2},
      {"date": "2025-01-15", "score": 4.4},
      {"date": "2025-01-16", "score": 4.1}
    ]
  }
}
```

#### Get System Analytics
```http
GET /api/v1/analytics/system?period=24h&metrics=latency,throughput,errors
```

### 6. Automation API

#### List Available Tools
```http
GET /api/v1/automation/tools
```

**Response:**
```json
{
  "tools": [
    {
      "id": "salesforce_lookup",
      "name": "Salesforce Customer Lookup",
      "description": "Look up customer information in Salesforce",
      "category": "crm",
      "parameters": {
        "customer_id": {"type": "string", "required": true},
        "fields": {"type": "array", "required": false}
      }
    },
    {
      "id": "send_email",
      "name": "Send Email",
      "description": "Send email to customer",
      "category": "communication",
      "parameters": {
        "to": {"type": "string", "required": true},
        "subject": {"type": "string", "required": true},
        "body": {"type": "string", "required": true}
      }
    }
  ]
}
```

#### Execute Tool
```http
POST /api/v1/automation/tools/{tool_id}/execute
```

**Request Body:**
```json
{
  "parameters": {
    "customer_id": "cust_789",
    "fields": ["name", "email", "account_status"]
  },
  "context": {
    "call_id": "call_456789",
    "agent_id": "agent_123456"
  }
}
```

## WebSocket API

### Voice Stream Connection
```javascript
// Connect to voice stream
const ws = new WebSocket('wss://api.aybiza.com/ws/voice/stream');

ws.onopen = function() {
  // Authenticate connection
  ws.send(JSON.stringify({
    type: 'auth',
    token: 'your-jwt-token',
    call_id: 'call_456789'
  }));
};

// Handle audio frames
ws.onmessage = function(event) {
  const message = JSON.parse(event.data);
  
  switch(message.type) {
    case 'audio_frame':
      // Handle incoming audio
      playAudioFrame(message.data);
      break;
      
    case 'transcript':
      // Handle transcription
      displayTranscript(message.text, message.speaker);
      break;
      
    case 'agent_response':
      // Handle agent text response
      displayAgentResponse(message.text);
      break;
  }
};

// Send audio frame
function sendAudioFrame(audioData) {
  ws.send(JSON.stringify({
    type: 'audio_frame',
    data: audioData,
    format: 'mulaw',
    sequence: frameSequence++
  }));
}
```

### Real-time Events
```javascript
// Connect to events stream
const eventsWs = new WebSocket('wss://api.aybiza.com/ws/events');

eventsWs.onmessage = function(event) {
  const eventData = JSON.parse(event.data);
  
  switch(eventData.type) {
    case 'call_started':
      console.log('Call started:', eventData.call_id);
      break;
      
    case 'call_ended':
      console.log('Call ended:', eventData.call_id, eventData.duration);
      break;
      
    case 'agent_status_changed':
      console.log('Agent status:', eventData.agent_id, eventData.status);
      break;
  }
};
```

## Webhook API

### Voice Call Events

#### Incoming Call Webhook
```http
POST /your-webhook-endpoint
Content-Type: application/json
X-AYBIZA-Signature: sha256=signature
X-AYBIZA-Event: call.incoming
```

**Payload:**
```json
{
  "event": "call.incoming",
  "call_id": "call_456789",
  "call_sid": "twilio_call_sid_123",
  "agent_id": "agent_123456",
  "phone_number": "+14155551234",
  "caller_number": "+19876543210",
  "timestamp": "2025-01-15T14:30:00Z",
  "metadata": {
    "caller_city": "San Francisco",
    "caller_state": "CA",
    "caller_country": "US"
  }
}
```

#### Call Completed Webhook
```json
{
  "event": "call.completed",
  "call_id": "call_456789",
  "call_sid": "twilio_call_sid_123",
  "agent_id": "agent_123456",
  "status": "completed",
  "duration": 225,
  "timestamp": "2025-01-15T14:33:45Z",
  "analytics": {
    "sentiment_score": 0.8,
    "satisfaction_score": 4.2,
    "resolution_status": "resolved"
  },
  "cost": {
    "total_cost": 0.18,
    "breakdown": {
      "voice_minutes": 0.075,
      "stt_processing": 0.012,
      "llm_tokens": 0.074,
      "tts_characters": 0.019
    }
  }
}
```

### Agent Events

#### Agent Status Change
```json
{
  "event": "agent.status_changed",
  "agent_id": "agent_123456",
  "old_status": "inactive",
  "new_status": "active",
  "timestamp": "2025-01-15T09:00:00Z"
}
```

### System Events

#### System Alert
```json
{
  "event": "system.alert",
  "alert_type": "performance_degradation",
  "severity": "warning",
  "message": "Voice processing latency elevated: 85ms average",
  "affected_components": ["voice_pipeline"],
  "timestamp": "2025-01-15T16:45:00Z"
}
```

## Billing & Payment APIs

### Get Billing Account
```http
GET /api/v1/billing/account
```

**Response:**
```json
{
  "account_number": "BILL-1234-5678",
  "billing_type": "postpaid",
  "payment_terms": "net_30",
  "credit_limit": 10000.00,
  "current_balance": 1250.50,
  "currency": "USD",
  "status": "active",
  "auto_recharge": {
    "enabled": false,
    "amount": 1000.00,
    "threshold": 100.00
  }
}
```

### List Invoices
```http
GET /api/v1/billing/invoices?page=1&limit=20&status=paid
```

**Response:**
```json
{
  "data": [
    {
      "invoice_number": "INV-2025-01-000123",
      "billing_period": {
        "start": "2025-01-01",
        "end": "2025-01-31"
      },
      "status": "paid",
      "amounts": {
        "subtotal": 2500.00,
        "tax": 225.00,
        "total": 2725.00
      },
      "due_date": "2025-02-15",
      "paid_date": "2025-02-10",
      "pdf_url": "https://invoices.aybiza.com/..."
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45
  }
}
```

### Get Current Usage
```http
GET /api/v1/billing/usage/current
```

**Response:**
```json
{
  "period": {
    "start": "2025-01-01",
    "end": "2025-01-23"
  },
  "usage": {
    "call_minutes": {
      "inbound": 12450,
      "outbound": 8320,
      "cost": 1567.70
    },
    "agent_sessions": {
      "count": 3420,
      "cost": 342.00
    },
    "api_calls": {
      "count": 145000,
      "cost": 72.50
    },
    "storage_gb": {
      "amount": 125.5,
      "cost": 12.55
    },
    "tool_executions": {
      "count": 8900,
      "cost": 89.00
    }
  },
  "estimated_total": 2083.75
}
```

### Add Payment Method
```http
POST /api/v1/billing/payment-methods
```

**Request Body:**
```json
{
  "type": "credit_card",
  "stripe_payment_method_id": "pm_1234567890",
  "is_primary": true
}
```

## Identity & SSO APIs

### Configure SSO
```http
POST /api/v1/identity/sso/configure
```

**Request Body:**
```json
{
  "provider": "okta",
  "idp_entity_id": "http://www.okta.com/...",
  "sso_url": "https://company.okta.com/app/...",
  "certificate": "-----BEGIN CERTIFICATE-----...",
  "attribute_mapping": {
    "email": "user.email",
    "name": "user.displayName",
    "department": "user.department"
  },
  "auto_provision_users": true,
  "default_role": "agent_user"
}
```

### Get SSO Configuration
```http
GET /api/v1/identity/sso/configuration
```

**Response:**
```json
{
  "provider": "okta",
  "enabled": true,
  "status": "active",
  "idp_entity_id": "http://www.okta.com/...",
  "sso_url": "https://company.okta.com/app/...",
  "metadata_url": "https://api.aybiza.com/sso/metadata",
  "auto_provision_users": true,
  "jit_provisioning": true,
  "statistics": {
    "total_logins": 1240,
    "active_users": 89,
    "last_login": "2025-01-23T10:30:00Z"
  }
}
```

### List Active Sessions
```http
GET /api/v1/identity/sessions
```

**Response:**
```json
{
  "sessions": [
    {
      "session_id": "sess_123456",
      "user_id": "USR-1234567890",
      "ip_address": "192.168.1.100",
      "user_agent": "Mozilla/5.0...",
      "location": {
        "city": "San Francisco",
        "region": "CA",
        "country": "US"
      },
      "created_at": "2025-01-23T08:00:00Z",
      "last_activity": "2025-01-23T10:45:00Z",
      "expires_at": "2025-01-24T08:00:00Z"
    }
  ]
}
```

## KYC & Compliance APIs

### Submit KYC Verification
```http
POST /api/v1/compliance/kyc/submit
```

**Request Body:**
```json
{
  "verification_type": "business",
  "business_info": {
    "legal_name": "ACME Corporation",
    "registration_number": "12345678",
    "tax_id": "98-7654321",
    "address": {
      "line1": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "postal_code": "94105",
      "country": "US"
    }
  },
  "documents": [
    {
      "type": "incorporation",
      "file_id": "file_123456"
    },
    {
      "type": "tax_certificate",
      "file_id": "file_789012"
    }
  ]
}
```

### Get KYC Status
```http
GET /api/v1/compliance/kyc/status
```

**Response:**
```json
{
  "verification_id": "kyc_123456",
  "status": "approved",
  "risk_level": "low",
  "verified_at": "2025-01-20T14:30:00Z",
  "expiry_date": "2026-01-20",
  "documents": [
    {
      "type": "incorporation",
      "status": "verified",
      "verified_at": "2025-01-20T14:25:00Z"
    }
  ]
}
```

### Check Sanctions
```http
POST /api/v1/compliance/sanctions/check
```

**Request Body:**
```json
{
  "entity_type": "business",
  "name": "ACME Corporation",
  "country": "US"
}
```

**Response:**
```json
{
  "status": "clear",
  "checked_at": "2025-01-23T10:00:00Z",
  "next_check_date": "2025-02-23",
  "matches": []
}
```

## Feature Management APIs

### List Feature Flags
```http
GET /api/v1/features/flags
```

**Response:**
```json
{
  "flags": [
    {
      "name": "claude_4_models",
      "description": "Enable Claude 4 Opus and Sonnet models",
      "enabled": true,
      "rollout_percentage": 100,
      "configuration": {
        "models": ["opus-4", "sonnet-4"]
      }
    },
    {
      "name": "advanced_analytics",
      "description": "Enhanced analytics dashboard",
      "enabled": true,
      "rollout_percentage": 50,
      "targeting_rules": {
        "tenant_tiers": ["enterprise"]
      }
    }
  ]
}
```

### Update Feature Flag
```http
PUT /api/v1/features/flags/claude_4_models
```

**Request Body:**
```json
{
  "enabled": true,
  "rollout_percentage": 75,
  "configuration": {
    "models": ["opus-4", "sonnet-4"],
    "extended_thinking": true
  }
}
```

## White Label APIs

### Configure White Label
```http
POST /api/v1/whitelabel/configure
```

**Request Body:**
```json
{
  "brand_name": "ACME Voice AI",
  "logo_url": "https://cdn.acme.com/logo.png",
  "favicon_url": "https://cdn.acme.com/favicon.ico",
  "colors": {
    "primary": "#1E40AF",
    "secondary": "#10B981"
  },
  "custom_domain": "voice.acme.com",
  "support": {
    "email": "support@acme.com",
    "phone": "+1-800-ACME-AI"
  }
}
```

## Quality & Analytics APIs

### Get Call Quality Score
```http
GET /api/v1/quality/calls/{call_id}/score
```

**Response:**
```json
{
  "call_id": "call_123456",
  "scores": {
    "overall": 0.92,
    "audio_quality": 0.95,
    "agent_performance": 0.89,
    "customer_satisfaction": 0.91,
    "compliance": 1.0
  },
  "issues_detected": [],
  "highlights": [
    "Excellent problem resolution",
    "Clear communication throughout"
  ],
  "scored_at": "2025-01-23T10:45:00Z"
}
```

### Get Agent Performance Metrics
```http
GET /api/v1/quality/agents/{agent_id}/performance?period=7d
```

**Response:**
```json
{
  "agent_id": "agent_123456",
  "period": {
    "start": "2025-01-16",
    "end": "2025-01-23"
  },
  "metrics": {
    "total_calls": 245,
    "successful_calls": 238,
    "average_duration": 185,
    "average_response_time_ms": 750,
    "customer_satisfaction": 4.3,
    "first_call_resolution": 0.82,
    "error_rate": 0.028
  },
  "trends": {
    "satisfaction_trend": "improving",
    "efficiency_trend": "stable"
  }
}
```

### Submit Customer Feedback
```http
POST /api/v1/quality/feedback
```

**Request Body:**
```json
{
  "call_id": "call_123456",
  "rating": 5,
  "nps_score": 9,
  "comment": "Very helpful and resolved my issue quickly",
  "categories": ["helpful", "fast", "knowledgeable"]
}
```

## Partner APIs

### List Partner Organizations
```http
GET /api/v1/partners/organizations
```

**Response:**
```json
{
  "organizations": [
    {
      "organization_id": "org_123456",
      "name": "ACME Corp",
      "relationship_type": "managed",
      "created_at": "2025-01-01T00:00:00Z",
      "statistics": {
        "total_agents": 12,
        "active_calls_today": 145,
        "monthly_revenue": 4500.00
      }
    }
  ]
}
```

### Get Partner Dashboard
```http
GET /api/v1/partners/dashboard
```

**Response:**
```json
{
  "partner": {
    "code": "PTR-1234-5678",
    "tier": "gold",
    "commission_rate": 25.00
  },
  "statistics": {
    "total_organizations": 15,
    "active_organizations": 14,
    "monthly_revenue": 45000.00,
    "monthly_commission": 11250.00,
    "ytd_commission": 125000.00
  }
}
```

## Error Handling

### HTTP Status Codes
- **200**: Success
- **201**: Created
- **400**: Bad Request
- **401**: Unauthorized
- **403**: Forbidden
- **404**: Not Found
- **409**: Conflict
- **422**: Unprocessable Entity
- **429**: Rate Limited
- **500**: Internal Server Error
- **503**: Service Unavailable

### Error Response Format
```json
{
  "error": {
    "code": "INVALID_AGENT_CONFIG",
    "message": "The agent configuration is invalid",
    "details": {
      "field": "system_prompt",
      "reason": "exceeds_max_length",
      "max_length": 32000
    },
    "request_id": "req_123456789",
    "timestamp": "2025-01-15T10:30:00Z"
  }
}
```

### Common Error Codes
- `AUTHENTICATION_REQUIRED`: API key or JWT token required
- `INVALID_API_KEY`: API key is invalid or expired
- `RATE_LIMIT_EXCEEDED`: API rate limit exceeded
- `AGENT_NOT_FOUND`: Specified agent does not exist
- `PHONE_NUMBER_UNAVAILABLE`: Requested phone number not available
- `INSUFFICIENT_BALANCE`: Account balance insufficient for operation
- `CALL_IN_PROGRESS`: Cannot modify agent while calls are active
- `INVALID_WEBHOOK_SIGNATURE`: Webhook signature verification failed

## Rate Limits

### API Rate Limits
- **Standard Plan**: 1,000 requests per hour
- **Professional Plan**: 10,000 requests per hour
- **Enterprise Plan**: 100,000 requests per hour

### Voice Call Limits
- **Concurrent Calls**: Based on plan (10/100/unlimited)
- **Monthly Minutes**: Based on plan and usage billing

### Webhook Retry Policy
- **Retry Attempts**: 3 retries with exponential backoff
- **Timeout**: 30 seconds per attempt
- **Backoff**: 1s, 2s, 4s

## SDKs and Libraries

### Node.js SDK
```javascript
const Aybiza = require('@aybiza/node-sdk');

const client = new Aybiza({
  apiKey: 'your-api-key',
  environment: 'production' // or 'staging'
});

// Create an agent
const agent = await client.agents.create({
  name: 'My Agent',
  system_prompt: 'You are a helpful assistant'
});

// Start a call
const call = await client.calls.create({
  agent_id: agent.id,
  to_number: '+19876543210'
});
```

### Python SDK
```python
from aybiza import Client

client = Client(
    api_key='your-api-key',
    environment='production'
)

# Create an agent
agent = client.agents.create(
    name='My Agent',
    system_prompt='You are a helpful assistant'
)

# Start a call
call = client.calls.create(
    agent_id=agent.id,
    to_number='+19876543210'
)
```

### cURL Examples
```bash
# Create agent
curl -X POST https://api.aybiza.com/api/v1/agents \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Agent",
    "system_prompt": "You are a helpful assistant"
  }'

# Get agent
curl https://api.aybiza.com/api/v1/agents/agent_123456 \
  -H "Authorization: Bearer your-api-key"
```

## Changelog

### v1.4.0 (2025-01-23)
- Added comprehensive Billing & Payment APIs with usage metering
- Added Identity & SSO APIs supporting SAML, OIDC, Azure AD, and Okta
- Added KYC & Compliance APIs with business verification and sanctions screening
- Added Feature Management APIs with feature flags and rollout controls
- Added White Label APIs for custom branding and domains
- Added Quality & Analytics APIs with AI-powered call scoring
- Added Partner APIs for reseller and partner management
- Enhanced API documentation with detailed request/response examples

### v1.3.0 (2025-01-23)
- Added Claude 4 Opus and Sonnet model support on AWS Bedrock
- Added Python code execution runtime support
- Enhanced documentation for AWS Bedrock Claude 4 integration
- Note: Some Claude 4 features may have limitations on Bedrock - see aws_bedrock_claude_implementation.md

### v1.2.0 (2025-01-15)
- Added automation tools API
- Enhanced analytics endpoints
- Improved error handling
- Added webhook signature verification

### v1.1.0 (2024-12-01)
- Added phone number management API
- Introduced real-time events WebSocket
- Enhanced call analytics
- Added multi-language support

### v1.0.0 (2024-10-01)
- Initial API release
- Basic agent management
- Voice call handling
- Core analytics

## Support

For API support and questions:
- **Documentation**: https://docs.aybiza.com
- **Support Email**: api-support@aybiza.com
- **Status Page**: https://status.aybiza.com
- **Community**: https://community.aybiza.com