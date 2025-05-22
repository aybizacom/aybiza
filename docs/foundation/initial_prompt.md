# Role: Enterprise AI Voice Agent Platform Developer Guide

# Task: Build an enterprise-grade AI voice agent platform with hybrid Cloudflare+AWS architecture using Elixir/Erlang

# Specifics:
- Build hybrid edge-cloud architecture combining Cloudflare global edge with AWS backend
- Use Elixir 1.18.3+ umbrella project structure with Docker containerization
- Implement ultra-low latency voice processing (<50ms for US/UK, <100ms globally)
- Support flexible phone number integration (managed, SIP trunk, traditional Twilio)
- Connect to Deepgram Nova-3 STT and Aura-2 TTS for premium voice quality
- Integrate Amazon Bedrock Claude 3.7 Sonnet for advanced conversational AI
- Implement enterprise-grade security with Zero Trust architecture
- Design for global scale (10,000+ concurrent calls per region)
- Include automation framework for business process integration
- Target 99.99% availability with multi-region failover

# Context:
AYBIZA is an enterprise-grade AI voice agent platform that enables businesses to create, deploy, and manage AI-powered voice agents with near-human conversation capabilities. The platform leverages a cutting-edge hybrid architecture combining Cloudflare's global edge network (300+ locations) with AWS backend services to deliver optimal performance, security, and cost efficiency.

The system processes speech in real-time with industry-leading latency, generates intelligent AI responses using state-of-the-art language models, and converts responses to natural-sounding speech. The platform supports multiple phone number acquisition methods, advanced automation workflows, and comprehensive analytics.

The hybrid architecture provides 28.6% cost savings compared to AWS-only solutions while delivering superior global performance through edge computing and intelligent routing.

# Enhanced Project Structure:
```
/aybiza (Umbrella Root)
  /cloudflare-workers       # Edge computing functions
    /voice-router          # Intelligent request routing
    /cache-manager         # Edge caching logic
    /auth-validator        # Edge authentication
  /apps
    /voice_gateway         # Twilio integration & call management
    /voice_pipeline        # Audio processing with Membrane Framework
    /conversation_engine   # Conversation state & Claude integration
    /agent_manager         # Agent configuration & deployment
    /phone_manager         # Flexible phone number management
    /automation_engine     # Workflow & tool execution framework
    /call_analytics        # Real-time analytics & insights
    /security              # Zero Trust security & compliance
  /config                  # Shared configuration
  /deps                    # Dependencies
  /docker                  # Container configuration
  /docs                    # Comprehensive documentation
  /priv                    # Private assets and migrations
  /terraform               # Infrastructure as code
```

# Hybrid Architecture Flow:
```
Global Users → Cloudflare Edge (300+ locations) →
  Intelligent Routing → Optimal AWS Region →
    Voice Gateway → Voice Pipeline →
      Deepgram Nova-3 STT → Claude 3.7 Conversation Engine →
        Automation Framework (if needed) → Deepgram Aura-2 TTS →
          Edge Optimization → Global Response
```

# Enhanced Voice Processing Pipeline:
```
Incoming Call → Twilio → Cloudflare Edge Processing →
  AWS Region Selection → Voice Gateway →
    μ-law Audio Stream → Deepgram Nova-3 STT →
      Context Management → Claude 3.7 Sonnet Streaming →
        Tool Execution (optional) → Deepgram Aura-2 TTS →
          Natural Speech → Edge Optimization → Caller
```

# Phone Number Integration Options:
```
1. AYBIZA Managed Numbers (Reseller Model)
   └── One-click setup, managed billing, integrated support

2. SIP Trunk Integration (BYOC)
   └── Existing numbers, avoid porting, UK/Enterprise focus

3. Traditional Twilio Integration
   └── Existing Twilio customers, full control
```

# Technology Stack (Latest):

## Edge Layer (Cloudflare)
- **Computing**: Cloudflare Workers (V8 JavaScript)
- **Storage**: Cloudflare KV, Durable Objects
- **Security**: WAF, DDoS (10+ Tbps), Bot Management
- **CDN**: 300+ global edge locations
- **DNS**: Anycast with health checks

## Backend Layer (AWS)
- **Runtime**: Elixir 1.18.3 / Erlang OTP 27.3.4
- **Framework**: Phoenix 1.7.21+ with LiveView
- **Database**: PostgreSQL 16.9+ with TimescaleDB 2.20.0
- **Cache**: Redis 8.0+ with cluster mode
- **Media**: Membrane Framework 0.11.0+
- **Containers**: Docker with multi-stage builds
- **Orchestration**: ECS Fargate with auto-scaling

## AI/ML Stack (Premium)
- **LLM**: Amazon Bedrock
  - Claude 3.7 Sonnet (anthropic.claude-3-7-sonnet-20250219-v1:0) - Latest with extended thinking
  - Claude 3.5 Sonnet v2 (anthropic.claude-3-5-sonnet-20241022-v2:0) - Balanced performance  
  - Claude 3.5 Haiku (anthropic.claude-3-5-haiku-20241022-v1:0) - Fastest intelligence
  - Claude 3 Haiku (anthropic.claude-3-haiku-20240307-v1:0) - Most cost-effective
- **STT**: Deepgram Nova-3 (54% accuracy improvement, multilingual)
- **TTS**: Deepgram Aura-2 (context-aware, natural speech, emotion control)

# Enhanced Agent Configuration Example:
```json
{
  "name": "Customer Support Agent",
  "version": "2.1.0",
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
  "greeting": "Hello, thank you for calling ACME support. How can I help you today?",
  "models": {
    "default": "claude-3-5-sonnet-v2",
    "fast": "claude-3-5-haiku",
    "complex": "claude-3-7-sonnet"
  },
  "automation": {
    "tools_enabled": true,
    "max_tool_calls": 5,
    "async_workflows": true
  },
  "parameters": {
    "max_response_time": 1500,
    "interruption_threshold": 0.7,
    "context_window": 32000,
    "temperature": 0.3
  },
  "phone_integration": {
    "type": "managed", // "managed", "sip_trunk", "twilio"
    "region": "us-east-1",
    "fallback_region": "us-west-2"
  }
}
```

# Core Technical Decisions (Updated):

1. **Hybrid Architecture**: Cloudflare edge + AWS backend for optimal performance and cost
2. **Latest Tech Stack**: Elixir 1.18.3, OTP 27.3.4, Phoenix 1.7.21, PostgreSQL 16.9
3. **Premium AI**: Claude 3.7 Sonnet, Nova-3 STT, Aura-2 TTS for best-in-class quality
4. **Zero Conversion**: Direct μ-law audio processing (no format conversion overhead)
5. **Multi-Model**: Dynamic model selection based on query complexity and latency requirements
6. **Edge-First**: Process at edge when possible, fallback to AWS for complex operations
7. **Flexible Integration**: Support multiple phone number acquisition methods
8. **Automation Ready**: Built-in framework for workflow and tool integration
9. **Global Scale**: Multi-region active-active deployment with intelligent routing
10. **Security by Design**: Zero Trust architecture with comprehensive compliance

# Implementation Roadmap:

## Phase 1: Foundation (Months 1-2)
1. Set up hybrid Cloudflare + AWS infrastructure
2. Implement core voice processing pipeline
3. Deploy Elixir umbrella with basic applications
4. Configure Docker containerization and CI/CD
5. Establish security framework and monitoring

## Phase 2: Advanced Features (Months 2-3)  
1. Implement flexible phone number management
2. Add multi-model Claude integration with streaming
3. Deploy automation framework with tool registry
4. Enhance edge processing and caching
5. Implement comprehensive analytics

## Phase 3: Enterprise Features (Months 3-4)
1. Multi-tenant isolation and compliance
2. Advanced monitoring and alerting
3. Global deployment and failover
4. Performance optimization and cost reduction
5. Enterprise security and audit capabilities

# Performance Targets:
- **Edge Response**: <10ms for cached requests
- **Voice Processing**: <50ms US/UK, <100ms globally  
- **STT First Result**: <50ms with Nova-3
- **LLM First Token**: <200ms (Sonnet), <100ms (Haiku)
- **TTS First Audio**: <50ms with Aura-2
- **Availability**: 99.99% with multi-region failover
- **Throughput**: 10,000+ concurrent calls per region

# Cost Optimization:
- **28.6% Savings**: vs AWS-only architecture
- **Edge Efficiency**: 50-70% reduction in origin requests
- **Smart Scaling**: Predictive scaling based on demand
- **Geographic Optimization**: Minimal data transfer costs
- **Reserved Capacity**: For predictable workloads

# Security Implementation (Zero Trust):
1. **Edge Security**: Cloudflare WAF with DDoS protection
2. **Network Security**: VPC with private subnets and security groups  
3. **Identity Management**: Multi-factor authentication with biometrics
4. **Data Protection**: End-to-end encryption with field-level PII protection
5. **Compliance**: SOC 2, HIPAA, GDPR, PCI DSS ready
6. **Audit**: Comprehensive logging across edge and backend
7. **Monitoring**: Real-time threat detection and response

# Getting Started (Updated):
1. **Infrastructure Setup**: Deploy Cloudflare + AWS hybrid environment
2. **Core Applications**: Implement umbrella project with 8 core apps
3. **Edge Integration**: Deploy Cloudflare Workers for routing and caching
4. **Voice Pipeline**: Build ultra-low latency audio processing
5. **AI Integration**: Connect to Bedrock Claude models with streaming
6. **Phone Integration**: Implement flexible number acquisition methods
7. **Automation Framework**: Add tool registry and workflow engine
8. **Testing & Monitoring**: Comprehensive testing and observability
9. **Security & Compliance**: Implement Zero Trust security model
10. **Global Deployment**: Multi-region deployment with failover

# Key Documentation Resources (Updated):
- [Elixir 1.18.3 Documentation](https://elixir-lang.org/docs.html)
- [Phoenix 1.7.21 Framework](https://hexdocs.pm/phoenix/overview.html)
- [Membrane Framework 0.11.0](https://membrane.stream/)
- [Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [AWS Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [Deepgram Nova-3 API](https://developers.deepgram.com/docs/nova-3)
- [Deepgram Aura-2 TTS](https://developers.deepgram.com/docs/aura-tts)
- [Twilio Media Streams](https://www.twilio.com/docs/voice/twiml/stream)
- [PostgreSQL 16.9](https://www.postgresql.org/docs/16/)
- [TimescaleDB 2.20.0](https://docs.timescale.com/)

# Testing Strategy (Enhanced):
1. **Unit Tests**: Individual component testing with 90%+ coverage
2. **Integration Tests**: Component interaction validation
3. **End-to-End Tests**: Full pipeline testing with real services
4. **Performance Tests**: Latency and throughput validation
5. **Security Tests**: Vulnerability assessment and penetration testing
6. **Load Tests**: Scalability validation under high load
7. **Edge Tests**: Cloudflare Worker functionality validation
8. **Multi-Region Tests**: Failover and geographic routing
9. **Compliance Tests**: SOC 2, HIPAA, GDPR validation

The development follows an iterative approach, building core capabilities first and expanding with advanced features. The hybrid architecture ensures optimal performance and cost efficiency while maintaining enterprise-grade security and compliance.