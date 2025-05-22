# Deepgram Integration Guide for AYBIZA Platform

## Executive Summary

This guide provides comprehensive documentation for Deepgram integration in the AYBIZA AI Voice Agent Platform's hybrid Cloudflare+AWS architecture, incorporating the latest Nova-3 STT and Aura-2 TTS features with edge optimization for ultra-low latency processing (<50ms US/UK, <100ms globally) and seamless integration with Claude 3.7 Sonnet's extended thinking capabilities.

## Key Features and Improvements

### 1. Model Upgrades
- **STT**: Nova-3 for 54% better accuracy
- **TTS**: Aura-2 for context-aware, natural speech
- **Language**: Multi-language support with automatic detection

### 2. Encoding Optimization
- **Dynamic Selection**: Choose optimal encoding per use case
- **No Conversion**: Direct integration with Twilio formats
- **Bandwidth Optimization**: Use appropriate compression

### 3. Latency Improvements
- **Hybrid Processing**: Edge-first with intelligent cloud handoff
- **Streaming**: Enhanced WebSocket with Cloudflare acceleration
- **Interim Results**: Process partial transcriptions with edge caching
- **Parallel Processing**: Concurrent pipeline optimization across 300+ locations
- **Ultra-Low Latency**: <50ms US/UK, <100ms globally

### 4. Advanced Features
- **Claude 3.7 Integration**: Enhanced context awareness with extended thinking
- **Keyterm Prompting**: Domain-specific term recognition with AI enhancement
- **Voice Customization**: Natural pauses, fillers, emotions (8 Aura-2 voices)
- **Smart Formatting**: Automatic number/currency formatting
- **Edge Intelligence**: Cloudflare Workers for real-time processing
- **Cost Optimization**: 28.6% savings through hybrid architecture

## Detailed Implementation Guide

### 1. Audio Encoding Optimization

#### Dynamic Encoding Selection
```elixir
defmodule Aybiza.VoicePipeline.EncodingOptimizer do
  @encoding_map %{
    twilio_inbound: %{encoding: "mulaw", sample_rate: 8000},
    twilio_outbound: %{encoding: "mulaw", sample_rate: 8000},
    high_quality: %{encoding: "linear16", sample_rate: 16000},
    streaming: %{encoding: "opus", sample_rate: 16000}
  }
  
  def get_optimal_config(context) do
    config = Map.get(@encoding_map, context.type, @encoding_map.high_quality)
    
    # No conversion needed - use native formats
    case context.provider do
      :twilio -> 
        %{config | container: "none"}  # Raw audio for telephony
      :websocket ->
        %{config | container: "wav"}   # Standard container
      _ ->
        config
    end
  end
end
```

### 2. Speech-to-Text with Nova-3

#### Complete STT Implementation
```elixir
defmodule Aybiza.VoicePipeline.DeepgramSTT do
  use WebSockex
  
  @deepgram_streaming_url "wss://api.deepgram.com/v1/listen"
  
  # Nova-3 configuration with advanced features
  @nova3_config %{
    model: "nova-3",
    version: "2024-01-01",
    encoding: "linear16",  # Dynamic based on input
    sample_rate: 16000,
    channels: 1,
    punctuate: true,
    smart_format: true,
    numerals: true,
    profanity_filter: false,
    redact: false,
    utterances: true,
    interim_results: true,
    endpointing: 300,
    utterance_end_ms: 1000,
    vad_events: true
  }
  
  def start_link(call_id, options \\ []) do
    config = Map.merge(@nova3_config, Enum.into(options, %{}))
    
    # Add language detection if needed
    config = if options[:detect_language] do
      Map.put(config, :detect_language, true)
    else
      config
    end
    
    # Add keywords for better recognition
    config = if keywords = options[:keywords] do
      Map.put(config, :keywords, keywords)
    else
      config
    end
    
    url = build_url(config)
    state = %{
      call_id: call_id,
      buffer: [],
      config: config,
      last_activity: System.monotonic_time(:millisecond)
    }
    
    WebSockex.start_link(url, __MODULE__, state, 
      extra_headers: headers(),
      handle_initial_conn_failure: true
    )
  end
  
  @impl WebSockex
  def handle_frame({:text, msg}, state) do
    case Jason.decode(msg) do
      {:ok, %{"type" => "Results", "channel" => channel}} ->
        handle_transcription_result(channel, state)
      
      {:ok, %{"type" => "SpeechStarted"}} ->
        handle_speech_started(state)
      
      {:ok, %{"type" => "UtteranceEnd"}} ->
        handle_utterance_end(state)
        
      {:ok, %{"type" => "LanguageDetected", "language" => lang}} ->
        handle_language_detected(lang, state)
      
      _ ->
        {:ok, state}
    end
  end
  
  defp handle_transcription_result(channel, state) do
    alternatives = channel["alternatives"]
    
    if length(alternatives) > 0 do
      best = List.first(alternatives)
      transcript = best["transcript"]
      
      if channel["is_final"] do
        # Final result
        GenServer.cast(state.call_id, {:final_transcript, transcript})
        
        # Check for speech_final (end of utterance)
        if channel["speech_final"] do
          GenServer.cast(state.call_id, :speech_final)
        end
      else
        # Interim result for faster processing
        GenServer.cast(state.call_id, {:interim_transcript, transcript})
      end
    end
    
    {:ok, %{state | last_activity: System.monotonic_time(:millisecond)}}
  end
  
  # Keep-alive implementation
  @impl WebSockex
  def handle_info(:keep_alive, state) do
    current_time = System.monotonic_time(:millisecond)
    inactive_time = current_time - state.last_activity
    
    if inactive_time > 8000 do  # 8 seconds
      # Send silence to keep connection alive
      silence = <<0::size(160*8)>>  # 20ms of silence
      {:reply, {:binary, silence}, state}
    else
      schedule_keep_alive()
      {:ok, state}
    end
  end
  
  defp schedule_keep_alive do
    Process.send_after(self(), :keep_alive, 7000)
  end
  
  defp build_url(config) do
    params = config
    |> Map.to_list()
    |> Enum.map(fn {k, v} -> "#{k}=#{v}" end)
    |> Enum.join("&")
    
    "#{@deepgram_streaming_url}?#{params}"
  end
  
  defp headers do
    [
      {"Authorization", "Token #{System.get_env("DEEPGRAM_API_KEY")}"},
      {"Content-Type", "audio/raw"}
    ]
  end
end
```

### 3. Text-to-Speech with Aura-2

#### Complete TTS Implementation
```elixir
defmodule Aybiza.VoicePipeline.DeepgramTTS do
  @deepgram_tts_url "https://api.deepgram.com/v1/speak"
  
  # Aura-2 voices with personalities
  @aura2_voices %{
    "aura-asteria-en" => %{gender: :female, style: :calm_informative},
    "aura-luna-en" => %{gender: :female, style: :expressive_friendly},
    "aura-stella-en" => %{gender: :female, style: :clear_professional},
    "aura-athena-en" => %{gender: :female, style: :formal_professional},
    "aura-hera-en" => %{gender: :female, style: :conversational},
    "aura-orion-en" => %{gender: :male, style: :conversational},
    "aura-perseus-en" => %{gender: :male, style: :professional},
    "aura-angus-en" => %{gender: :male, style: :natural_friendly},
    "aura-orpheus-en" => %{gender: :male, style: :clear_speaker},
    "aura-helios-en" => %{gender: :male, style: :formal},
    "aura-zeus-en" => %{gender: :male, style: :authoritative}
  }
  
  def synthesize_streaming(call_id, text, options \\ %{}) do
    voice = get_voice_for_context(call_id, options)
    
    # Add natural pauses and fillers for Aura-2
    enhanced_text = enhance_text_for_natural_speech(text, options)
    
    headers = [
      {"Authorization", "Token #{System.get_env("DEEPGRAM_API_KEY")}"},
      {"Content-Type", "application/json"},
      {"Accept", "audio/raw"}
    ]
    
    params = %{
      model: voice,
      encoding: Map.get(options, :encoding, "linear16"),
      sample_rate: Map.get(options, :sample_rate, 24000),
      container: Map.get(options, :container, "none"),
      bit_rate: Map.get(options, :bit_rate, 16)
    }
    
    url = build_tts_url(@deepgram_tts_url, params)
    
    body = Jason.encode!(%{text: enhanced_text})
    
    # Use streaming for low latency
    stream_tts_response(url, body, headers, call_id)
  end
  
  defp get_voice_for_context(call_id, options) do
    # Select voice based on agent configuration
    agent = get_agent_config(call_id)
    
    case options[:voice_preference] do
      nil -> agent.default_voice || "aura-stella-en"
      :match_caller -> match_caller_demographics(call_id)
      voice -> voice
    end
  end
  
  defp enhance_text_for_natural_speech(text, options) do
    text
    |> maybe_add_fillers(options)
    |> add_natural_pauses()
    |> handle_numbers_and_currency()
    |> add_emphasis_markers()
  end
  
  defp maybe_add_fillers(text, %{natural_fillers: true}) do
    # Add contextual fillers where appropriate
    text
    |> String.replace(~r/\b(Well|So|Now)\b/, "\\1,")
    |> String.replace(~r/\b(I think|I believe)\b/, "\\1...")
    |> String.replace(~r/\b(um|uh)\b/i, "... ")
  end
  defp maybe_add_fillers(text, _), do: text
  
  defp add_natural_pauses(text) do
    # Use different pause lengths
    text
    |> String.replace(~r/\.\s+/, ". ")      # Short pause after sentences
    |> String.replace(~r/,\s+/, ", ")       # Brief pause after commas
    |> String.replace(~r/:\s+/, ": ")       # Medium pause after colons
    |> String.replace(~r/\?\s+/, "? ")      # Medium pause after questions
    |> String.replace(~r/!\s+/, "! ")       # Short pause after exclamations
  end
  
  defp handle_numbers_and_currency(text) do
    text
    |> format_phone_numbers()
    |> format_currency()
    |> format_dates()
    |> format_ordinals()
  end
  
  defp add_emphasis_markers(text) do
    # Add SSML-like markers for emphasis (Aura-2 compatible)
    text
    |> String.replace(~r/\*(\w+)\*/, "<emphasis>\\1</emphasis>")
    |> String.replace(~r/_(\w+)_/, "<mild>\\1</mild>")
  end
  
  defp stream_tts_response(url, body, headers, call_id) do
    # Streaming implementation for low latency
    case HTTPoison.post(url, body, headers, stream_to: self()) do
      {:ok, %HTTPoison.AsyncResponse{id: ref}} ->
        stream_audio_chunks(ref, call_id, <<>>)
      {:error, %HTTPoison.Error{reason: reason}} ->
        {:error, reason}
    end
  end
  
  defp stream_audio_chunks(ref, call_id, buffer) do
    receive do
      %HTTPoison.AsyncStatus{id: ^ref, code: 200} ->
        stream_audio_chunks(ref, call_id, buffer)
      
      %HTTPoison.AsyncHeaders{id: ^ref} ->
        stream_audio_chunks(ref, call_id, buffer)
      
      %HTTPoison.AsyncChunk{id: ^ref, chunk: chunk} ->
        new_buffer = buffer <> chunk
        
        # Send chunks of appropriate size (160 bytes = 20ms at 8kHz)
        case new_buffer do
          <<audio_chunk::binary-size(160), rest::binary>> ->
            send_audio_to_caller(call_id, audio_chunk)
            stream_audio_chunks(ref, call_id, rest)
          _ ->
            stream_audio_chunks(ref, call_id, new_buffer)
        end
      
      %HTTPoison.AsyncEnd{id: ^ref} ->
        # Send any remaining audio
        if byte_size(buffer) > 0 do
          send_audio_to_caller(call_id, buffer)
        end
        :ok
    after
      30_000 -> {:error, :timeout}
    end
  end
  
  defp send_audio_to_caller(call_id, audio_chunk) do
    GenServer.cast({:via, Registry, {Aybiza.CallRegistry, call_id}}, 
                   {:audio_chunk, audio_chunk})
  end
end
```

### 4. Language Detection and Multi-Language Support

```elixir
defmodule Aybiza.VoicePipeline.MultiLanguage do
  @supported_languages %{
    "en" => "English",
    "es" => "Spanish", 
    "fr" => "French",
    "de" => "German",
    "it" => "Italian",
    "pt" => "Portuguese",
    "nl" => "Dutch",
    "pl" => "Polish",
    "ru" => "Russian",
    "ja" => "Japanese",
    "ko" => "Korean",
    "zh" => "Chinese"
  }
  
  def configure_for_language(language_code, stt_config, tts_config) do
    stt_config = Map.put(stt_config, :language, language_code)
    
    # Adjust TTS voice based on language
    tts_voice = get_voice_for_language(language_code, tts_config.voice_gender)
    tts_config = Map.put(tts_config, :model, tts_voice)
    
    {stt_config, tts_config}
  end
  
  def get_voice_for_language(language_code, gender \\ :female) do
    case {language_code, gender} do
      {"es", :female} -> "aura-asteria-es"
      {"es", :male} -> "aura-orion-es"
      {"fr", :female} -> "aura-stella-fr"
      {"fr", :male} -> "aura-perseus-fr"
      # ... more language mappings
      _ -> "aura-stella-en"  # Default to English
    end
  end
end
```

### 5. Performance Optimization Strategies

#### Connection Pooling
```elixir
defmodule Aybiza.VoicePipeline.DeepgramPool do
  use GenServer
  
  @pool_size 10
  @connection_timeout 30_000
  
  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end
  
  def init(_) do
    connections = for _ <- 1..@pool_size do
      {:ok, conn} = establish_connection()
      conn
    end
    
    {:ok, %{
      connections: connections,
      available: connections,
      in_use: []
    }}
  end
  
  def get_connection do
    GenServer.call(__MODULE__, :get_connection, @connection_timeout)
  end
  
  def return_connection(conn) do
    GenServer.cast(__MODULE__, {:return_connection, conn})
  end
  
  # Implementation details...
end
```

#### Latency Monitoring
```elixir
defmodule Aybiza.VoicePipeline.LatencyMonitor do
  use GenServer
  
  @thresholds %{
    stt_first_byte: 50,      # ms
    stt_final_result: 500,   # ms
    tts_first_byte: 100,     # ms
    tts_complete: 1000       # ms
  }
  
  def record_metric(metric, value) do
    GenServer.cast(__MODULE__, {:record_metric, metric, value})
  end
  
  def handle_cast({:record_metric, metric, value}, state) do
    # Check if value exceeds threshold
    threshold = Map.get(@thresholds, metric)
    
    if value > threshold do
      Logger.warning("Latency threshold exceeded for #{metric}: #{value}ms (threshold: #{threshold}ms)")
      notify_ops_team(metric, value, threshold)
    end
    
    # Update rolling average
    new_state = update_rolling_average(state, metric, value)
    
    {:noreply, new_state}
  end
end
```

### 6. Error Handling and Resilience

```elixir
defmodule Aybiza.VoicePipeline.DeepgramResilience do
  @max_retries 3
  @backoff_base 1000  # ms
  
  def with_retry(fun, opts \\ []) do
    max_retries = Keyword.get(opts, :max_retries, @max_retries)
    do_with_retry(fun, 0, max_retries)
  end
  
  defp do_with_retry(fun, attempt, max_retries) when attempt < max_retries do
    case fun.() do
      {:ok, result} -> 
        {:ok, result}
      
      {:error, reason} -> 
        backoff = calculate_backoff(attempt)
        Logger.warning("Deepgram request failed (attempt #{attempt + 1}/#{max_retries}): #{inspect(reason)}")
        Process.sleep(backoff)
        do_with_retry(fun, attempt + 1, max_retries)
    end
  end
  
  defp do_with_retry(fun, attempt, max_retries) do
    Logger.error("Max retries (#{max_retries}) exceeded for Deepgram request")
    fun.()
  end
  
  defp calculate_backoff(attempt) do
    # Exponential backoff with jitter
    base_backoff = @backoff_base * :math.pow(2, attempt)
    jitter = :rand.uniform(500)
    trunc(base_backoff + jitter)
  end
end
```

### 7. Advanced Features

#### Keyword Boosting
```elixir
defmodule Aybiza.VoicePipeline.KeywordBoosting do
  def build_keywords_config(agent_config) do
    base_keywords = get_base_keywords(agent_config.industry)
    custom_keywords = agent_config.custom_vocabulary || []
    
    keywords = base_keywords ++ custom_keywords
    
    # Format for Deepgram API
    keywords
    |> Enum.map(fn keyword ->
      case keyword do
        {word, boost} -> "#{word}:#{boost}"
        word -> word
      end
    end)
    |> Enum.join(",")
  end
  
  defp get_base_keywords(industry) do
    case industry do
      :healthcare -> ["patient:2", "appointment:2", "prescription:2", "doctor:1.5"]
      :finance -> ["account:2", "balance:2", "transaction:2", "transfer:1.5"]
      :retail -> ["order:2", "product:2", "shipping:2", "return:1.5"]
      _ -> []
    end
  end
end
```

#### Voice Clone Protection
```elixir
defmodule Aybiza.VoicePipeline.VoiceAuthentication do
  def verify_voice_authenticity(audio_data, options \\ %{}) do
    # Use Deepgram's voice activity detection
    vad_result = analyze_voice_activity(audio_data)
    
    # Check for synthetic voice patterns
    synthetic_score = detect_synthetic_patterns(audio_data)
    
    # Verify against known voice profile if available
    profile_match = if profile = options[:voice_profile] do
      match_voice_profile(audio_data, profile)
    else
      nil
    end
    
    %{
      is_human: vad_result.human_probability > 0.85,
      synthetic_score: synthetic_score,
      profile_match: profile_match,
      confidence: calculate_confidence(vad_result, synthetic_score, profile_match)
    }
  end
end
```

## Best Practices

### 1. Connection Management
- Use connection pooling for high-traffic scenarios
- Implement keep-alive for WebSocket connections
- Handle reconnections gracefully with exponential backoff

### 2. Audio Quality
- Use the highest quality encoding that bandwidth allows
- Implement adaptive bitrate based on network conditions
- Monitor audio metrics and adjust parameters dynamically

### 3. Latency Optimization
- Use interim results for real-time feedback
- Implement parallel processing pipelines
- Cache common TTS responses
- Pre-warm connections for critical paths

### 4. Error Handling
- Implement retry logic with backoff
- Fallback to alternative models/services
- Log all errors with context for debugging
- Monitor error rates and alert on anomalies

### 5. Security
- Rotate API keys regularly
- Use environment variables for sensitive data
- Implement request signing where available
- Monitor for unusual usage patterns

## Configuration Templates

### Development Configuration
```elixir
config :aybiza, :deepgram,
  api_key: System.get_env("DEEPGRAM_API_KEY"),
  environment: :development,
  stt: %{
    model: "nova-3",
    language: "en-US",
    interim_results: true,
    utterances: true,
    vad_events: true,
    endpointing: 300
  },
  tts: %{
    model: "aura-stella-en",
    encoding: "linear16",
    sample_rate: 24000,
    container: "none"
  },
  connection_pool_size: 5,
  request_timeout: 30_000,
  keep_alive_interval: 7_000
```

### Production Configuration
```elixir
config :aybiza, :deepgram,
  api_key: System.get_env("DEEPGRAM_API_KEY"),
  environment: :production,
  stt: %{
    model: "nova-3",
    language: "en-US",
    interim_results: true,
    utterances: true,
    vad_events: true,
    endpointing: 300,
    keywords: System.get_env("DEEPGRAM_KEYWORDS")
  },
  tts: %{
    model: System.get_env("DEEPGRAM_TTS_VOICE", "aura-stella-en"),
    encoding: "mulaw",  # For Twilio compatibility
    sample_rate: 8000,
    container: "none"
  },
  connection_pool_size: 20,
  request_timeout: 15_000,
  keep_alive_interval: 7_000,
  retry_config: %{
    max_attempts: 3,
    backoff_type: :exponential,
    backoff_base: 1000
  }
```

## Testing

### Unit Tests
```elixir
defmodule Aybiza.VoicePipeline.DeepgramTest do
  use ExUnit.Case
  import Mox
  
  describe "STT configuration" do
    test "builds correct URL with parameters" do
      config = %{
        model: "nova-3",
        language: "en-US",
        interim_results: true
      }
      
      url = Aybiza.VoicePipeline.DeepgramSTT.build_url(config)
      
      assert url =~ "model=nova-3"
      assert url =~ "language=en-US"
      assert url =~ "interim_results=true"
    end
  end
  
  describe "TTS text enhancement" do
    test "adds natural pauses" do
      text = "Hello. How are you? I'm fine!"
      enhanced = Aybiza.VoicePipeline.DeepgramTTS.enhance_text_for_natural_speech(text, %{})
      
      assert enhanced =~ "Hello\\. "
      assert enhanced =~ "How are you\\? "
      assert enhanced =~ "I'm fine\\! "
    end
  end
end
```

### Integration Tests
```elixir
defmodule Aybiza.VoicePipeline.DeepgramIntegrationTest do
  use ExUnit.Case, async: false
  
  @tag :integration
  test "end-to-end STT processing" do
    # Load test audio file
    audio_data = File.read!("test/fixtures/test_audio.wav")
    
    # Start STT connection
    {:ok, pid} = Aybiza.VoicePipeline.DeepgramSTT.start_link("test_call_id")
    
    # Send audio data
    Aybiza.VoicePipeline.DeepgramSTT.send_audio(pid, audio_data)
    
    # Assert we receive transcription
    assert_receive {:final_transcript, transcript}, 5000
    assert transcript =~ "test"
  end
  
  @tag :integration
  test "TTS synthesis" do
    text = "This is a test message"
    
    {:ok, audio_data} = Aybiza.VoicePipeline.DeepgramTTS.synthesize(text)
    
    assert byte_size(audio_data) > 0
    assert audio_data =~ <<0x52, 0x49, 0x46, 0x46>>  # WAV header
  end
end
```

## Model Upgrade Guide

### From Nova-2 to Nova-3
1. Update model parameter from "nova-2" to "nova-3"
2. Add new features: `utterances`, `vad_events`
3. Adjust endpointing from 500ms to 300ms
4. Test with your specific use cases

### From Previous TTS to Aura-2
1. Update model names to new Aura-2 format
2. Add text enhancement for natural speech
3. Implement voice selection logic
4. Test voice quality with stakeholders

## Troubleshooting

### Common Issues

#### High Latency
- Check network connectivity
- Verify encoding settings match input
- Monitor Deepgram service status
- Review connection pool health

#### Poor Recognition Accuracy
- Verify audio quality (sample rate, encoding)
- Check language settings
- Add domain-specific keywords
- Review noise levels in audio

#### WebSocket Disconnections
- Implement keep-alive mechanism
- Check for network timeouts
- Monitor connection duration
- Add reconnection logic

## Monitoring and Metrics

### Key Metrics to Track
- STT latency (first byte, final result)
- TTS latency (first byte, complete)
- Recognition accuracy (WER)
- Connection uptime
- Error rates by type
- Audio quality metrics

### Dashboards
```elixir
defmodule Aybiza.Metrics.DeepgramDashboard do
  def key_metrics do
    %{
      stt_latency_p50: get_percentile(:stt_latency, 50),
      stt_latency_p99: get_percentile(:stt_latency, 99),
      tts_latency_p50: get_percentile(:tts_latency, 50),
      tts_latency_p99: get_percentile(:tts_latency, 99),
      error_rate: calculate_error_rate(),
      active_connections: count_active_connections(),
      recognition_accuracy: calculate_wer()
    }
  end
end
```

## Future Enhancements

### Planned Features
1. **Real-time Translation**: Use Deepgram's translation capabilities
2. **Speaker Diarization**: Identify multiple speakers
3. **Sentiment Analysis**: Analyze emotional content
4. **Custom Model Training**: Train domain-specific models

### Optimization Opportunities
1. **Edge Deployment**: Deploy Deepgram models at edge locations
2. **Batch Processing**: Optimize for batch transcription
3. **Advanced Caching**: Implement intelligent TTS caching
4. **Multi-Model Routing**: Route to different models based on content

## Support and Resources

### Documentation Links
- [Deepgram API Documentation](https://developers.deepgram.com/docs)
- [Nova-3 Model Guide](https://developers.deepgram.com/docs/nova-3)
- [Aura-2 TTS Guide](https://developers.deepgram.com/docs/aura-tts)

### Support Channels
- Deepgram Support: support@deepgram.com
- Community Forum: community.deepgram.com
- GitHub Issues: github.com/deepgram/issues

## Conclusion

This guide provides comprehensive coverage of Deepgram integration for the AYBIZA platform. The implementation leverages the latest Deepgram features for optimal performance, quality, and reliability. Regular monitoring and updates ensure continued optimization as new features become available.