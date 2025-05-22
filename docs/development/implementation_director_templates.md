# Implementation Director Templates

## Overview

This document provides ready-to-use templates for the Implementation Director methodology described in [40_implementation_director_guide.md](40_implementation_director_guide.md). These templates help standardize the creation of director configurations and facilitate consistent implementation of AYBIZA components.

## Director Configuration Templates

### Basic Configuration Template

```yaml
# AYBIZA Implementation Director Configuration - Basic Template

# The specification document to implement (path to .md file)
specification: path/to/specification.md

# The Claude model to use for code generation
# Recommended models:
# - claude-3-7-sonnet-20250219: For complex implementations (highest capabilities + extended thinking)
# - claude-3-5-sonnet-20241022: For moderate complexity implementations
# - claude-3-5-haiku-20241022: For simpler implementations
coder_model: claude-3-7-sonnet-20250219

# The Claude model to use for evaluation
# Recommended to use claude-3-7-sonnet for thorough reasoning and evaluation
evaluator_model: claude-3-7-sonnet-20250219

# Maximum number of implementation/refinement iterations
# Recommended range: 2-5 iterations
max_iterations: 3

# Command to run tests for validation
# Examples:
# - Elixir: mix test path/to/test_files
# - JavaScript: npm test
# - Python: pytest path/to/test_files
execution_command: mix test path/to/test_files

# Files that can be modified by Claude Code
# These should be the implementation files for the specification
editable_files:
  - path/to/file1.ext
  - path/to/file2.ext
  - path/to/directory/

# Files that provide context but cannot be modified
# These help Claude understand the codebase architecture
context_files:
  - path/to/context_file1.ext
  - path/to/context_file2.ext
  
# Evaluation focus
# Options:
# - default: Standard evaluation (functionality, correctness)
# - test_focused: Emphasizes test coverage and quality
# - security_focused: Emphasizes security best practices
evaluation_focus: default
```

### Comprehensive Configuration Template

```yaml
# AYBIZA Implementation Director Configuration - Comprehensive Template

# === Basic Configuration ===

# The specification document to implement (path to .md file or inline spec)
specification: path/to/specification.md

# The Claude model to use for code generation
coder_model: claude-3-7-sonnet-20250219

# The Claude model to use for evaluation
evaluator_model: claude-3-7-sonnet-20250219

# Maximum number of implementation/refinement iterations
max_iterations: 3

# Command to run tests for validation
execution_command: mix test path/to/test_files

# === File Management ===

# Files that can be modified by Claude Code
editable_files:
  - path/to/file1.ext
  - path/to/file2.ext
  - path/to/directory/

# Files that provide context but cannot be modified
context_files:
  - path/to/context_file1.ext
  - path/to/context_file2.ext

# === Evaluation Configuration ===

# Evaluation focus
evaluation_focus: default

# Additional evaluation criteria (optional)
evaluation_criteria:
  - All tests must pass with no warnings
  - Code must follow Elixir style guidelines
  - Documentation must be comprehensive
  - Performance must meet specified benchmarks

# === Additional Settings ===

# Generate documentation alongside code (true/false)
generate_documentation: true

# Code style to enforce
code_style: elixir_standard

# Security requirements to enforce
security_requirements:
  - No hardcoded credentials
  - Input validation for all external data
  - Proper error handling
  - Secure token management

# Performance requirements
performance_requirements:
  - Response time < 100ms
  - Memory usage < 200MB
  - CPU usage < 10% average

# Documentation requirements
documentation_requirements:
  - Module documentation
  - Function documentation with @spec
  - Usage examples
  - Architecture diagrams (when applicable)
```

## Implementation Plan Templates

### Basic Implementation Plan Template

```markdown
# [Component Name] Implementation Plan

## Specification
- [Path to specification document]

## Implementation Steps
1. [Step 1 description]
2. [Step 2 description]
3. [Step 3 description]
4. [Step 4 description]
5. [Step 5 description]

## Testing Criteria
- [Test criterion 1]
- [Test criterion 2]
- [Test criterion 3]
```

### Comprehensive Implementation Plan Template

```markdown
# [Component Name] Implementation Plan

## Specification
- [Path to specification document]

## Overview
[Brief description of the component and its purpose]

## Dependencies
- [Dependency 1]: [Description]
- [Dependency 2]: [Description]

## Implementation Steps
1. [Step 1]
   - [Subtask 1.1]
   - [Subtask 1.2]
   - Expected output: [Description]

2. [Step 2]
   - [Subtask 2.1]
   - [Subtask 2.2]
   - Expected output: [Description]

3. [Step 3]
   - [Subtask 3.1]
   - [Subtask 3.2]
   - Expected output: [Description]

## Testing Strategy
- Unit tests:
  - [Test area 1]
  - [Test area 2]

- Integration tests:
  - [Test scenario 1]
  - [Test scenario 2]

- Performance tests:
  - [Performance metric 1]: [Target]
  - [Performance metric 2]: [Target]

## Success Criteria
- [Criterion 1]
- [Criterion 2]
- [Criterion 3]

## Documentation Requirements
- [Documentation item 1]
- [Documentation item 2]

## Security Considerations
- [Security consideration 1]
- [Security consideration 2]
```

## Application-Specific Templates

### Authentication Service Director Configuration

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
  - test/aybiza/auth/

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

### Voice Pipeline Director Configuration

```yaml
# Voice Pipeline Director Configuration

specification: 46_voice_processing_service_specification.md

coder_model: claude-3-7-sonnet-20250219
evaluator_model: claude-3-7-sonnet-20250219
max_iterations: 4

execution_command: mix test test/aybiza/voice/

editable_files:
  - lib/aybiza/voice/pipeline.ex
  - lib/aybiza/voice/transcription.ex
  - lib/aybiza/voice/synthesis.ex
  - lib/aybiza/voice/activity_detection.ex
  - test/aybiza/voice/

context_files:
  - lib/aybiza/deepgram_client.ex
  - lib/aybiza/bedrock_client.ex
  - lib/aybiza/schema.ex

evaluation_focus: default

evaluation_criteria:
  - All tests must pass with no warnings
  - End-to-end latency must be below 300ms
  - Must handle various audio formats
  - Proper error handling for service outages
  - Streaming mode must work correctly
```

## Example Usage Guide

### How to Use These Templates

1. **Select the appropriate template** for your component or feature
2. **Copy the template** to a new file
3. **Customize the configuration** with your specific file paths and requirements
4. **Save the configuration** as a YAML file (e.g., `auth_service_director.yaml`)

### Director Workflow Example

```bash
# Start Claude Code in your terminal
claude
```

Then within Claude Code:

```
I'd like to implement the Authentication Service specification using the Implementation Director methodology.

Here's the specification:
[45_user_authentication_service_specification.md](45_user_authentication_service_specification.md)

Here's the director configuration:
```yaml
# Authentication Service Director Configuration
specification: 45_user_authentication_service_specification.md
coder_model: claude-3-5-sonnet-20241022
evaluator_model: claude-3-5-sonnet-20241022
max_iterations: 4
execution_command: mix test test/aybiza/auth/ --exclude integration
editable_files:
  - lib/aybiza/auth/authentication.ex
  - lib/aybiza/auth/session.ex
  - lib/aybiza/auth/token.ex
  - lib/aybiza/auth/mfa.ex
  - test/aybiza/auth/
context_files:
  - lib/aybiza/accounts/user.ex
  - lib/aybiza/schema.ex
  - lib/aybiza/config.ex
evaluation_focus: security_focused
```

Please guide the implementation process according to the Implementation Director methodology.
```

## Customizing Templates for Your Needs

### Elixir/Phoenix Specific Enhancements

For Elixir/Phoenix applications like AYBIZA, consider adding these specific configuration elements:

```yaml
# Elixir-specific configuration

# Formatter configuration
formatter_command: mix format

# Dialyzer type checking
dialyzer_command: mix dialyzer

# Code quality checks
credo_command: mix credo --strict

# Documentation generation
ex_doc_command: mix docs

# Additional Elixir-specific evaluation criteria
elixir_evaluation_criteria:
  - Functions have proper @spec annotations
  - Module documentation follows ExDoc standards
  - Supervisor strategies are appropriate
  - GenServer callbacks are optimized
```

### Security-Focused Templates

For components with stringent security requirements:

```yaml
# Security-focused configuration

security_focused_evaluator: true

security_evaluation_criteria:
  - No hardcoded credentials or secrets
  - All user inputs are validated and sanitized
  - Authentication tokens have appropriate lifetimes
  - Sensitive data is properly encrypted
  - Proper error handling without information disclosure
  - Defense against common attacks (injection, XSS, CSRF)
  - Secure session management
  - Rate limiting implemented for sensitive operations
  - Proper audit logging of security events

security_test_requirements:
  - Test for authentication bypass
  - Test for authorization bypass
  - Test for input validation failures
  - Test for improper error handling
  - Test for insecure direct object references
```

## Conclusion

These templates provide a standardized approach to implementing AYBIZA components using the Implementation Director methodology. By following these templates and customizing them for your specific needs, you can ensure consistent, high-quality implementation across all parts of the AYBIZA platform.

Use these templates in conjunction with the Implementation Director Guide ([40_implementation_director_guide.md](40_implementation_director_guide.md)) and the Specification Development Guide ([39_specification_development_guide.md](39_specification_development_guide.md)) for a comprehensive approach to specification-driven development with Claude Code.

## References

- [39_specification_development_guide.md](39_specification_development_guide.md)
- [40_implementation_director_guide.md](40_implementation_director_guide.md)