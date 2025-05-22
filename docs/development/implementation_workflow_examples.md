# Implementation Workflow Examples

## Overview

This document provides practical workflow examples that demonstrate how to use the Implementation Director methodology described in [40_implementation_director_guide.md](40_implementation_director_guide.md) along with the templates from [41_implementation_director_templates.md](41_implementation_director_templates.md) to implement AYBIZA components efficiently.

These workflow examples illustrate the end-to-end process from specification to completed implementation using Claude Code with the Implementation Director approach.

## Example 1: Implementing an Authentication Service

### Step 1: Create the Specification

First, create a specification for the authentication service following the guidelines in [39_specification_development_guide.md](39_specification_development_guide.md):

```markdown
# Authentication Service Specification
> Specification #45 - v1.0

## High-Level Objective

- Create a secure, flexible authentication service for AYBIZA voice agents

## Mid-Level Objectives

- Implement password-based authentication with Argon2 hashing
- Add JWT token generation and validation
- Support multi-factor authentication via SMS and authenticator apps
- Create session management with configurable timeouts
- Ensure comprehensive security testing

## Implementation Notes
- Use Elixir's Guardian library for JWT management
- Follow OWASP security best practices
- Store sensitive credentials in encrypted format
- Log all authentication attempts for audit purposes
- Include rate limiting for login attempts
- Ensure all functionality has >= 95% test coverage

## Context

### Beginning Context
- `lib/aybiza/accounts/user.ex` (existing)
- `lib/aybiza/schema.ex` (existing)
- `lib/aybiza/config.ex` (existing)
- `lib/aybiza/auth/` (empty directory)
- `test/aybiza/auth/` (empty directory)

### Ending Context
- `lib/aybiza/accounts/user.ex` (unchanged)
- `lib/aybiza/schema.ex` (unchanged)
- `lib/aybiza/config.ex` (unchanged)
- `lib/aybiza/auth/authentication.ex` (new)
- `lib/aybiza/auth/session.ex` (new)
- `lib/aybiza/auth/token.ex` (new)
- `lib/aybiza/auth/mfa.ex` (new)
- `test/aybiza/auth/authentication_test.exs` (new)
- `test/aybiza/auth/session_test.exs` (new)
- `test/aybiza/auth/token_test.exs` (new)
- `test/aybiza/auth/mfa_test.exs` (new)

## Low-Level Tasks
> Ordered from start to finish

1. Create the authentication module
```claude
CREATE lib/aybiza/auth/authentication.ex:
    IMPLEMENT Authentication module with:
        - verify_password(user, password) function using Argon2
        - authenticate_user(email, password) function 
        - include proper error handling for invalid credentials
        - add rate limiting for failed attempts
```

2. Create the token module
```claude
CREATE lib/aybiza/auth/token.ex:
    IMPLEMENT Token module with:
        - generate_token(user) function using Guardian
        - verify_token(token) function
        - refresh_token(token) function
        - proper error handling for invalid/expired tokens
```

3. Create the session module
```claude
CREATE lib/aybiza/auth/session.ex:
    IMPLEMENT Session module with:
        - create_session(user) function
        - validate_session(session_id) function
        - expire_session(session_id) function
        - configure session timeouts
```

4. Create the MFA module
```claude
CREATE lib/aybiza/auth/mfa.ex:
    IMPLEMENT MFA module with:
        - enable_sms_mfa(user, phone_number) function
        - enable_totp_mfa(user) function that returns setup key
        - verify_mfa_code(user, code) function
        - proper error handling
```

5. Write tests for authentication
```claude
CREATE test/aybiza/auth/authentication_test.exs:
    IMPLEMENT tests for all Authentication functions:
        - Test successful authentication
        - Test failed authentication
        - Test rate limiting
```

6. Write tests for token management
```claude
CREATE test/aybiza/auth/token_test.exs:
    IMPLEMENT tests for all Token functions:
        - Test token generation
        - Test token verification
        - Test token refresh
        - Test expired tokens
```

7. Write tests for session management
```claude
CREATE test/aybiza/auth/session_test.exs:
    IMPLEMENT tests for all Session functions:
        - Test session creation
        - Test session validation
        - Test session expiration
```

8. Write tests for MFA
```claude
CREATE test/aybiza/auth/mfa_test.exs:
    IMPLEMENT tests for all MFA functions:
        - Test SMS MFA enrollment and verification
        - Test TOTP MFA enrollment and verification
        - Test invalid code scenarios
```
```

### Step 2: Create the Director Configuration

Create a director configuration based on the templates:

```yaml
# Authentication Service Director Configuration

specification: 45_user_authentication_service_specification.md

coder_model: claude-3-7-sonnet-20250219
evaluator_model: claude-3-7-sonnet-20250219
max_iterations: 4

execution_command: mix test test/aybiza/auth/ --exclude integration

editable_files:
  - lib/aybiza/auth/authentication.ex
  - lib/aybiza/auth/session.ex
  - lib/aybiza/auth/token.ex
  - lib/aybiza/auth/mfa.ex
  - test/aybiza/auth/authentication_test.exs
  - test/aybiza/auth/session_test.exs
  - test/aybiza/auth/token_test.exs
  - test/aybiza/auth/mfa_test.exs

context_files:
  - lib/aybiza/accounts/user.ex
  - lib/aybiza/schema.ex
  - lib/aybiza/config.ex

evaluation_focus: security_focused

evaluation_criteria:
  - All tests must pass with no warnings
  - Password hashing must use Argon2id
  - Tokens must use proper expiration
  - All inputs must be validated
  - No sensitive data in logs
```

### Step 3: Run Claude Code with the Director Approach

In your terminal:

```bash
claude
```

In Claude Code, paste:

```
I'd like to implement the Authentication Service specification using the Implementation Director methodology with extended thinking for security-critical components.

Here's the specification:
[Paste the authentication service specification]

Here's the director configuration:
[Paste the YAML configuration]

Please guide me through the implementation process using the Implementation Director approach, and use extended thinking for the security-critical parts:

1. For the authentication module, please think hard about proper password hashing and validation to prevent timing attacks and ensure secure credential handling.

2. For the token module, please think harder about JWT implementation details, focusing on proper signing, validation, and preventing common JWT vulnerabilities.

3. For the MFA module, ultrathink about the security implications of different MFA approaches and how to securely implement them without introducing additional attack vectors.
```

This approach leverages Claude 3.7 Sonnet's extended thinking capabilities for the most security-critical components, ensuring thorough reasoning about potential security implications during implementation.

### Step 4: Director-Guided Implementation

Claude Code will now guide the implementation process:

1. **Analyze Specification**: Claude will analyze the specification requirements
2. **Plan Implementation**: Claude will create an implementation plan based on the low-level tasks
3. **Implement Code**: Claude will implement each module in the specified order
4. **Run Tests**: Claude will run the test command
5. **Evaluate Results**: Claude will check if the implementation meets requirements
6. **Iterate and Improve**: If needed, Claude will make improvements based on test results

### Step 5: Review and Finalize

After Claude completes the implementation:

1. Review the implementation for:
   - Code quality
   - Security considerations
   - Test coverage
   - Documentation
2. Make any manual adjustments if needed
3. Verify the implementation against the original specification

## Example 2: Implementing a Voice Activity Detection Feature

### Step 1: Create the Specification

```markdown
# Voice Activity Detection Specification
> Specification #46 - v1.0

## High-Level Objective

- Create a highly responsive voice activity detection system for AYBIZA voice agents

## Mid-Level Objectives

- Implement real-time voice activity detection with low latency
- Add configurable sensitivity settings for different environments
- Create adaptive background noise filtering
- Ensure compatibility with existing voice pipeline
- Optimize for resource efficiency

## Implementation Notes
- Implementation must be compatible with Membrane Framework
- Latency must be under 100ms for detection
- CPU usage should be under 5% when idle
- Use existing audio input streams
- Include comprehensive test suite with real audio samples

## Context

### Beginning Context
- `lib/aybiza/voice/pipeline.ex` (existing)
- `lib/aybiza/voice/audio_input.ex` (existing)
- `test/aybiza/voice/` (existing directory)
- `test/fixtures/audio_samples/` (directory with test audio files)

### Ending Context
- `lib/aybiza/voice/pipeline.ex` (modified)
- `lib/aybiza/voice/audio_input.ex` (unchanged)
- `lib/aybiza/voice/vad.ex` (new)
- `lib/aybiza/voice/noise_filter.ex` (new)
- `test/aybiza/voice/vad_test.exs` (new)
- `test/aybiza/voice/noise_filter_test.exs` (new)

## Low-Level Tasks
> Ordered from start to finish

1. Create voice activity detection module
```claude
CREATE lib/aybiza/voice/vad.ex:
    IMPLEMENT VoiceActivityDetection module with:
        - detect_activity(audio_frame) function returning true/false
        - configure_sensitivity(level) function
        - get_background_noise_level() function
        - reset_detection_state() function
```

2. Create noise filtering module
```claude
CREATE lib/aybiza/voice/noise_filter.ex:
    IMPLEMENT NoiseFilter module with:
        - filter_audio(audio_frame) function
        - adapt_to_environment(duration_ms) function
        - get_filter_stats() function
```

3. Update pipeline to integrate voice activity detection
```claude
UPDATE lib/aybiza/voice/pipeline.ex:
    MODIFY initialize_pipeline() function to integrate VoiceActivityDetection module
    ADD handle_voice_activity(activity_detected) callback
    ENSURE proper integration with existing audio flow
```

4. Write tests for voice activity detection
```claude
CREATE test/aybiza/voice/vad_test.exs:
    IMPLEMENT tests for VAD functionality:
        - Test detection with clear speech
        - Test silence detection
        - Test with background noise
        - Test sensitivity configurations
```

5. Write tests for noise filtering
```claude
CREATE test/aybiza/voice/noise_filter_test.exs:
    IMPLEMENT tests for noise filtering:
        - Test filter effectiveness
        - Test adaptation to environments
        - Test with various noise levels
```
```

### Step 2: Create Director Configuration

```yaml
# Voice Activity Detection Director Configuration

specification: 46_voice_activity_detection_specification.md

coder_model: claude-3-7-sonnet-20250219
evaluator_model: claude-3-7-sonnet-20250219
max_iterations: 3

execution_command: mix test test/aybiza/voice/

editable_files:
  - lib/aybiza/voice/vad.ex
  - lib/aybiza/voice/noise_filter.ex
  - lib/aybiza/voice/pipeline.ex
  - test/aybiza/voice/vad_test.exs
  - test/aybiza/voice/noise_filter_test.exs

context_files:
  - lib/aybiza/voice/audio_input.ex
  - test/fixtures/audio_samples/

evaluation_focus: default

evaluation_criteria:
  - All tests must pass
  - Voice detection latency < 100ms
  - CPU usage < 5% when idle
  - Proper integration with Membrane Framework
  - Effective noise filtering
```

### Step 3: Run Implementation Director Process

Using Claude Code, follow the same pattern as in Example 1 to implement the voice activity detection feature.

## Example 3: Multi-Specification Project Implementation

For a larger project involving multiple related specifications, follow this workflow:

### Step 1: Create a Project Plan

```markdown
# Voice Agent Core Components Implementation Plan

## Components
1. Authentication Service - [Handles user and agent authentication]
2. Voice Activity Detection - [Detects speech in audio streams]
3. Conversation Manager - [Manages dialog flow and state]

## Implementation Order
1. Specification #45: Authentication Service
2. Specification #46: Voice Activity Detection
3. Specification #47: Conversation Manager

## Integration Points
- Authentication Service provides agent validation for Conversation Manager
- Voice Activity Detection triggers input events for Conversation Manager
- Conversation Manager orchestrates the overall voice interaction flow
```

### Step 2: Implement Each Specification in Sequence

1. Implement Authentication Service using its specification and director configuration
2. Validate the Authentication Service implementation
3. Implement Voice Activity Detection using its specification and director configuration
4. Validate the Voice Activity Detection implementation
5. Implement Conversation Manager using its specification and director configuration
6. Validate the Conversation Manager implementation

### Step 3: Perform Integration Testing

After implementing all components, create an integration test specification:

```markdown
# Voice Agent Integration Test Specification
> Specification #48 - v1.0

## High-Level Objective

- Validate that all voice agent core components work together correctly

## Mid-Level Objectives

- Test end-to-end authentication flow
- Verify voice activity triggers conversation events
- Test conversation state management
- Validate system performance under load
- Ensure security across component boundaries

## Implementation Notes
- Use existing component implementations
- Focus on integration points rather than individual component functionality
- Include both success and failure scenarios
- Test with realistic voice input samples

## Context

### Beginning Context
- Authentication Service (implemented from Spec #45)
- Voice Activity Detection (implemented from Spec #46)
- Conversation Manager (implemented from Spec #47)
- `test/integration/` (empty directory)

### Ending Context
- `test/integration/authentication_flow_test.exs` (new)
- `test/integration/voice_conversation_test.exs` (new)
- `test/integration/performance_test.exs` (new)
- `test/integration/security_test.exs` (new)

## Low-Level Tasks
> Ordered from start to finish

1. Create authentication flow integration tests
```claude
CREATE test/integration/authentication_flow_test.exs:
    IMPLEMENT integration tests for authentication flow:
        - Test agent authentication with conversation system
        - Test authentication failures and system response
        - Test session management across subsystems
```

2. Create voice conversation integration tests
```claude
CREATE test/integration/voice_conversation_test.exs:
    IMPLEMENT integration tests for voice conversation flow:
        - Test voice activity triggering conversation updates
        - Test full conversation cycle with simulated voice input
        - Test error handling between components
```

3. Create performance integration tests
```claude
CREATE test/integration/performance_test.exs:
    IMPLEMENT performance tests for the integrated system:
        - Test end-to-end latency
        - Test system under load (multiple concurrent conversations)
        - Test resource usage
```

4. Create security integration tests
```claude
CREATE test/integration/security_test.exs:
    IMPLEMENT security tests for the integrated system:
        - Test authentication boundaries
        - Test authorization controls
        - Test for data leakage between components
```
```

### Step 4: Create Director Configuration for Integration Testing

```yaml
# Integration Testing Director Configuration

specification: 48_voice_agent_integration_test_specification.md

coder_model: claude-3-7-sonnet-20250219
evaluator_model: claude-3-7-sonnet-20250219
max_iterations: 3

execution_command: mix test test/integration/

editable_files:
  - test/integration/authentication_flow_test.exs
  - test/integration/voice_conversation_test.exs
  - test/integration/performance_test.exs
  - test/integration/security_test.exs

context_files:
  - lib/aybiza/auth/
  - lib/aybiza/voice/
  - lib/aybiza/conversation/
  - test/fixtures/

evaluation_focus: default

evaluation_criteria:
  - All integration tests must pass
  - End-to-end latency < 500ms
  - No security breaches between components
  - Proper error handling across boundaries
```

## Best Practices from These Examples

Based on these workflow examples, here are some best practices for using the Implementation Director methodology:

### 1. Specification Development

- **Start with clear objectives**: Define high-level and mid-level objectives precisely
- **Be explicit about constraints**: Include clear performance, security, and compatibility requirements
- **Define concrete tasks**: Break down implementation into specific, actionable steps
- **Include test requirements**: Define what successful tests look like

### 2. Director Configuration

- **Choose appropriate models**: More complex components need more capable models
- **Set reasonable iteration limits**: Most implementations succeed within 2-4 iterations
- **Include all relevant files**: Both editable and context files should be comprehensive
- **Choose the right evaluation focus**: Security-focused for auth, performance-focused for real-time features

### 3. Implementation Process

- **Review intermediate results**: Check Claude's progress after each iteration
- **Provide feedback when needed**: Guide Claude if it misses important aspects
- **Validate thoroughly**: Run additional tests beyond the automated validation
- **Document key decisions**: Note important implementation decisions for future reference

### 4. Multi-Component Projects

- **Implement foundational components first**: Start with components others depend on
- **Validate each component individually**: Ensure each piece works before integration
- **Plan integration carefully**: Define how components should interact
- **Test end-to-end flows**: Validate the complete system behavior

## Conclusion

These workflow examples demonstrate how to effectively use the Implementation Director methodology to implement AYBIZA components. By following these patterns, you can create high-quality, well-tested, and secure components that integrate smoothly into the larger AYBIZA platform.

The Implementation Director approach streamlines the development process, ensuring consistent quality and reducing the cognitive load on developers by automating repetitive aspects of implementation while maintaining quality standards.

## References

- [39_specification_development_guide.md](39_specification_development_guide.md)
- [40_implementation_director_guide.md](40_implementation_director_guide.md)
- [41_implementation_director_templates.md](41_implementation_director_templates.md)