# AWS Bedrock Claude Implementation for AYBIZA Voice Agents

## Overview
This document provides a comprehensive implementation guide for Claude models on AWS Bedrock within the AYBIZA hybrid Cloudflare+AWS architecture. We leverage the latest Claude models (including Claude 3.7 Sonnet with extended thinking) for optimal performance, ultra-low latency, and enterprise-grade voice interactions.

## Available Claude Models on AWS Bedrock (Updated 2025)

### Model Selection Strategy

#### 1. Claude 3.7 Sonnet (Latest & Most Advanced - February 2025)
- **Model ID**: `anthropic.claude-3-7-sonnet-20250219-v1:0`
- **Purpose**: Complex reasoning with toggleable extended thinking
- **Context Window**: 200K tokens
- **Max Output**: 64,000 tokens
- **Pricing**: $3.00/MTok input, $15.00/MTok output
- **Training Data Cutoff**: November 2024
- **Key Features**:
  - **Extended Thinking**: Transparent reasoning with budget control (1K-64K tokens)
  - **Highest Intelligence**: Best-in-class performance on complex logical tasks
  - **Advanced Tool Use**: Enhanced function calling with improved accuracy
  - **Vision Capabilities**: Advanced image understanding and analysis
  - **Streaming Support**: Optimized for real-time applications
- **Performance**:
  - First Token Latency: ~200-250ms (standard), +500-2000ms (extended thinking)
  - Best for: Complex analysis requiring deep reasoning
- **Voice Agent Use Cases**:
  - Multi-step technical troubleshooting
  - Complex customer issue resolution
  - Strategic consultation requiring deep analysis
  - Advanced conversation context understanding

#### 2. Claude 3.5 Sonnet v2 (Primary Production Model - October 2024)
- **Model ID**: `anthropic.claude-3-5-sonnet-20241022-v2:0`
- **Purpose**: Main workhorse for voice interactions
- **Context Window**: 200K tokens
- **Max Output**: 8,192 tokens
- **Pricing**: $3.00/MTok input, $15.00/MTok output
- **Training Data Cutoff**: April 2024
- **Key Features**:
  - **Enhanced Tool Use**: Improved function calling reliability
  - **Computer Use**: Can interact with computer interfaces (limited in voice)
  - **Streaming Optimized**: Excellent for real-time applications
  - **Vision Capabilities**: Strong image understanding
  - **Balanced Performance**: Optimal mix of intelligence and speed
- **Performance**:
  - First Token Latency: ~150-200ms
  - Best for: Standard customer service with tool integration
- **Voice Agent Use Cases**:
  - Customer service with CRM integration
  - Real-time conversation handling
  - Tool-based workflow automation
  - Complex but time-sensitive interactions

#### 3. Claude 3.5 Haiku (Fastest Intelligence - October 2024)
- **Model ID**: `anthropic.claude-3-5-haiku-20241022-v1:0`
- **Purpose**: Ultra-fast responses with significantly improved intelligence
- **Context Window**: 200K tokens
- **Max Output**: 8,192 tokens
- **Pricing**: $0.80/MTok input, $4.00/MTok output (73% cheaper than Sonnet)
- **Training Data Cutoff**: July 2024
- **Key Features**:
  - **Fastest Intelligence**: Best speed-to-intelligence ratio in Claude family
  - **Ultra-Low Latency**: Optimized for real-time applications
  - **Cost Effective**: Major improvement over Claude 3 Haiku
  - **Text-Only**: No vision support for maximum speed
- **Performance**:
  - First Token Latency: ~80-120ms
  - Best for: Fast responses requiring moderate intelligence
- **Voice Agent Use Cases**:
  - Quick clarifications and acknowledgments
  - Simple routing and qualification
  - High-volume customer interactions
  - Real-time conversation flows requiring speed

#### 4. Claude 3 Haiku (Most Cost-Effective - March 2024)
- **Model ID**: `anthropic.claude-3-haiku-20240307-v1:0`
- **Purpose**: High-volume, simple interactions
- **Context Window**: 200K tokens
- **Max Output**: 4,096 tokens
- **Pricing**: $0.25/MTok input, $1.25/MTok output (92% cheaper than Sonnet)
- **Training Data Cutoff**: August 2023
- **Key Features**:
  - **Lowest Cost**: Most economical option
  - **Basic Intelligence**: Suitable for simple, structured tasks
  - **Vision Support**: Basic image understanding
  - **Fast Response**: Quick processing times
- **Performance**:
  - First Token Latency: ~100-150ms
  - Best for: Simple, high-volume interactions
- **Voice Agent Use Cases**:
  - Greetings and farewells
  - Basic FAQ responses
  - Simple yes/no confirmations
  - Fallback scenarios

## Hybrid Architecture Implementation

### 1. Environment Configuration (Enhanced)

```bash
# AWS Bedrock Configuration for Hybrid Architecture
AWS_ACCESS_KEY_ID=your_aws_access_key_id
AWS_SECRET_ACCESS_KEY=your_aws_secret_access_key
AWS_REGION=us-east-1  # Primary region
AWS_FALLBACK_REGION=us-west-2  # Fallback region

# Multi-Region Model Configuration
AWS_BEDROCK_MODELS="anthropic.claude-3-7-sonnet-20250219-v1:0,anthropic.claude-3-5-sonnet-20241022-v2:0,anthropic.claude-3-5-haiku-20241022-v1:0,anthropic.claude-3-haiku-20240307-v1:0"
AWS_DEFAULT_BEDROCK_MODEL=anthropic.claude-3-5-sonnet-20241022-v2:0

# Performance Optimization Settings
LATENCY_THRESHOLD_MS=200
EXTENDED_THINKING_THRESHOLD=0.9
COST_OPTIMIZATION_ENABLED=true
EDGE_CACHE_ENABLED=true
CLOUDFLARE_REGION_MAPPING=true

# Connection Pool Settings
BEDROCK_POOL_SIZE=10
BEDROCK_CONNECTION_TIMEOUT=30000
BEDROCK_REQUEST_TIMEOUT=15000
```

### 2. Enhanced Bedrock Client with Hybrid Architecture Support

```elixir
defmodule Aybiza.AWS.BedrockClient do
  @moduledoc """
  Enhanced AWS Bedrock client for Claude models with hybrid Cloudflare+AWS support
  """
  
  use GenServer
  require Logger
  
  # Latest Claude model constants
  @claude_3_7_sonnet "anthropic.claude-3-7-sonnet-20250219-v1:0"
  @claude_3_5_sonnet "anthropic.claude-3-5-sonnet-20241022-v2:0"
  @claude_3_5_haiku "anthropic.claude-3-5-haiku-20241022-v1:0"
  @claude_3_haiku "anthropic.claude-3-haiku-20240307-v1:0"
  
  @model_capabilities %{
    @claude_3_7_sonnet => %{
      intelligence: 10,
      speed: 6,
      cost: 1,
      extended_thinking: true,
      tool_use: true,
      vision: true,
      max_tokens: 64000
    },
    @claude_3_5_sonnet => %{
      intelligence: 9,
      speed: 8,
      cost: 1,
      extended_thinking: false,
      tool_use: true,
      vision: true,
      max_tokens: 8192
    },
    @claude_3_5_haiku => %{
      intelligence: 7,
      speed: 10,
      cost: 4,
      extended_thinking: false,
      tool_use: false,
      vision: false,
      max_tokens: 8192
    },
    @claude_3_haiku => %{
      intelligence: 5,
      speed: 9,
      cost: 12,
      extended_thinking: false,
      tool_use: false,
      vision: true,
      max_tokens: 4096
    }
  }
  
  @doc """
  Generate a response using optimal model selection for hybrid architecture
  """
  def generate_response(prompt, context, options \\ %{}) do
    # Determine optimal region based on Cloudflare edge data
    region = determine_optimal_region(context)
    
    # Select model based on requirements and regional availability
    {model_id, region} = select_optimal_model_and_region(prompt, context, options, region)
    
    # Build payload with enhanced features
    payload = build_enhanced_payload(prompt, context, options, model_id)
    
    # Track request start time for latency monitoring
    start_time = System.monotonic_time(:millisecond)
    
    response = if options[:stream] do
      stream_response_with_monitoring(model_id, payload, region, start_time, context)
    else
      invoke_model_with_monitoring(model_id, payload, region, start_time, context)
    end
    
    # Record metrics for optimization
    record_performance_metrics(model_id, region, response, start_time, context)
    
    response
  end
  
  @doc """
  Enhanced streaming with sentence-level processing for voice optimization
  """
  def stream_response_with_monitoring(model_id, payload, region, start_time, context) do
    # Enable streaming in payload
    payload = Map.put(payload, "stream", true)
    
    # Create stream with region-specific client
    stream = create_regional_stream(model_id, payload, region)
    
    # Process stream with voice optimization
    stream
    |> Stream.transform(%{buffer: "", sentence_count: 0, first_token_time: nil}, 
       &process_streaming_chunk(&1, &2, start_time, context))
    |> Stream.each(&handle_voice_optimized_chunk(&1, context))
  end
  
  @doc """
  Intelligent model selection with regional optimization
  """
  def select_optimal_model_and_region(prompt, context, options, preferred_region) do
    # Assess complexity and requirements
    complexity = assess_query_complexity(prompt, context)
    latency_requirement = Map.get(options, :max_latency, 500)
    cost_sensitivity = Map.get(options, :cost_optimize, false)
    
    # Select model based on requirements
    model_id = cond do
      # Extended thinking for very complex queries
      complexity > 0.9 && latency_requirement > 1000 ->
        @claude_3_7_sonnet
      
      # Ultra-fast responses for simple queries
      complexity < 0.3 && latency_requirement < 150 ->
        if cost_sensitivity, do: @claude_3_haiku, else: @claude_3_5_haiku
      
      # Fast responses with better intelligence
      complexity < 0.6 && latency_requirement < 200 ->
        @claude_3_5_haiku
      
      # Tool use requirements
      requires_tool_use?(context) ->
        @claude_3_5_sonnet
      
      # Complex queries needing intelligence
      complexity > 0.7 ->
        @claude_3_5_sonnet
      
      # Default balanced choice
      true ->
        @claude_3_5_sonnet
    end
    
    # Verify model availability in region
    available_region = ensure_model_availability(model_id, preferred_region)
    
    {model_id, available_region}
  end
  
  @doc """
  Build enhanced payload with latest features
  """
  def build_enhanced_payload(prompt, context, options, model_id) do
    base_payload = %{
      "anthropic_version" => "bedrock-2023-05-31",
      "max_tokens" => determine_max_tokens(model_id, options),
      "messages" => build_conversation_messages(prompt, context),
      "temperature" => Map.get(options, :temperature, 0.3),  # Lower for voice consistency
      "top_p" => Map.get(options, :top_p, 0.9),
      "top_k" => Map.get(options, :top_k, 250)
    }
    
    # Add system prompt with voice optimization
    base_payload = if context[:system_prompt] do
      Map.put(base_payload, "system", enhance_system_prompt_for_voice(context[:system_prompt]))
    else
      base_payload
    end
    
    # Add tools if available and supported by model
    base_payload = if context[:tools] && supports_tool_use?(model_id) do
      Map.merge(base_payload, %{
        "tools" => context[:tools],
        "tool_choice" => Map.get(options, :tool_choice, "auto")
      })
    else
      base_payload
    end
    
    # Add extended thinking for Claude 3.7 Sonnet
    if model_id == @claude_3_7_sonnet && options[:extended_thinking] do
      thinking_budget = Map.get(options, :thinking_budget, 5000)
      Map.merge(base_payload, %{
        "thinking" => true,
        "budget_tokens" => min(thinking_budget, 64000)  # Cap at model limit
      })
    else
      base_payload
    end
  end
  
  @doc """
  Process streaming chunks with voice optimization
  """
  def process_streaming_chunk(chunk, acc, start_time, context) do
    case parse_bedrock_chunk(chunk) do
      %{"type" => "content_block_delta", "delta" => %{"text" => text}} ->
        # Record first token time
        acc = if acc.first_token_time == nil do
          first_token_latency = System.monotonic_time(:millisecond) - start_time
          record_first_token_latency(first_token_latency, context)
          %{acc | first_token_time: System.monotonic_time(:millisecond)}
        else
          acc
        end
        
        # Accumulate text and detect sentence boundaries
        new_buffer = acc.buffer <> text
        
        case extract_complete_sentences(new_buffer) do
          {[], remaining_buffer} ->
            # No complete sentences yet
            {[], %{acc | buffer: remaining_buffer}}
          
          {sentences, remaining_buffer} ->
            # Send complete sentences immediately for TTS
            sentence_events = Enum.map(sentences, fn sentence ->
              %{
                type: "sentence_complete",
                text: sentence,
                sequence: acc.sentence_count + 1,
                context: context
              }
            end)
            
            {sentence_events, %{acc | 
              buffer: remaining_buffer, 
              sentence_count: acc.sentence_count + length(sentences)
            }}
        end
      
      %{"type" => "content_block_stop"} ->
        # Send any remaining text
        if acc.buffer != "" do
          final_event = %{
            type: "content_final",
            text: acc.buffer,
            sequence: acc.sentence_count + 1,
            context: context
          }
          {[final_event], %{acc | buffer: ""}}
        else
          {[], acc}
        end
      
      %{"type" => "message_stop"} ->
        # Stream complete
        {[%{type: "stream_complete", context: context}], acc}
      
      %{"type" => "error", "error" => error} ->
        # Handle errors gracefully
        Logger.error("Bedrock streaming error: #{inspect(error)}")
        {[%{type: "error", error: error, context: context}], acc}
      
      _ ->
        # Ignore other chunk types
        {[], acc}
    end
  end
  
  @doc """
  Handle voice-optimized chunks for immediate TTS processing
  """
  def handle_voice_optimized_chunk(event, context) do
    case event.type do
      "sentence_complete" ->
        # Send to TTS immediately for ultra-low latency
        Task.start(fn ->
          Aybiza.VoicePipeline.DeepgramTTS.synthesize_streaming(
            context.call_id,
            event.text,
            %{
              priority: :high,
              sequence: event.sequence,
              model: get_optimal_tts_voice(context)
            }
          )
        end)
      
      "content_final" ->
        # Send remaining content
        Task.start(fn ->
          Aybiza.VoicePipeline.DeepgramTTS.synthesize_streaming(
            context.call_id,
            event.text,
            %{
              priority: :normal,
              sequence: event.sequence,
              final: true
            }
          )
        end)
      
      "stream_complete" ->
        # Notify that LLM processing is complete
        GenServer.cast(context.call_id, {:llm_complete, System.monotonic_time(:millisecond)})
      
      "error" ->
        # Handle errors with fallback response
        handle_streaming_error(event.error, context)
    end
  end
  
  @doc """
  Enhanced complexity assessment with pattern recognition
  """
  def assess_query_complexity(prompt, context) do
    # Word count factor
    word_count = String.split(prompt) |> length()
    word_factor = min(word_count / 50, 1.0)
    
    # Pattern recognition
    complex_patterns = [
      ~r/(analyze|compare|evaluate|reasoning|logic|step.by.step)/i,
      ~r/(troubleshoot|debug|diagnose|investigate)/i,
      ~r/(calculate|compute|determine|solve)/i,
      ~r/(explain.why|walk.me.through|break.down)/i
    ]
    
    pattern_score = complex_patterns
    |> Enum.count(&Regex.match?(&1, prompt))
    |> Kernel./(length(complex_patterns))
    
    # Context factors
    context_factor = cond do
      length(context[:conversation_history] || []) > 5 -> 0.3
      context[:requires_tools] -> 0.4
      context[:multi_turn] -> 0.2
      true -> 0.0
    end
    
    # Calculate final complexity (0.0 to 1.0)
    complexity = (word_factor * 0.3) + (pattern_score * 0.5) + context_factor
    min(complexity, 1.0)
  end
  
  @doc """
  Determine optimal AWS region based on Cloudflare edge data
  """
  def determine_optimal_region(context) do
    # Use Cloudflare country data if available
    country = Map.get(context, :cloudflare_country, "US")
    
    region_map = %{
      "US" => "us-east-1",
      "CA" => "us-east-1",
      "GB" => "eu-west-1",
      "DE" => "eu-central-1",
      "FR" => "eu-west-1",
      "IT" => "eu-central-1",
      "ES" => "eu-west-1",
      "NL" => "eu-west-1",
      "SG" => "ap-southeast-1",
      "JP" => "ap-northeast-1",
      "AU" => "ap-southeast-1",
      "IN" => "ap-south-1"
    }
    
    Map.get(region_map, country, "us-east-1")
  end
  
  @doc """
  Enhanced error handling with intelligent fallbacks
  """
  def handle_bedrock_error(error, context) do
    case error do
      {:error, %{"type" => "ThrottlingException"}} ->
        handle_throttling_with_model_fallback(context)
      
      {:error, %{"type" => "ModelNotReadyException"}} ->
        use_alternative_model_in_region(context)
      
      {:error, %{"type" => "ServiceUnavailableException"}} ->
        switch_to_fallback_region(context)
      
      {:error, %{"type" => "ValidationException"}} ->
        fix_payload_and_retry(context)
      
      {:error, :timeout} ->
        switch_to_faster_model_and_retry(context)
      
      _ ->
        use_cached_or_generic_response(context)
    end
  end
  
  defp handle_throttling_with_model_fallback(context) do
    # Switch to less resource-intensive model
    fallback_model = case context.current_model do
      @claude_3_7_sonnet -> @claude_3_5_sonnet
      @claude_3_5_sonnet -> @claude_3_5_haiku
      @claude_3_5_haiku -> @claude_3_haiku
      _ -> @claude_3_haiku
    end
    
    # Implement exponential backoff
    delay = calculate_backoff_delay(context.retry_count || 0)
    Process.sleep(delay)
    
    # Retry with fallback model
    new_context = %{context | current_model: fallback_model, retry_count: (context.retry_count || 0) + 1}
    generate_response(context.prompt, new_context, context.options)
  end
  
  defp extract_complete_sentences(text) do
    # Enhanced sentence boundary detection for voice
    sentences = Regex.split(~r/(?<=[.!?])\s+(?=[A-Z])/, text, include_captures: false, trim: true)
    
    case sentences do
      [] -> 
        {[], text}
      [incomplete] when not String.match?(incomplete, ~r/[.!?]$/) -> 
        {[], incomplete}
      sentences ->
        # Find sentences that end with punctuation
        {complete, remaining} = Enum.split_while(sentences, &String.match?(&1, ~r/[.!?]$/))
        remaining_text = Enum.join(remaining, " ")
        {complete, remaining_text}
    end
  end
  
  defp enhance_system_prompt_for_voice(system_prompt) do
    voice_enhancement = """
    
    VOICE INTERACTION GUIDELINES:
    - Keep responses under 50 words for quick delivery
    - Use natural, conversational language suitable for speech
    - Avoid complex punctuation that sounds awkward when spoken
    - End responses with clear closure or a specific question
    - Use contractions for natural speech flow (don't, can't, I'll)
    """
    
    system_prompt <> voice_enhancement
  end
  
  defp supports_tool_use?(model_id) do
    model_id in [@claude_3_7_sonnet, @claude_3_5_sonnet]
  end
  
  defp determine_max_tokens(model_id, options) do
    default_max = case model_id do
      @claude_3_7_sonnet -> 2048  # Conservative for voice, can go up to 64K
      @claude_3_5_sonnet -> 1024  # Conservative for voice
      @claude_3_5_haiku -> 512   # Fast responses
      @claude_3_haiku -> 256     # Simple responses
    end
    
    Map.get(options, :max_tokens, default_max)
  end
  
  defp record_performance_metrics(model_id, region, response, start_time, context) do
    total_latency = System.monotonic_time(:millisecond) - start_time
    
    # Extract metrics from response
    metrics = %{
      model_id: model_id,
      region: region,
      total_latency: total_latency,
      input_tokens: get_input_tokens(response),
      output_tokens: get_output_tokens(response),
      thinking_tokens: get_thinking_tokens(response),
      cost_usd: calculate_cost(model_id, response),
      call_sid: Map.get(context, :call_sid),
      tenant_id: Map.get(context, :tenant_id),
      cloudflare_region: Map.get(context, :cloudflare_region)
    }
    
    # Record to both CloudWatch and TimescaleDB
    Task.start(fn ->
      Aybiza.Metrics.Claude.record_model_performance(model_id, metrics)
    end)
  end
  
  defp calculate_cost(model_id, response) do
    input_tokens = get_input_tokens(response)
    output_tokens = get_output_tokens(response)
    thinking_tokens = get_thinking_tokens(response)
    
    costs = case model_id do
      @claude_3_7_sonnet -> %{input: 3.00, output: 15.00}
      @claude_3_5_sonnet -> %{input: 3.00, output: 15.00}
      @claude_3_5_haiku -> %{input: 0.80, output: 4.00}
      @claude_3_haiku -> %{input: 0.25, output: 1.25}
    end
    
    input_cost = (input_tokens / 1000) * costs.input
    output_cost = ((output_tokens + thinking_tokens) / 1000) * costs.output
    
    input_cost + output_cost
  end
end
```

### 3. Enhanced Voice Agent Integration with Extended Thinking

```elixir
defmodule Aybiza.VoiceAgent.ClaudeConversation do
  @moduledoc """
  Enhanced Claude-powered conversation handler with extended thinking support
  """
  
  alias Aybiza.AWS.BedrockClient
  
  @voice_system_prompts %{
    customer_service: """
    You are a professional customer service voice assistant. 
    
    VOICE GUIDELINES:
    - Keep responses under 40 words for quick delivery
    - Use warm, professional tone suitable for phone conversations
    - Acknowledge customer concerns before providing solutions
    - Ask one clarifying question at a time
    - Confirm understanding before taking action
    
    RESPONSE FORMAT:
    - Start with acknowledgment ("I understand..." or "I see...")
    - Provide clear, actionable next steps
    - End with specific question or confirmation
    - Use natural speech patterns with contractions
    """,
    
    technical_support: """
    You are a knowledgeable technical support specialist.
    
    TECHNICAL COMMUNICATION:
    - Explain complex concepts in simple terms
    - Break down solutions into clear steps
    - Confirm understanding after each major point
    - Use analogies to clarify technical concepts
    
    TROUBLESHOOTING APPROACH:
    - Start with most common solutions
    - Ask diagnostic questions systematically
    - Provide step-by-step guidance
    - Offer alternatives if initial solution fails
    """,
    
    sales_assistant: """
    You are a consultative sales assistant focused on understanding needs.
    
    SALES APPROACH:
    - Listen actively to identify customer needs
    - Ask qualifying questions to understand requirements
    - Focus on benefits and value, not just features
    - Be helpful and consultative, not pushy
    
    CONVERSATION STYLE:
    - Build rapport through active listening
    - Match customer's pace and communication style
    - Use enthusiasm appropriately
    - Always provide clear next steps
    """
  }
  
  def handle_user_input(call_id, transcript, context) do
    # Get agent configuration
    agent_config = get_agent_config(context.agent_id)
    
    # Assess if extended thinking would be beneficial
    extended_thinking_needed = should_use_extended_thinking?(transcript, context)
    
    # Build enhanced conversation context
    conversation_context = %{
      conversation_history: get_conversation_history(call_id),
      system_prompt: get_system_prompt(agent_config),
      tools: get_available_tools(agent_config),
      complexity: BedrockClient.assess_query_complexity(transcript, context),
      cloudflare_country: Map.get(context, :country),
      cloudflare_region: Map.get(context, :edge_region),
      call_sid: call_id,
      tenant_id: context.tenant_id,
      requires_tools: has_tool_intent?(transcript)
    }
    
    # Enhanced response options
    options = %{
      stream: true,  # Always stream for voice
      max_tokens: determine_response_length(transcript, context),
      temperature: 0.3,  # Lower for consistent voice responses
      max_latency: get_latency_requirement(context),
      extended_thinking: extended_thinking_needed,
      thinking_budget: if(extended_thinking_needed, do: 3000, else: 0),
      cost_optimize: Map.get(context, :cost_optimize, false)
    }
    
    # Generate response with enhanced monitoring
    start_time = System.monotonic_time(:millisecond)
    
    response_stream = BedrockClient.generate_response(
      transcript,
      conversation_context,
      options
    )
    
    # Process with voice optimization
    process_enhanced_streaming_response(call_id, response_stream, start_time, context)
  end
  
  defp should_use_extended_thinking?(transcript, context) do
    # Check for patterns that benefit from extended thinking
    thinking_patterns = [
      ~r/(analyze|analyze\s+the|break\s+down|step\s+by\s+step)/i,
      ~r/(complex|complicated|detailed\s+explanation)/i,
      ~r/(troubleshoot|debug|diagnose|investigate)/i,
      ~r/(compare|evaluate|assess|determine\s+the\s+best)/i,
      ~r/(reasoning|logic|why\s+does|how\s+does\s+this\s+work)/i
    ]
    
    pattern_match = Enum.any?(thinking_patterns, &Regex.match?(&1, transcript))
    
    # Additional context factors
    has_complexity = String.length(transcript) > 100
    has_context = length(Map.get(context, :conversation_history, [])) > 3
    is_technical = Map.get(context, :agent_type) == :technical_support
    
    # Use extended thinking if multiple factors align
    (pattern_match && (has_complexity || is_technical)) || 
    (pattern_match && has_context && String.length(transcript) > 50)
  end
  
  defp process_enhanced_streaming_response(call_id, stream, start_time, context) do
    # Enhanced streaming with parallel TTS processing
    Task.async_stream(stream, fn chunk ->
      case chunk do
        %{type: "sentence_complete", text: text, sequence: seq} ->
          # Immediate TTS processing for ultra-low latency
          tts_start = System.monotonic_time(:millisecond)
          
          Aybiza.VoicePipeline.DeepgramTTS.synthesize_streaming(
            call_id,
            optimize_text_for_speech(text),
            %{
              voice: get_optimal_voice_for_context(context),
              priority: :high,
              sequence: seq,
              encoding: "mulaw",  # Direct Twilio compatibility
              sample_rate: 8000,
              container: "none"
            }
          )
          
          # Record TTS latency
          tts_latency = System.monotonic_time(:millisecond) - tts_start
          record_tts_latency(call_id, tts_latency, seq)
        
        %{type: "content_final", text: text} ->
          # Process final content
          if String.trim(text) != "" do
            Aybiza.VoicePipeline.DeepgramTTS.synthesize_streaming(
              call_id,
              optimize_text_for_speech(text),
              %{final: true}
            )
          end
        
        %{type: "stream_complete"} ->
          # Record total conversation latency
          total_latency = System.monotonic_time(:millisecond) - start_time
          record_conversation_latency(call_id, total_latency, context)
        
        %{type: "error", error: error} ->
          # Handle errors with graceful fallback
          handle_conversation_error(call_id, error, context)
      end
    end, timeout: 30_000, max_concurrency: 3)
    |> Stream.run()
  end
  
  defp optimize_text_for_speech(text) do
    text
    |> String.replace(~r/\s+/, " ")  # Normalize whitespace
    |> String.replace(~r/([.!?])\s*$/, "\\1")  # Ensure proper punctuation
    |> String.replace(~r/\b(URL|API|ID|FAQ)\b/, fn match ->
      # Spell out common acronyms for better TTS
      case match do
        "URL" -> "U-R-L"
        "API" -> "A-P-I"
        "ID" -> "I-D"
        "FAQ" -> "F-A-Q"
        _ -> match
      end
    end)
  end
  
  defp get_optimal_voice_for_context(context) do
    # Select voice based on agent type and customer preference
    case Map.get(context, :agent_type, :customer_service) do
      :technical_support -> "aura-perseus-en"  # Professional male
      :sales_assistant -> "aura-luna-en"       # Expressive female
      :customer_service -> "aura-asteria-en"   # Calm, informative female
      _ -> "aura-stella-en"                     # Clear, professional female
    end
  end
  
  defp determine_response_length(transcript, context) do
    # Adaptive response length based on query complexity
    base_length = 150  # Words, not tokens
    
    cond do
      String.contains?(transcript, ["explain", "detail", "walk me through"]) -> base_length * 2
      String.contains?(transcript, ["quick", "fast", "briefly"]) -> base_length * 0.5
      Map.get(context, :agent_type) == :technical_support -> base_length * 1.5
      true -> base_length
    end
    |> round()
    |> max(50)   # Minimum response length
    |> min(300)  # Maximum for voice interactions
  end
  
  defp has_tool_intent?(transcript) do
    tool_patterns = [
      ~r/(look\s+up|search|find|check|verify)/i,
      ~r/(create|update|modify|change|set)/i,
      ~r/(send|email|notify|alert)/i,
      ~r/(schedule|book|reserve|cancel)/i,
      ~r/(transfer|escalate|connect)/i
    ]
    
    Enum.any?(tool_patterns, &Regex.match?(&1, transcript))
  end
  
  defp get_system_prompt(agent_config) do
    agent_type = Map.get(agent_config, :type, :customer_service)
    base_prompt = Map.get(@voice_system_prompts, agent_type, @voice_system_prompts.customer_service)
    
    # Add custom instructions if available
    custom_instructions = Map.get(agent_config, :custom_instructions, "")
    
    if custom_instructions != "" do
      base_prompt <> "\n\nCUSTOM INSTRUCTIONS:\n" <> custom_instructions
    else
      base_prompt
    end
  end
end
```

### 4. Performance Monitoring and Optimization

```elixir
defmodule Aybiza.VoiceAgent.PerformanceMonitor do
  @moduledoc """
  Advanced performance monitoring for hybrid Claude implementation
  """
  
  def track_conversation_performance(call_id, metrics) do
    # Enhanced metrics for hybrid architecture
    enhanced_metrics = Map.merge(metrics, %{
      edge_processing_time: calculate_edge_processing_time(metrics),
      model_selection_accuracy: assess_model_selection(metrics),
      cost_optimization_score: calculate_cost_efficiency(metrics),
      latency_breakdown: analyze_latency_breakdown(metrics)
    })
    
    # Multi-destination logging
    log_to_cloudwatch(enhanced_metrics)
    log_to_timescaledb(enhanced_metrics)
    update_real_time_dashboard(enhanced_metrics)
  end
  
  def analyze_model_performance_trends(period \\ "24h") do
    %{
      model_usage_distribution: get_model_usage_stats(period),
      latency_trends: analyze_latency_trends(period),
      cost_trends: analyze_cost_trends(period),
      quality_metrics: assess_conversation_quality(period),
      optimization_recommendations: generate_optimization_recommendations(period)
    }
  end
  
  defp calculate_edge_processing_time(metrics) do
    # Time spent at Cloudflare edge before reaching AWS
    edge_start = Map.get(metrics, :cloudflare_timestamp)
    aws_start = Map.get(metrics, :aws_timestamp)
    
    if edge_start && aws_start do
      aws_start - edge_start
    else
      nil
    end
  end
  
  defp assess_model_selection(metrics) do
    # Evaluate if the selected model was optimal
    actual_complexity = Map.get(metrics, :actual_complexity, 0.5)
    selected_model = Map.get(metrics, :model_used)
    
    optimal_model = determine_optimal_model_for_complexity(actual_complexity)
    
    %{
      selected: selected_model,
      optimal: optimal_model,
      was_optimal: selected_model == optimal_model,
      efficiency_score: calculate_efficiency_score(selected_model, optimal_model)
    }
  end
  
  def generate_optimization_recommendations(period) do
    performance_data = get_performance_data(period)
    
    recommendations = []
    
    # Model selection optimization
    recommendations = if should_adjust_model_selection?(performance_data) do
      [suggest_model_selection_changes(performance_data) | recommendations]
    else
      recommendations
    end
    
    # Latency optimization
    recommendations = if has_latency_issues?(performance_data) do
      [suggest_latency_optimizations(performance_data) | recommendations]
    else
      recommendations
    end
    
    # Cost optimization
    recommendations = if has_cost_optimization_potential?(performance_data) do
      [suggest_cost_optimizations(performance_data) | recommendations]
    else
      recommendations
    end
    
    recommendations
  end
end
```

### 5. Advanced Error Handling and Circuit Breakers

```elixir
defmodule Aybiza.VoiceAgent.CircuitBreaker do
  @moduledoc """
  Circuit breaker implementation for Claude model resilience
  """
  
  use GenServer
  
  @failure_threshold 5
  @timeout_threshold 3
  @recovery_time 30_000  # 30 seconds
  
  def start_link(model_id) do
    GenServer.start_link(__MODULE__, model_id, name: via_tuple(model_id))
  end
  
  def call_with_circuit_breaker(model_id, fun) do
    case GenServer.call(via_tuple(model_id), :get_state) do
      :closed ->
        # Circuit is closed, proceed with call
        execute_with_monitoring(model_id, fun)
      
      :open ->
        # Circuit is open, use fallback
        {:error, :circuit_breaker_open}
      
      :half_open ->
        # Circuit is half-open, try one request
        case execute_with_monitoring(model_id, fun) do
          {:ok, result} ->
            GenServer.cast(via_tuple(model_id), :success)
            {:ok, result}
          error ->
            GenServer.cast(via_tuple(model_id), :failure)
            error
        end
    end
  end
  
  defp execute_with_monitoring(model_id, fun) do
    start_time = System.monotonic_time(:millisecond)
    
    try do
      result = fun.()
      duration = System.monotonic_time(:millisecond) - start_time
      
      # Record success
      GenServer.cast(via_tuple(model_id), {:success, duration})
      {:ok, result}
    rescue
      error ->
        duration = System.monotonic_time(:millisecond) - start_time
        GenServer.cast(via_tuple(model_id), {:failure, error, duration})
        {:error, error}
    end
  end
  
  def init(model_id) do
    {:ok, %{
      model_id: model_id,
      state: :closed,
      failure_count: 0,
      last_failure_time: nil,
      success_count: 0
    }}
  end
  
  def handle_call(:get_state, _from, state) do
    # Check if we should transition from open to half-open
    new_state = maybe_transition_to_half_open(state)
    {:reply, new_state.state, new_state}
  end
  
  def handle_cast({:failure, _error, _duration}, state) do
    new_failure_count = state.failure_count + 1
    
    new_state = %{state | 
      failure_count: new_failure_count,
      last_failure_time: System.monotonic_time(:millisecond)
    }
    
    # Open circuit if threshold exceeded
    if new_failure_count >= @failure_threshold do
      Logger.warning("Circuit breaker opened for model #{state.model_id}")
      {:noreply, %{new_state | state: :open}}
    else
      {:noreply, new_state}
    end
  end
  
  def handle_cast(:success, state) do
    case state.state do
      :half_open ->
        # Success in half-open state, close the circuit
        Logger.info("Circuit breaker closed for model #{state.model_id}")
        {:noreply, %{state | state: :closed, failure_count: 0}}
      
      _ ->
        # Normal success
        {:noreply, %{state | success_count: state.success_count + 1}}
    end
  end
  
  defp maybe_transition_to_half_open(state) do
    if state.state == :open && 
       state.last_failure_time && 
       (System.monotonic_time(:millisecond) - state.last_failure_time) > @recovery_time do
      %{state | state: :half_open}
    else
      state
    end
  end
  
  defp via_tuple(model_id) do
    {:via, Registry, {Aybiza.VoiceAgent.CircuitBreakerRegistry, model_id}}
  end
end
```

## Implementation Checklist (Updated)

### Phase 1: Foundation (Weeks 1-2)
- [ ] Set up AWS Bedrock access with latest model IDs
- [ ] Configure multi-region deployment
- [ ] Implement enhanced Bedrock client with hybrid support
- [ ] Test connectivity to all Claude models
- [ ] Implement circuit breakers and error handling
- [ ] Set up performance monitoring

### Phase 2: Voice Integration (Weeks 3-4)
- [ ] Implement streaming with sentence-level processing
- [ ] Create voice-optimized system prompts
- [ ] Integrate with enhanced STT/TTS pipeline
- [ ] Implement intelligent model selection
- [ ] Test end-to-end latency (target <200ms)
- [ ] Implement extended thinking for complex queries

### Phase 3: Optimization (Weeks 5-6)
- [ ] Implement dynamic model switching
- [ ] Add response caching with semantic similarity
- [ ] Optimize streaming chunk processing
- [ ] Implement parallel TTS processing
- [ ] Add comprehensive performance monitoring
- [ ] Optimize for cost efficiency

### Phase 4: Advanced Features (Weeks 7-8)
- [ ] Implement tool use with function calling
- [ ] Add conversation memory and context management
- [ ] Implement sentiment analysis and quality scoring
- [ ] Add proactive assistance capabilities
- [ ] Implement A/B testing for model selection
- [ ] Add advanced analytics and insights

### Phase 5: Production Readiness (Weeks 9-10)
- [ ] Complete error handling and fallback strategies
- [ ] Add comprehensive logging and audit trails
- [ ] Implement rate limiting and quota management
- [ ] Add cost tracking and billing integration
- [ ] Deploy monitoring dashboard
- [ ] Conduct load testing and performance validation

This comprehensive implementation guide provides everything needed to build a production-ready voice agent system using the latest Claude models on AWS Bedrock within the AYBIZA hybrid Cloudflare+AWS architecture, ensuring optimal performance, cost efficiency, and reliability.