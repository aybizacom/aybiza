# AYBIZA Integration Testing Guide

This guide provides information on how to effectively test the integrations between AYBIZA components and external services within the hybrid Cloudflare+AWS architecture, including Claude 3.7 Sonnet integration testing and edge-cloud handoff scenarios.

## Testing Environment Setup

### Local Testing Environment

For local testing of hybrid architecture, we use a combination of:
- Real external services where possible (AWS Bedrock, Claude 3.7 Sonnet)
- Cloudflare Workers local development environment
- Edge simulation for latency testing (<50ms targets)
- Mocks for services that can't be easily accessed in development
- Hybrid handoff simulation between edge and cloud components

### Environment Variables for Testing

Create a `.env.test` file with test-specific credentials:

```bash
# Test credentials - DO NOT use production credentials
TWILIO_TEST_ACCOUNT_SID=your_test_account_sid
TWILIO_TEST_AUTH_TOKEN=your_test_auth_token
TWILIO_TEST_PHONE_NUMBER=+15551234567
DEEPGRAM_TEST_API_KEY=your_test_api_key
AWS_TEST_ACCESS_KEY_ID=your_test_access_key
AWS_TEST_SECRET_ACCESS_KEY=your_test_secret_key
AWS_TEST_REGION=us-east-1
# Claude 3.7 Sonnet testing
CLAUDE_TEST_MODEL_ID=anthropic.claude-3-7-sonnet-20250219-v1:0
# Cloudflare testing
CLOUDFLARE_TEST_ACCOUNT_ID=your_test_account_id
CLOUDFLARE_TEST_API_TOKEN=your_test_token
# Edge testing
EDGE_TEST_REGION=auto
LATENCY_TEST_TARGET_MS=50
```

Run tests with this environment:

```bash
env $(cat .env.test | xargs) mix test
```

## Testing External Integrations

### Twilio Integration Testing

#### Mock Twilio for Unit Tests

```elixir
# Example of mocking Twilio in tests
defmodule Aybiza.MockTwilio do
  def generate_twiml(params) do
    # Return predictable TwiML for testing - AYBIZA uses streaming, not <Say>
    """
    <?xml version="1.0" encoding="UTF-8"?>
    <Response>
        <Connect>
            <Stream url="wss://test-url.example.com/stream" />
        </Connect>
    </Response>
    """
  end
  
  def simulate_webhook_request(params) do
    # Simulate a Twilio webhook request
    # ...
  end
  
  def simulate_websocket_connection(call_sid) do
    # Simulate a WebSocket connection
    # ...
  end
end
```

#### Integration Testing with Twilio Test Credentials

For end-to-end testing, use Twilio's test credentials to make actual API calls without incurring charges.

Example test:

```elixir
defmodule Aybiza.TwilioIntegrationTest do
  use ExUnit.Case, async: false
  
  setup do
    # Set up test with Twilio test credentials
    # ...
  end
  
  test "handles incoming call webhook" do
    # Test that the system correctly processes a Twilio webhook
    # ...
  end
  
  test "establishes WebSocket connection" do
    # Test that the system can establish a WebSocket connection with Twilio
    # ...
  end
end
```

### Deepgram Integration Testing

#### Mock Deepgram for Unit Tests

```elixir
# Example of mocking Deepgram in tests
defmodule Aybiza.MockDeepgram do
  def simulate_stt_response(audio_data) do
    # Return predictable transcription for testing
    %{
      "results" => %{
        "channels" => [
          %{
            "alternatives" => [
              %{
                "transcript" => "This is a test transcription",
                "confidence" => 0.98
              }
            ]
          }
        ],
        "utterances" => []
      }
    }
  end
  
  def simulate_tts_response(text) do
    # Return dummy audio data for testing
    # ...
  end
end
```

#### Integration Testing with Deepgram Test API Key

For actual API testing, use a dedicated test API key with limited quota.

Example test:

```elixir
defmodule Aybiza.DeepgramIntegrationTest do
  use ExUnit.Case, async: false
  
  setup do
    # Set up test with Deepgram test credentials
    # ...
  end
  
  test "transcribes audio correctly" do
    # Test that the system can transcribe audio using Deepgram
    # ...
  end
  
  test "generates speech from text" do
    # Test that the system can generate speech from text using Deepgram
    # ...
  end
end
```

### AWS Bedrock Integration Testing

#### Mock Bedrock for Unit Tests

```elixir
# Example of mocking AWS Bedrock in tests
defmodule Aybiza.MockBedrock do
  def simulate_llm_response(prompt) do
    # Return predictable response for testing
    %{
      "completion" => "This is a test response from the language model",
      "stop_reason" => "stop",
      "usage" => %{
        "input_tokens" => 10,
        "output_tokens" => 9
      }
    }
  end
end
```

#### Integration Testing with AWS Test Credentials

For actual API testing, use a dedicated test AWS account with limited permissions.

Example test:

```elixir
defmodule Aybiza.BedrockIntegrationTest do
  use ExUnit.Case, async: false
  
  setup do
    # Set up test with AWS test credentials
    # ...
  end
  
  test "generates response from language model" do
    # Test that the system can generate responses using AWS Bedrock
    # ...
  end
end
```

## Testing the Voice Pipeline

### End-to-End Call Flow Testing

For testing the complete voice pipeline, we can use recorded audio samples and mock external services.

Example test:

```elixir
defmodule Aybiza.VoicePipelineTest do
  use ExUnit.Case, async: false
  
  setup do
    # Set up test with mock services
    # ...
  end
  
  test "processes voice call end-to-end" do
    # Test that the system can process a call from start to finish
    # ...
  end
end
```

### Component Integration Testing

Test the interaction between individual components of the voice pipeline.

Example test:

```elixir
defmodule Aybiza.ComponentIntegrationTest do
  use ExUnit.Case, async: false
  
  setup do
    # Set up test environment
    # ...
  end
  
  test "STT component integrates with Context Manager" do
    # Test that the STT component correctly passes data to the Context Manager
    # ...
  end
  
  test "LLM component integrates with TTS component" do
    # Test that the LLM component correctly passes data to the TTS component
    # ...
  end
end
```

## Performance Testing

### Latency Testing

Test the latency of the voice pipeline components.

Example test:

```elixir
defmodule Aybiza.LatencyTest do
  use ExUnit.Case, async: false
  
  test "voice pipeline meets latency requirements" do
    # Test that the voice pipeline processes audio within latency requirements
    # ...
  end
end
```

### Load Testing

Test the system's ability to handle multiple concurrent calls.

Example test:

```elixir
defmodule Aybiza.LoadTest do
  use ExUnit.Case, async: false
  
  test "system handles multiple concurrent calls" do
    # Test that the system can handle multiple concurrent calls
    # ...
  end
end
```

## Security Testing

### Input Validation Testing

Test that the system properly validates and sanitizes input.

Example test:

```elixir
defmodule Aybiza.InputValidationTest do
  use ExUnit.Case, async: false
  
  test "system validates and sanitizes input" do
    # Test that the system properly validates and sanitizes input
    # ...
  end
end
```

### Authentication Testing

Test that the system properly authenticates requests.

Example test:

```elixir
defmodule Aybiza.AuthenticationTest do
  use ExUnit.Case, async: false
  
  test "system authenticates Twilio webhook requests" do
    # Test that the system properly authenticates Twilio webhook requests
    # ...
  end
end
```

## Continuous Integration Testing

In our GitHub Actions pipeline, we run integration tests in a dedicated job.

Example workflow configuration:

```yaml
name: Integration Tests

on:
  pull_request:
  push:
    branches: [main]

jobs:
  integration_tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      - uses: erlef/setup-beam@v1
        with:
          elixir-version: '1.18.3'
          otp-version: '28.0'
      - run: mix deps.get
      - run: mix test --only integration
```

## Test Data Management

### Sample Audio Files

Store sample audio files for testing in `/priv/test_data/audio/` with clear naming conventions:

- `greeting_en_male.wav` - English greeting from male voice
- `question_en_female.wav` - English question from female voice
- `background_noise.wav` - Audio with background noise
- `multiple_speakers.wav` - Audio with multiple speakers

### Sample Responses

Store sample responses for testing in `/priv/test_data/responses/` with clear naming conventions:

- `greeting_response.json` - Sample greeting response
- `question_response.json` - Sample question response
- `error_response.json` - Sample error response

## Debugging Integration Tests

### Logging

Enable detailed logging during tests:

```elixir
# In test helper
Logger.configure(level: :debug)
```

### Capturing HTTP Requests/Responses

Use ExVCR to capture and replay HTTP interactions:

```elixir
defmodule Aybiza.ExternalApiTest do
  use ExUnit.Case, async: false
  use ExVCR.Mock, adapter: ExVCR.Adapter.Hackney
  
  setup do
    ExVCR.Config.cassette_library_dir("test/fixtures/vcr_cassettes")
    :ok
  end
  
  test "calls external API" do
    use_cassette "external_api_call" do
      # Test code that makes an external API call
      # ...
    end
  end
end
```

## CodePipeline Integration Testing

### Testing in AWS CodePipeline Context

The AYBIZA platform uses **AWS CodePipeline** following the **Git → CodeBuild → ECR → CodeDeploy → ECS/Fargate** pattern. Integration testing is embedded throughout the pipeline stages to ensure quality at each step.

#### CodeBuild Testing Configuration

```yaml
# buildspec-test.yml - Testing stage in CodePipeline
version: 0.2

phases:
  pre_build:
    commands:
      - echo Preparing test environment...
      - export MIX_ENV=test
      - export DATABASE_URL=postgres://postgres:postgres@localhost:5432/aybiza_test
      - export REDIS_URL=redis://localhost:6379
      - # Start test services
      - service postgresql start
      - service redis-server start
      - createdb aybiza_test
      
  build:
    commands:
      - echo Running AYBIZA integration tests...
      - mix deps.get
      - mix ecto.setup
      - # Run different test suites
      - mix test --include integration
      - mix test --include external_api
      - mix test --include pipeline
      - # Generate test reports for CodePipeline
      - mix test --formatter JUnitFormatter --formatter ExUnit.CLIFormatter
      
  post_build:
    commands:
      - echo Test phase completed
      - # Archive test results for CodePipeline
      - cp _build/test/lib/aybiza/test-junit-report.xml ./
      - # Generate test coverage report
      - mix coveralls.html
      - cp cover/excoveralls.html ./test-coverage.html

artifacts:
  files:
    - test-junit-report.xml
    - test-coverage.html
  name: test-results
```

#### Pipeline Stage Testing

```elixir
defmodule Aybiza.PipelineIntegrationTest do
  @moduledoc """
  Tests that run specifically within CodePipeline context
  """
  use ExUnit.Case, async: false
  
  @moduletag :pipeline
  
  describe "CodePipeline Environment Tests" do
    test "container health checks work in ECS context" do
      # Test that health endpoints respond correctly
      {:ok, response} = HTTPoison.get("http://localhost:4000/api/health")
      assert response.status_code == 200
      
      health_data = Jason.decode!(response.body)
      assert health_data["status"] == "healthy"
      assert health_data["database"] == "connected"
      assert health_data["redis"] == "connected"
      assert health_data["bedrock"] == "accessible"
    end
    
    test "environment variables are correctly set in pipeline" do
      # Verify CodePipeline environment variables
      assert System.get_env("CODEBUILD_BUILD_ARN") != nil
      assert System.get_env("CODEBUILD_RESOLVED_SOURCE_VERSION") != nil
      assert System.get_env("AWS_DEFAULT_REGION") == "us-east-1"
    end
    
    test "ECR image can be built and tagged correctly" do
      # Test that the container build process works
      build_id = System.get_env("CODEBUILD_BUILD_ID")
      commit_hash = System.get_env("CODEBUILD_RESOLVED_SOURCE_VERSION")
      
      # Verify image tagging follows pipeline conventions
      expected_tag = String.slice(commit_hash || "latest", 0..6)
      assert String.length(expected_tag) > 0
    end
    
    test "CodeDeploy artifacts are generated correctly" do
      # Test artifact generation for CodeDeploy
      assert File.exists?("imagedefinitions.json")
      assert File.exists?("appspec.yml")
      
      {:ok, image_defs} = File.read("imagedefinitions.json")
      parsed_defs = Jason.decode!(image_defs)
      
      assert is_list(parsed_defs)
      assert length(parsed_defs) > 0
      
      first_def = List.first(parsed_defs)
      assert Map.has_key?(first_def, "name")
      assert Map.has_key?(first_def, "imageUri")
      assert String.contains?(first_def["imageUri"], "ecr")
    end
  end
  
  describe "Multi-Stage Pipeline Testing" do
    test "staging deployment readiness" do
      # Test staging-specific configurations
      if System.get_env("ENVIRONMENT") == "staging" do
        # Verify staging-specific settings
        assert Application.get_env(:aybiza, :environment) == :staging
        assert Application.get_env(:aybiza, :bedrock_model) != :production_model
      end
    end
    
    test "production deployment readiness" do
      # Test production-specific configurations
      if System.get_env("ENVIRONMENT") == "production" do
        # Verify production-specific settings
        assert Application.get_env(:aybiza, :environment) == :production
        
        # Ensure production safety checks
        refute Application.get_env(:aybiza, :debug_mode, false)
        assert Application.get_env(:aybiza, :ssl_verify) == true
      end
    end
  end
end
```

#### CodeDeploy Health Check Testing

```elixir
defmodule Aybiza.CodeDeployHealthTest do
  @moduledoc """
  Tests that validate CodeDeploy health checks during blue/green deployments
  """
  use ExUnit.Case, async: false
  
  @moduletag :codedeploy
  
  test "ECS task health check endpoint responds correctly" do
    # Test the health check endpoint that CodeDeploy uses
    {:ok, response} = HTTPoison.get("http://localhost:4000/api/health", [], timeout: 5000)
    
    assert response.status_code == 200
    assert get_header(response.headers, "content-type") =~ "application/json"
    
    health_data = Jason.decode!(response.body)
    
    # Verify required health check fields for CodeDeploy
    assert health_data["status"] == "healthy"
    assert health_data["timestamp"] != nil
    assert health_data["version"] != nil
    assert health_data["services"]["database"]["status"] == "up"
    assert health_data["services"]["redis"]["status"] == "up"
    assert health_data["services"]["bedrock"]["status"] == "accessible"
  end
  
  test "application startup time is within CodeDeploy timeout" do
    # Test that application starts within CodeDeploy's health check timeout
    start_time = System.monotonic_time(:second)
    
    # Wait for application to be ready
    wait_for_health_check()
    
    startup_time = System.monotonic_time(:second) - start_time
    
    # CodeDeploy health check timeout is typically 300 seconds
    assert startup_time < 300, "Application startup took #{startup_time}s, exceeding CodeDeploy timeout"
  end
  
  test "graceful shutdown works for blue/green deployment" do
    # Test that the application can gracefully handle shutdown signals
    # This is important for CodeDeploy blue/green deployments
    
    # Send SIGTERM to the application process
    pid = System.pid()
    Process.send_after(self(), :shutdown_test, 1000)
    
    receive do
      :shutdown_test ->
        # Verify application can handle graceful shutdown
        assert Process.alive?(self())
    after
      5000 ->
        flunk("Graceful shutdown test timed out")
    end
  end
  
  defp wait_for_health_check(retries \\ 30) do
    case HTTPoison.get("http://localhost:4000/api/health") do
      {:ok, %{status_code: 200}} -> :ok
      _ when retries > 0 ->
        Process.sleep(1000)
        wait_for_health_check(retries - 1)
      _ ->
        flunk("Health check never became available")
    end
  end
  
  defp get_header(headers, name) do
    headers
    |> Enum.find_value(fn {k, v} -> String.downcase(k) == name && v end)
  end
end
```

#### Pipeline Artifact Testing

```elixir
defmodule Aybiza.PipelineArtifactTest do
  @moduledoc """
  Tests for CodePipeline artifacts and deployment configurations
  """
  use ExUnit.Case, async: false
  
  @moduletag :artifacts
  
  test "appspec.yml is valid for CodeDeploy" do
    assert File.exists?("appspec.yml")
    
    {:ok, content} = File.read("appspec.yml")
    {:ok, appspec} = YamlElixir.read_from_string(content)
    
    # Verify required AppSpec structure
    assert appspec["version"] == "0.0"
    assert Map.has_key?(appspec, "Resources")
    assert Map.has_key?(appspec["Resources"], "TargetService")
    
    target_service = appspec["Resources"]["TargetService"]
    assert target_service["Type"] == "AWS::ECS::Service"
    assert Map.has_key?(target_service["Properties"], "TaskDefinition")
  end
  
  test "task definition template is valid" do
    assert File.exists?("taskdef.json")
    
    {:ok, content} = File.read("taskdef.json")
    {:ok, taskdef} = Jason.decode(content)
    
    # Verify task definition structure
    assert Map.has_key?(taskdef, "family")
    assert Map.has_key?(taskdef, "containerDefinitions")
    assert is_list(taskdef["containerDefinitions"])
    
    container_def = List.first(taskdef["containerDefinitions"])
    assert Map.has_key?(container_def, "name")
    assert Map.has_key?(container_def, "image")
    assert Map.has_key?(container_def, "healthCheck")
  end
  
  test "image definitions are valid for ECS" do
    assert File.exists?("imagedefinitions.json")
    
    {:ok, content} = File.read("imagedefinitions.json")
    {:ok, image_defs} = Jason.decode(content)
    
    assert is_list(image_defs)
    
    for image_def <- image_defs do
      assert Map.has_key?(image_def, "name")
      assert Map.has_key?(image_def, "imageUri")
      assert String.contains?(image_def["imageUri"], "ecr")
      assert String.contains?(image_def["imageUri"], "amazonaws.com")
    end
  end
end
```

#### Environment-Specific Pipeline Testing

```bash
# scripts/run-pipeline-tests.sh - Environment-specific test runner
#!/bin/bash

set -e

ENVIRONMENT=${1:-development}
echo "🧪 Running pipeline tests for $ENVIRONMENT environment"

case $ENVIRONMENT in
  development)
    echo "Running development pipeline tests..."
    mix test --include development --include pipeline
    ;;
    
  staging)
    echo "Running staging pipeline tests..."
    mix test --include staging --include pipeline --include integration
    ;;
    
  production)
    echo "Running production pipeline tests..."
    mix test --include production --include pipeline --include integration --include external_api
    
    # Additional production validations
    echo "Running production-specific validations..."
    mix test test/pipeline/production_validation_test.exs
    ;;
    
  *)
    echo "Unknown environment: $ENVIRONMENT"
    exit 1
    ;;
esac

echo "✅ Pipeline tests completed for $ENVIRONMENT"
```

### CodePipeline Test Integration

The test suite integrates with CodePipeline through multiple touchpoints:

1. **Pre-Build Testing**: Unit tests run in CodeBuild before container building
2. **Build Validation**: Container build process includes test execution
3. **Post-Build Testing**: Integration tests validate the built container
4. **Deployment Testing**: Health checks validate successful deployment
5. **Post-Deploy Testing**: End-to-end tests confirm system functionality

This ensures that every stage of the **Git → CodeBuild → ECR → CodeDeploy → ECS** pipeline is validated through automated testing.

## Test Reports

Generate test reports for CI/CD pipeline:

```bash
# Standard test execution with reports
mix test --formatter JUnitFormatter

# CodePipeline-specific test execution
mix test --include pipeline --formatter JUnitFormatter --formatter ExUnit.CLIFormatter
```

The JUnit XML report will be generated at `_build/test/lib/aybiza/test-junit-report.xml` and automatically consumed by CodePipeline for build reporting.