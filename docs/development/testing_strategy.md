# AYBIZA Comprehensive Testing Strategy

## Overview
This document defines the testing strategy for the AYBIZA AI voice agent platform, covering the hybrid Cloudflare+AWS architecture with real-time voice processing requirements.

## Testing Pyramid Architecture

```
                    🔺 E2E Tests
                   🔺🔺🔺 Integration Tests  
                 🔺🔺🔺🔺🔺 Unit Tests
               🔺🔺🔺🔺🔺🔺🔺 Edge Tests
```

## Test Categories

### 1. Unit Tests (70% of total tests)
**Target Coverage**: 90%+ line coverage, 85%+ branch coverage
**Technology**: ExUnit with Mox for mocking

#### Core Components Testing:
```elixir
# Example: Voice Pipeline Unit Test
defmodule VoicePipeline.AudioProcessorTest do
  use ExUnit.Case, async: true
  import Mox

  setup :verify_on_exit!

  describe "process_audio_frame/2" do
    test "processes μ-law audio frame correctly" do
      # Mock external dependencies
      expect(DeepgramMock, :send_audio, fn audio_data ->
        {:ok, %{transcript: "hello world"}}
      end)

      frame = generate_mulaw_frame()
      
      assert {:ok, result} = AudioProcessor.process_audio_frame(frame, :mulaw)
      assert result.format == :mulaw
      assert result.processed_at
    end

    test "handles corrupted audio frames gracefully" do
      corrupted_frame = <<0xFF, 0xFF, 0xFF>>
      
      assert {:error, :invalid_audio_format} = 
        AudioProcessor.process_audio_frame(corrupted_frame, :mulaw)
    end
  end
end
```

#### Testing Patterns:
- **GenServer State Testing**: Verify state transitions and side effects
- **Membrane Element Testing**: Test audio processing pipeline components  
- **Business Logic Testing**: Pure function testing with property-based tests
- **Error Handling Testing**: Comprehensive error scenario coverage

### 2. Edge Tests (15% of total tests)
**Technology**: Wrangler Testing Framework + Miniflare
**Purpose**: Test Cloudflare Workers functionality

#### Edge Worker Testing:
```javascript
// Example: Voice Router Worker Test
import { describe, it, expect, beforeEach } from 'vitest';
import { unstable_dev } from 'wrangler';

describe('Voice Router Worker', () => {
  let worker;

  beforeEach(async () => {
    worker = await unstable_dev('src/voice-router.js', {
      experimental: { disableExperimentalWarning: true },
    });
  });

  it('routes US calls to us-east-1', async () => {
    const request = new Request('https://api.aybiza.com/voice/inbound', {
      method: 'POST',
      headers: {
        'CF-IPCountry': 'US',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ call_sid: 'test-123' })
    });

    const response = await worker.fetch(request);
    const result = await response.json();
    
    expect(result.region).toBe('us-east-1');
    expect(response.headers.get('X-Routed-Region')).toBe('us-east-1');
  });

  it('caches agent configurations effectively', async () => {
    // Test edge caching behavior
    const agentConfigRequest = new Request('https://api.aybiza.com/agents/test-agent');
    
    // First request - cache miss
    const response1 = await worker.fetch(agentConfigRequest);
    expect(response1.headers.get('CF-Cache-Status')).toBe('MISS');
    
    // Second request - cache hit
    const response2 = await worker.fetch(agentConfigRequest);
    expect(response2.headers.get('CF-Cache-Status')).toBe('HIT');
  });
});
```

### 3. Integration Tests (10% of total tests)
**Technology**: ExUnit with Sandbox mode, Docker Compose for services

#### Service Integration Testing:
```elixir
defmodule AybizaWeb.VoiceGatewayIntegrationTest do
  use AybizaWeb.ConnCase
  import Phoenix.ChannelTest

  @endpoint AybizaWeb.Endpoint
  @moduletag :integration

  setup do
    # Start test services
    start_supervised!({VoiceGateway.CallSupervisor, []})
    start_supervised!({VoicePipeline.AudioProcessor, []})
    
    {:ok, socket} = connect(VoiceGateway.CallSocket, %{})
    {:ok, socket: socket}
  end

  test "complete voice processing pipeline", %{socket: socket} do
    # Simulate Twilio webhook
    call_params = %{
      "CallSid" => "test-call-123",
      "From" => "+1234567890",
      "To" => "+1987654321"
    }

    # Test call initiation
    {:ok, _call_pid} = VoiceGateway.start_call(call_params)
    
    # Simulate audio stream
    audio_frame = generate_test_audio_frame()
    ref = push(socket, "audio_frame", %{data: audio_frame})
    
    assert_reply ref, :ok, %{status: "processing"}
    
    # Verify pipeline processing
    assert_push "stt_result", %{transcript: transcript}
    assert String.length(transcript) > 0
    
    assert_push "llm_response", %{response: response}
    assert String.length(response) > 0
    
    assert_push "tts_audio", %{audio_data: audio_data}
    assert byte_size(audio_data) > 0
  end

  test "handles concurrent calls correctly" do
    # Test multiple concurrent call processing
    call_count = 10
    
    tasks = for i <- 1..call_count do
      Task.async(fn ->
        call_params = %{"CallSid" => "test-call-#{i}"}
        VoiceGateway.start_call(call_params)
      end)
    end
    
    results = Task.await_many(tasks, 5000)
    
    # Verify all calls started successfully
    assert Enum.all?(results, fn {:ok, _pid} -> true; _ -> false end)
    
    # Verify resource cleanup
    Process.sleep(1000)
    active_calls = VoiceGateway.CallSupervisor.count_children()
    assert active_calls.active <= call_count
  end
end
```

### 4. End-to-End Tests (5% of total tests)
**Technology**: Wallaby + Hound for browser automation, External service mocks

#### Real-World Scenario Testing:
```elixir
defmodule AybizaE2ETest do
  use ExUnit.Case
  use Wallaby.Feature

  @moduletag :e2e
  @moduletag timeout: 60_000

  feature "complete voice agent call flow", %{session: session} do
    # Setup test environment
    agent_config = create_test_agent()
    phone_number = acquire_test_phone_number()
    
    # Simulate incoming call via Twilio
    call_response = simulate_twilio_call(phone_number, agent_config.id)
    assert call_response.status == 200
    
    # Wait for call processing
    assert_call_connected(call_response.call_sid)
    
    # Test voice interaction
    send_audio_test_phrase("Hello, I need help with my account")
    
    # Verify agent response
    response_audio = receive_agent_response(timeout: 5000)
    assert byte_size(response_audio) > 0
    
    # Verify call analytics
    call_record = get_call_record(call_response.call_sid)
    assert call_record.status == "completed"
    assert call_record.duration > 0
    assert length(call_record.transcript) > 0
  end

  feature "multi-region failover", %{session: session} do
    # Test primary region failure
    simulate_region_failure("us-east-1")
    
    # Initiate call during failure
    call_response = simulate_twilio_call(test_phone_number(), test_agent_id())
    
    # Verify failover to secondary region
    assert call_response.headers["X-Processed-Region"] == "us-west-2"
    assert call_response.status == 200
    
    # Verify call quality maintained
    response_time = measure_response_time(call_response.call_sid)
    assert response_time < 500 # ms
  end
end
```

## Performance Testing

### 1. Load Testing
**Technology**: Artillery.io + K6 for HTTP load testing

#### Voice Pipeline Load Test:
```yaml
# artillery-voice-load.yml
config:
  target: 'https://api.aybiza.com'
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Ramp up"
    - duration: 300
      arrivalRate: 100
      name: "Sustained load"
    - duration: 60
      arrivalRate: 200
      name: "Peak load"

scenarios:
  - name: "Voice call simulation"
    weight: 100
    engine: ws
    processor: "./voice-load-processor.js"
```

```javascript
// voice-load-processor.js
module.exports = {
  setupCall: (context, events, done) => {
    context.vars.callSid = generateCallSid();
    context.vars.audioFrames = generateTestAudioFrames(10);
    return done();
  },
  
  processAudioFrames: (context, events, done) => {
    const ws = context.ws;
    context.vars.audioFrames.forEach((frame, index) => {
      setTimeout(() => {
        ws.send(JSON.stringify({
          type: 'audio_frame',
          data: frame,
          sequence: index
        }));
      }, index * 20); // 20ms intervals
    });
    return done();
  }
};
```

### 2. Stress Testing
**Purpose**: Test system behavior under extreme load
**Metrics**: Response time, error rate, resource utilization

#### Stress Test Scenarios:
- **Concurrent Calls**: 10,000+ simultaneous voice calls
- **Audio Processing**: High-frequency audio frame processing
- **Database Load**: Massive concurrent read/write operations
- **Memory Pressure**: Large context windows with multiple agents

### 3. Latency Testing
**Purpose**: Verify ultra-low latency requirements (<100ms)

```elixir
defmodule LatencyTest do
  use ExUnit.Case
  
  @tag :performance
  test "end-to-end voice processing latency" do
    # Measure complete pipeline latency
    start_time = System.monotonic_time(:millisecond)
    
    # Send audio frame
    audio_frame = generate_test_frame()
    {:ok, _result} = VoicePipeline.process_frame(audio_frame)
    
    end_time = System.monotonic_time(:millisecond)
    latency = end_time - start_time
    
    # Assert latency requirements
    assert latency < 100, "Latency #{latency}ms exceeds 100ms requirement"
  end
  
  @tag :performance  
  test "STT processing latency" do
    audio_data = load_test_audio("hello_world.wav")
    
    {time, {:ok, result}} = :timer.tc(fn ->
      DeepgramClient.transcribe(audio_data)
    end)
    
    latency_ms = time / 1000
    assert latency_ms < 50, "STT latency #{latency_ms}ms exceeds 50ms target"
  end
end
```

## Security Testing

### 1. Authentication Testing
```elixir
defmodule SecurityTest do
  use AybizaWeb.ConnCase
  
  test "rejects unauthenticated requests" do
    conn = build_conn()
    |> post("/api/v1/agents", %{name: "test"})
    
    assert conn.status == 401
    assert conn.resp_body =~ "authentication required"
  end
  
  test "validates JWT tokens properly" do
    # Test expired tokens
    expired_token = generate_expired_jwt()
    conn = build_conn()
    |> put_req_header("authorization", "Bearer #{expired_token}")
    |> get("/api/v1/agents")
    
    assert conn.status == 401
    
    # Test malformed tokens
    malformed_token = "invalid.jwt.token"
    conn = build_conn()
    |> put_req_header("authorization", "Bearer #{malformed_token}")
    |> get("/api/v1/agents")
    
    assert conn.status == 401
  end
end
```

### 2. Input Validation Testing
```elixir
defmodule InputValidationTest do
  use AybizaWeb.ConnCase
  
  test "sanitizes audio input data" do
    # Test malicious audio data
    malicious_audio = generate_malicious_audio_frame()
    
    result = VoicePipeline.AudioProcessor.process_frame(malicious_audio)
    assert {:error, :invalid_audio_format} = result
  end
  
  test "validates agent configuration" do
    invalid_config = %{
      "system_prompt" => String.duplicate("x", 100_000), # Too long
      "voice" => "invalid-voice-id"
    }
    
    conn = build_conn()
    |> authenticate_user()
    |> post("/api/v1/agents", invalid_config)
    
    assert conn.status == 422
    assert conn.resp_body =~ "system_prompt too long"
  end
end
```

## Test Infrastructure

### 1. Test Environment Setup
```yaml
# docker-compose.test.yml
version: '3.8'
services:
  postgres-test:
    image: postgres:16.9
    environment:
      POSTGRES_DB: aybiza_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5433:5432"
      
  redis-test:
    image: redis:8.0
    ports:
      - "6380:6379"
      
  deepgram-mock:
    image: wiremock/wiremock:latest
    ports:
      - "8081:8080"
    volumes:
      - ./test/mocks/deepgram:/home/wiremock
      
  twilio-mock:
    image: wiremock/wiremock:latest
    ports:
      - "8082:8080"
    volumes:
      - ./test/mocks/twilio:/home/wiremock
```

### 2. Continuous Integration Pipeline
```yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16.9
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
          
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Elixir
      uses: erlef/setup-beam@v1
      with:
        elixir-version: '1.18.3'
        otp-version: '27.3.4'
        
    - name: Install dependencies
      run: mix deps.get
      
    - name: Setup database
      run: mix ecto.setup
      
    - name: Run unit tests
      run: mix test --exclude integration --exclude e2e
      
    - name: Run integration tests
      run: mix test --only integration
      
    - name: Run edge tests
      run: |
        cd cloudflare-workers
        npm test
        
    - name: Generate coverage report
      run: mix coveralls.github
```

## Test Data Management

### 1. Test Fixtures
```elixir
defmodule AybizaWeb.Fixtures do
  def agent_fixture(attrs \\ %{}) do
    %{
      name: "Test Agent",
      system_prompt: "You are a helpful assistant",
      voice: "aura-asteria-en",
      language: "en-US"
    }
    |> Map.merge(attrs)
    |> Aybiza.AgentManager.create_agent()
  end
  
  def audio_frame_fixture(format \\ :mulaw) do
    case format do
      :mulaw -> File.read!("test/fixtures/audio/sample.mulaw")
      :linear16 -> File.read!("test/fixtures/audio/sample.wav")
    end
  end
end
```

### 2. Mock Services
```elixir
# Mock Deepgram API responses
Mox.defmock(DeepgramMock, for: Aybiza.VoicePipeline.DeepgramBehaviour)

defmodule DeepgramMockResponses do
  def stt_response do
    %{
      "results" => %{
        "channels" => [
          %{
            "alternatives" => [
              %{
                "transcript" => "Hello, how can I help you today?",
                "confidence" => 0.95
              }
            ]
          }
        ]
      }
    }
  end
  
  def tts_response do
    {:ok, File.read!("test/fixtures/audio/response.wav")}
  end
end
```

## Test Metrics and Reporting

### 1. Coverage Requirements
- **Unit Tests**: 90% line coverage, 85% branch coverage
- **Integration Tests**: 80% API endpoint coverage
- **E2E Tests**: 100% critical user journey coverage

### 2. Performance Benchmarks
- **Voice Processing Latency**: < 100ms (95th percentile)
- **API Response Time**: < 200ms (95th percentile)
- **Database Query Time**: < 50ms (95th percentile)
- **Concurrent Call Capacity**: 10,000+ calls per node

### 3. Quality Gates
- All tests must pass before merge
- Coverage requirements must be met
- Performance benchmarks must be maintained
- Security tests must pass
- No critical or high severity vulnerabilities

## Test Maintenance

### 1. Test Review Process
- Code reviews must include test review
- New features require corresponding tests
- Test failures must be addressed immediately
- Regular test suite maintenance and optimization

### 2. Test Environment Maintenance
- Regular updates to test dependencies
- Mock service updates for API changes
- Test data refresh and cleanup
- Performance baseline updates

This comprehensive testing strategy ensures the AYBIZA platform maintains high quality, performance, and security standards across all components of the hybrid architecture.