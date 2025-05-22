# Membrane Framework Integration Guide for Claude Code

## Introduction

This guide provides comprehensive instructions for using Claude Code to implement and interact with the Membrane Framework in the AYBIZA AI Voice Agent Platform. The Membrane Framework is a critical component of AYBIZA's voice pipeline, enabling ultra-low latency real-time audio processing.

## Membrane Framework Overview

[Membrane Framework](https://membraneframework.org/) is an Elixir multimedia processing framework optimized for real-time, low-latency streaming. In AYBIZA, it powers the voice pipeline with the following capabilities:

- Audio streaming to/from Twilio WebSockets
- Voice activity detection (VAD)
- Audio quality monitoring
- Buffer management for jitter compensation
- Integration with Deepgram STT and TTS services
- Integration with AWS Bedrock Claude LLM

## Key Membrane Concepts for Claude Code

As a Claude Code user implementing AYBIZA components, you need to understand these core Membrane concepts:

### 1. Pipeline Structure

Membrane organizes media processing into pipelines composed of connected elements:

- **Source Elements**: Produce data (e.g., WebSocket audio source from Twilio)
- **Filter Elements**: Transform data (e.g., Voice Activity Detector)
- **Sink Elements**: Consume data (e.g., Deepgram STT sink)

Example pipeline creation:

```elixir
def create_pipeline(call_id) do
  Pipeline.start_link(%Pipeline.Options{
    elements: [
      # Define elements here
      source: %Source.WebSocket{
        connection: {:twilio, call_id},
        format: :mulaw,
        sample_rate: 8000
      },
      vad: %Filter.VAD{
        threshold: 1000,
        frame_duration: 20
      },
      sink: %Sink.Deepgram{
        api_key: System.get_env("DEEPGRAM_API_KEY"),
        model: "nova-3"
      }
    ],
    
    # Link elements together
    links: [
      link(:source) |> to(:vad) |> to(:sink)
    ]
  })
end
```

### 2. Element Types and Implementation

Claude Code should implement Membrane elements using the appropriate behaviours:

```elixir
defmodule Aybiza.VoicePipeline.VADDetector do
  use Membrane.Filter
  
  def_input_pad :input,
    accepted_format: %Membrane.RawAudio{
      sample_format: :s16le,
      channels: 1,
      sample_rate: 8000
    }
    
  def_output_pad :output,
    accepted_format: %Membrane.RawAudio{
      sample_format: :s16le,
      channels: 1,
      sample_rate: 8000
    }
    
  # Implement required callbacks
  @impl true
  def handle_init(_ctx, options) do
    state = %{
      threshold: options.threshold,
      frame_duration: options.frame_duration,
      voice_detected: false
    }
    {[], state}
  end
  
  @impl true
  def handle_buffer(:input, buffer, _ctx, state) do
    # Process audio buffer
    {is_voice, energy} = detect_voice_activity(buffer.payload)
    
    # Determine if state changed
    state_changed = is_voice != state.voice_detected
    
    # Update state and possibly emit events
    if state_changed do
      event = if is_voice do
        %Aybiza.VoicePipeline.Events.VoiceActivityStarted{}
      else
        %Aybiza.VoicePipeline.Events.VoiceActivityEnded{}
      end
      
      actions = [
        {:buffer, {:output, buffer}},
        {:event, {:output, event}}
      ]
      
      {actions, %{state | voice_detected: is_voice}}
    else
      {[buffer: {:output, buffer}], state}
    end
  end
end
```

### 3. Link Patterns and Flow Control

Claude Code must understand how to properly link elements and control data flow:

```elixir
# Basic linear pipeline
link(:source) |> to(:filter) |> to(:sink)

# Branching pipeline
link(:source)
|> via_in(:input) |> to(:filter1)
|> via_out(:output) |> to(:sink1)

link(:source)
|> via_in(:input) |> to(:filter2)
|> via_out(:output) |> to(:sink2)

# Dynamic linking
link(:source)
|> via_in(:input, options: [stream_format: %Membrane.RawAudio{
  sample_format: :s16le,
  channels: 1,
  sample_rate: 8000
}])
|> to(:sink)
```

## Implementation Patterns for Voice Pipeline Elements

Claude Code should implement AYBIZA voice pipeline elements following these patterns:

### 1. WebSocket Audio Source

```elixir
defmodule Aybiza.VoicePipeline.TwilioWebSocketSource do
  use Membrane.Source
  
  def_output_pad :output,
    accepted_format: %Membrane.RawAudio{
      sample_format: :mulaw,
      channels: 1,
      sample_rate: 8000
    }
  
  # Implementation for handling Twilio WebSocket frames
  # See apps/voice_pipeline/lib/voice_pipeline/audio_input.ex for reference
end
```

### 2. Voice Activity Detector

```elixir
defmodule Aybiza.VoicePipeline.VADDetector do
  use Membrane.Filter
  
  # Implement filter for detecting voice activity
  # Must emit VoiceActivityStarted and VoiceActivityEnded events
  # See apps/voice_pipeline/lib/voice_pipeline/vad.ex for reference
end
```

### 3. Deepgram STT Integration

```elixir
defmodule Aybiza.VoicePipeline.DeepgramSTT do
  use Membrane.Sink
  
  def_input_pad :input,
    accepted_format: %Membrane.RawAudio{
      sample_format: :mulaw,
      channels: 1,
      sample_rate: 8000
    }
  
  # Implementation for sending audio to Deepgram
  # Must handle streaming API with interim results
  # See apps/voice_pipeline/lib/voice_pipeline/deepgram_stt.ex for reference
end
```

### 4. Deepgram TTS Integration

```elixir
defmodule Aybiza.VoicePipeline.DeepgramTTS do
  use Membrane.Source
  
  def_output_pad :output,
    accepted_format: %Membrane.RawAudio{
      sample_format: :mulaw,
      channels: 1,
      sample_rate: 8000
    }
  
  # Implementation for receiving audio from Deepgram TTS
  # Must handle streaming with low latency
  # See apps/voice_pipeline/lib/voice_pipeline/deepgram_tts.ex for reference
end
```

## Performance Optimizations

Claude Code must implement these performance optimizations to meet AYBIZA's ultra-low latency requirements:

### 1. Direct μ-law Processing

AYBIZA's voice pipeline is optimized for direct μ-law processing without format conversion:

```elixir
# INCORRECT - Converts to PCM then back to μ-law
defmodule Aybiza.VoicePipeline.Incorrect do
  def process_audio(mulaw_data) do
    pcm_data = Mulaw.decode(mulaw_data)
    processed = apply_processing(pcm_data)
    Mulaw.encode(processed)
  end
end

# CORRECT - Direct μ-law processing
defmodule Aybiza.VoicePipeline.Correct do
  def process_audio(mulaw_data) do
    # Process directly in μ-law format
    apply_mulaw_processing(mulaw_data)
  end
end
```

### 2. Parallel Processing with Tasks

Use Task.async for parallel processing of audio chunks:

```elixir
def process_chunks(chunks) do
  tasks = Enum.map(chunks, fn chunk ->
    Task.async(fn -> process_chunk(chunk) end)
  end)
  
  # Wait for all tasks with timeout
  Task.yield_many(tasks, 50)
  |> Enum.map(fn {task, result} ->
    case result do
      {:ok, result} -> result
      nil ->
        Task.shutdown(task, :brutal_kill)
        {:error, :timeout}
    end
  end)
end
```

### 3. Buffer Management

Implement adaptive buffer management:

```elixir
defmodule Aybiza.VoicePipeline.BufferManager do
  use Membrane.Filter
  
  def_options target_buffer_ms: [
    spec: non_neg_integer(),
    default: 100,
    description: "Target buffer size in ms"
  ]
  
  def_options max_buffer_ms: [
    spec: non_neg_integer(),
    default: 500,
    description: "Maximum buffer size in ms"
  ]
  
  # Implementation for adaptive buffer management
  # See apps/voice_pipeline/lib/voice_pipeline/buffer_manager.ex for reference
end
```

## Testing Patterns for Membrane Framework

Claude Code should implement these testing patterns for Membrane components:

### 1. Element Unit Tests

```elixir
defmodule Aybiza.VoicePipeline.VADDetectorTest do
  use ExUnit.Case
  
  test "detects voice activity correctly" do
    # Create test audio buffer with voice
    buffer_with_voice = %Membrane.Buffer{
      payload: create_test_audio_with_voice()
    }
    
    # Create detector with test state
    state = %{threshold: 1000, frame_duration: 20, voice_detected: false}
    
    # Call handle_buffer directly
    {actions, new_state} = Aybiza.VoicePipeline.VADDetector.handle_buffer(
      :input, buffer_with_voice, %{}, state
    )
    
    # Assert that voice was detected
    assert new_state.voice_detected == true
    
    # Assert that VoiceActivityStarted event was emitted
    assert Enum.any?(actions, fn
      {:event, {:output, %Aybiza.VoicePipeline.Events.VoiceActivityStarted{}}} -> true
      _ -> false
    end)
  end
end
```

### 2. Pipeline Integration Tests

```elixir
defmodule Aybiza.VoicePipeline.PipelineTest do
  use ExUnit.Case
  
  test "pipeline processes audio correctly" do
    # Create test pipeline
    {:ok, pipeline} = Aybiza.VoicePipeline.create_test_pipeline()
    
    # Send test data
    Membrane.Pipeline.message(pipeline, :source, {:buffer, test_audio_data()})
    
    # Wait for processing
    Process.sleep(100)
    
    # Get results from test sink
    results = Aybiza.Test.Sink.get_buffers(pipeline, :sink)
    
    # Assert correct processing
    assert length(results) > 0
  end
end
```

### 3. Performance Testing

```elixir
defmodule Aybiza.VoicePipeline.PerformanceTest do
  use ExUnit.Case
  
  @tag :performance
  test "meets latency requirements" do
    # Create test pipeline
    {:ok, pipeline} = Aybiza.VoicePipeline.create_test_pipeline()
    
    # Measure processing time
    start_time = System.monotonic_time(:millisecond)
    
    # Send test data
    Membrane.Pipeline.message(pipeline, :source, {:buffer, test_audio_data()})
    
    # Wait for first result
    results = Aybiza.Test.Sink.wait_for_buffers(pipeline, :sink, 1, 100)
    
    end_time = System.monotonic_time(:millisecond)
    processing_time = end_time - start_time
    
    # Assert latency requirement
    assert processing_time < 50, "Processing time exceeded 50ms: #{processing_time}ms"
  end
end
```

## Common Pitfalls and Solutions

Claude Code should avoid these common pitfalls when implementing Membrane components:

### 1. Blocking Operations in Element Callbacks

**Problem**: Blocking operations in element callbacks stall the entire pipeline.

**Solution**: Use async operations with message callbacks:

```elixir
# INCORRECT - Blocking API call
def handle_buffer(:input, buffer, _ctx, state) do
  result = HTTPoison.post!(api_url, buffer.payload)
  # Process result...
end

# CORRECT - Non-blocking with message handling
def handle_buffer(:input, buffer, _ctx, state) do
  Task.start(fn ->
    result = HTTPoison.post!(api_url, buffer.payload)
    send(self(), {:api_result, result})
  end)
  
  {[buffer: {:output, buffer}], state}
end

def handle_info({:api_result, result}, _ctx, state) do
  # Process result...
  {[], state}
end
```

### 2. Improper Buffer Handling

**Problem**: Not respecting buffer continuity or timestamps.

**Solution**: Preserve buffer metadata:

```elixir
# INCORRECT - Loses buffer metadata
def handle_buffer(:input, buffer, _ctx, state) do
  new_payload = process_data(buffer.payload)
  new_buffer = %Membrane.Buffer{payload: new_payload}
  
  {[buffer: {:output, new_buffer}], state}
end

# CORRECT - Preserves buffer metadata
def handle_buffer(:input, buffer, _ctx, state) do
  new_payload = process_data(buffer.payload)
  new_buffer = %{buffer | payload: new_payload}
  
  {[buffer: {:output, new_buffer}], state}
end
```

### 3. Memory Management Issues

**Problem**: Creating large binaries or memory leaks.

**Solution**: Use proper binary operations and garbage collection:

```elixir
# INCORRECT - Creates unnecessary binary copies
def handle_buffer(:input, buffer, _ctx, state) do
  # This creates a full copy of the binary
  processed = buffer.payload <> <<0>>
  
  {[buffer: {:output, %{buffer | payload: processed}}], state}
end

# CORRECT - Uses binary references efficiently
def handle_buffer(:input, buffer, _ctx, state) do
  # More efficient binary handling
  processed = process_without_copying(buffer.payload)
  
  # Trigger garbage collection if needed
  if large_processing_done?(state) do
    :erlang.garbage_collect()
  end
  
  {[buffer: {:output, %{buffer | payload: processed}}], state}
end
```

## Membrane Framework Debugging

Claude Code should implement these debugging strategies for Membrane components:

### 1. Enabling Debug Logs

```elixir
# Enable debug logs for pipeline
config :logger, :console, level: :debug

# Enable element-specific logging
Pipeline.start_link(%Pipeline.Options{
  elements: [
    source: %Source.WebSocket{debug: true}
  ]
})
```

### 2. Tracing Element Events

```elixir
# Add notification element to trace events
link(:source)
|> to(:notifier, %Membrane.Debug.Notifier{name: "source_output"})
|> to(:filter)
```

### 3. Monitoring Buffer Flow

```elixir
defmodule Aybiza.Debug.BufferMonitor do
  use Membrane.Filter
  
  require Logger
  
  def_input_pad :input, accepted_format: _any
  def_output_pad :output, accepted_format: _any
  
  @impl true
  def handle_buffer(:input, buffer, _ctx, state) do
    Logger.debug("Buffer received: size=#{byte_size(buffer.payload)}, pts=#{buffer.pts}")
    {[buffer: {:output, buffer}], state}
  end
end
```

## Integration with AWS Bedrock Claude Models

Claude Code must implement proper integration between Membrane Framework and AWS Bedrock Claude models:

```elixir
defmodule Aybiza.VoicePipeline.BedrockLLM do
  use GenServer
  require Logger
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: via_tuple(opts.call_id))
  end
  
  def init(opts) do
    # Subscribe to STT events
    Phoenix.PubSub.subscribe(Aybiza.PubSub, "stt:#{opts.call_id}")
    
    {:ok, %{
      call_id: opts.call_id,
      transcripts: [],
      context: opts.context,
      bedrock_client: Aybiza.AWS.BedrockClient.new(opts.tenant_id)
    }}
  end
  
  def handle_info({:transcript, transcript}, state) do
    # Handle new transcript from STT
    new_state = %{state | transcripts: [transcript | state.transcripts]}
    
    if transcript.is_final do
      # Send to Bedrock Claude
      Task.start(fn ->
        response = state.bedrock_client.invoke(%{
          model_id: "anthropic.claude-3-5-sonnet-20240229-v1:0",
          prompt: build_prompt(new_state),
          max_tokens: 1000,
          temperature: 0.7
        })
        
        send(self(), {:llm_response, response})
      end)
    end
    
    {:noreply, new_state}
  end
  
  def handle_info({:llm_response, response}, state) do
    # Handle LLM response and send to TTS
    Phoenix.PubSub.broadcast(
      Aybiza.PubSub,
      "tts:#{state.call_id}",
      {:tts_request, response.completion}
    )
    
    {:noreply, state}
  end
  
  # Helper functions...
end
```

## Real-Time Audio Processing Requirements

Claude Code must ensure implementations meet these performance requirements:

- Voice Activity Detection: < 10ms
- STT First Result: < 50ms
- LLM First Token: < 200ms
- TTS First Audio: < 50ms
- End-to-End Latency: < 300ms

## Conclusion

By following this guide, Claude Code can successfully implement Membrane Framework components for AYBIZA's voice pipeline, ensuring proper integration, performance, and fault tolerance. Use the reference implementations and testing patterns to create high-quality, production-ready code.

## References

- [Membrane Framework Documentation](https://membraneframework.org/guide)
- [AYBIZA Voice Pipeline Architecture](08_voice_pipeline_architecture.md)
- [AWS Bedrock Claude Implementation](11_aws_bedrock_claude_implementation.md)
- [Deepgram Integration Guide](13_deepgram_integration_guide.md)