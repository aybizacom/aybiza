# Claude Models Reference for AWS Bedrock

## Overview
This document provides a comprehensive reference for all Claude models available on AWS Bedrock, including their capabilities, pricing, performance characteristics, and best practices for AI voice agents in the AYBIZA hybrid Cloudflare+AWS architecture.

## Available Claude Models on AWS Bedrock (Updated 2025)

### 1. Claude 3.7 Sonnet (Latest - February 2025)
- **Model ID**: `anthropic.claude-3-7-sonnet-20250219-v1:0`
- **Release Date**: February 19, 2025
- **Capabilities**: Highest intelligence with toggleable extended thinking
- **Context Window**: 200K tokens
- **Max Output**: 64,000 tokens
- **Pricing**: $3.00/MTok input, $15.00/MTok output
- **Training Data Cutoff**: November 2024
- **Key Features**:
  - **Extended Thinking**: Toggleable reasoning transparency with budget control
  - **Superior Reasoning**: Best-in-class performance on complex logical tasks
  - **Advanced Tool Use**: Enhanced function calling with improved accuracy
  - **Vision Capabilities**: Advanced image understanding and analysis
  - **Streaming Support**: Optimized for real-time applications
  - **Latest Knowledge**: Most up-to-date training data in Claude family
- **Performance**:
  - First Token Latency: ~200-250ms (without extended thinking)
  - Extended Thinking Latency: +500-2000ms (depending on complexity)
  - Throughput: High (streaming optimized)
- **Use Cases**: 
  - Complex multi-step reasoning and analysis
  - Advanced conversation handling with nuanced understanding
  - Scientific and technical problem solving
  - Detailed content generation requiring deep thinking
  - Enterprise applications requiring highest accuracy

### 2. Claude 3.5 Sonnet v2 (Updated October 2024)
- **Model ID**: `anthropic.claude-3-5-sonnet-20241022-v2:0`
- **Release Date**: October 22, 2024
- **Context Window**: 200K tokens
- **Max Output**: 8,192 tokens
- **Pricing**: $3.00/MTok input, $15.00/MTok output
- **Training Data Cutoff**: April 2024
- **Key Features**:
  - **Balanced Performance**: Optimal mix of intelligence and speed
  - **Enhanced Tool Use**: Improved function calling reliability
  - **Streaming Optimized**: Excellent for real-time applications
  - **Vision Capabilities**: Strong image understanding
  - **Computer Use**: Can interact with computer interfaces (limited in voice contexts)
- **Performance**:
  - First Token Latency: ~150-200ms
  - Throughput: High
  - Quality: Excellent for most use cases
- **Use Cases**: 
  - Primary model for voice agents requiring intelligence
  - Complex conversations with tool integration
  - Real-time interactions with moderate complexity
  - Customer service with advanced reasoning needs

### 3. Claude 3.5 Haiku (New - October 2024)
- **Model ID**: `anthropic.claude-3-5-haiku-20241022-v1:0`
- **Release Date**: October 22, 2024
- **Context Window**: 200K tokens
- **Max Output**: 8,192 tokens
- **Pricing**: $0.80/MTok input, $4.00/MTok output
- **Training Data Cutoff**: July 2024
- **Key Features**:
  - **Fastest Intelligence**: Best speed-to-intelligence ratio
  - **Ultra-Low Latency**: Optimized for real-time applications
  - **Cost Effective**: 5x cheaper than Sonnet models
  - **Streaming Optimized**: Excellent streaming performance
  - **No Vision Support**: Text-only for maximum speed
- **Performance**:
  - First Token Latency: ~80-120ms
  - Throughput: Very High
  - Quality: Significantly improved over Claude 3 Haiku
- **Use Cases**: 
  - Ultra-low latency voice applications (<150ms total)
  - High-volume interactions where cost matters
  - Simple to moderate complexity queries
  - Real-time conversation flows requiring speed
  - Greeting, acknowledgment, and routing responses

### 4. Claude 3 Haiku (Original)
- **Model ID**: `anthropic.claude-3-haiku-20240307-v1:0`
- **Release Date**: March 7, 2024
- **Context Window**: 200K tokens
- **Max Output**: 4,096 tokens
- **Pricing**: $0.25/MTok input, $1.25/MTok output
- **Training Data Cutoff**: August 2023
- **Key Features**:
  - **Most Cost-Effective**: Lowest pricing in Claude family
  - **Basic Intelligence**: Suitable for simple tasks
  - **Fast Response**: Quick processing times
  - **Vision Support**: Basic image understanding
- **Performance**:
  - First Token Latency: ~100-150ms
  - Throughput: High
  - Quality: Basic but reliable
- **Use Cases**: 
  - Simple queries and responses
  - Basic translations and summaries
  - Cost-sensitive applications with simple requirements
  - Fallback model for system resilience

### 5. Claude 3 Opus (Legacy)
- **Model ID**: `anthropic.claude-3-opus-20240229-v1:0`
- **Release Date**: February 29, 2024
- **Context Window**: 200K tokens
- **Max Output**: 4,096 tokens
- **Pricing**: $15.00/MTok input, $75.00/MTok output
- **Training Data Cutoff**: August 2023
- **Status**: ⚠️ Consider migrating to Claude 3.7 Sonnet for better performance
- **Key Features**:
  - **Deep Understanding**: Excellent for nuanced analysis
  - **High Cost**: Most expensive model
  - **Vision Capabilities**: Strong image understanding
  - **Older Training**: Less recent knowledge
- **Use Cases**: 
  - Legacy applications requiring migration
  - Specialized tasks where cost is not a concern
  - **Recommendation**: Migrate to Claude 3.7 Sonnet for better value

## Extended Thinking Feature (Claude 3.7 Sonnet)

### What is Extended Thinking?
Extended thinking allows Claude 3.7 Sonnet to show its reasoning process explicitly, similar to human "thinking out loud." This feature provides transparency into the model's decision-making process.

### Implementation
```elixir
defmodule Aybiza.AWS.BedrockExtendedThinking do
  def invoke_with_thinking(prompt, thinking_budget \\ 5000) do
    payload = %{
      "anthropic_version" => "bedrock-2023-05-31",
      "model" => "anthropic.claude-3-7-sonnet-20250219-v1:0",
      "messages" => [
        %{"role" => "user", "content" => prompt}
      ],
      "thinking" => true,
      "budget_tokens" => thinking_budget,
      "max_tokens" => 8192,
      "temperature" => 0.3  # Fixed for thinking mode
    }
    
    case AWS.BedrockRuntime.converse(client(), payload) do
      {:ok, response} ->
        # Response includes both thinking and final answer
        %{
          "thinking" => response["thinking"],  # The reasoning process
          "content" => response["content"]     # The final answer
        }
      error -> error
    end
  end
end
```

### Extended Thinking Configuration
- **Minimum Budget**: 1,024 tokens
- **Maximum Budget**: 64,000 tokens
- **Pricing**: All tokens (thinking + output) charged at output rate ($15/MTok)
- **Latency Impact**: Adds 500ms-2000ms depending on complexity
- **Temperature**: Fixed at model default (cannot be modified)
- **Streaming**: Compatible with streaming responses

### When to Use Extended Thinking
✅ **Recommended for**:
- Complex analytical tasks requiring step-by-step reasoning
- Multi-variable problem solving
- Debugging complex customer issues
- Strategic decision making
- Technical troubleshooting

❌ **Not recommended for**:
- Simple queries or greetings
- Time-sensitive voice interactions (<500ms)
- High-volume, low-complexity requests
- Cost-sensitive applications

## Regional Availability and Performance

### Model Availability by Region (Updated 2025)

| Model | US East 1 | US West 2 | EU West 1 | EU Central 1 | AP Southeast 1 | AP Northeast 1 |
|-------|-----------|-----------|-----------|--------------|----------------|----------------|
| Claude 3.7 Sonnet | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Claude 3.5 Sonnet v2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Claude 3.5 Haiku | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ |
| Claude 3 Haiku | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Claude 3 Opus | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |

### Performance by Region (Latency from US)
- **US East 1**: 80-120ms (Optimal for US East Coast)
- **US West 2**: 120-150ms (Optimal for US West Coast)
- **EU West 1**: 180-220ms (Optimal for Europe)
- **EU Central 1**: 200-240ms (Alternative for Europe)
- **AP Southeast 1**: 300-350ms (Optimal for Asia-Pacific)

### Hybrid Architecture Optimization
```elixir
defmodule Aybiza.Hybrid.ModelRouter do
  @region_preferences %{
    # Cloudflare edge to AWS region mapping
    "US" => "us-east-1",
    "CA" => "us-east-1", 
    "GB" => "eu-west-1",
    "DE" => "eu-central-1",
    "FR" => "eu-west-1",
    "SG" => "ap-southeast-1",
    "JP" => "ap-northeast-1",
    "AU" => "ap-southeast-1"
  }
  
  def select_optimal_region(country_code) do
    Map.get(@region_preferences, country_code, "us-east-1")
  end
  
  def get_model_with_region(model_preference, country_code) do
    region = select_optimal_region(country_code)
    
    # Check model availability in region
    case {model_preference, region} do
      {"claude-3-7-sonnet", region} when region in ["us-east-1", "us-west-2", "eu-west-1"] ->
        {model_preference, region}
      
      # Fallback to 3.5 Sonnet if 3.7 not available
      {"claude-3-7-sonnet", _region} ->
        {"claude-3-5-sonnet-v2", region}
      
      {model, region} ->
        {model, region}
    end
  end
end
```

## Dynamic Model Selection for Voice Agents

### Intelligent Model Selection Strategy
```elixir
defmodule Aybiza.ModelSelector.Intelligent do
  @simple_patterns [
    ~r/^(hi|hello|hey|good morning|good afternoon)/i,
    ~r/^(yes|no|okay|sure|thanks)/i,
    ~r/^(goodbye|bye|see you|talk to you later)/i
  ]
  
  @complex_patterns [
    ~r/(analyze|compare|explain why|walk me through)/i,
    ~r/(troubleshoot|debug|figure out|solve)/i,
    ~r/(calculate|compute|determine|evaluate)/i
  ]
  
  def select_model(input_text, context \\ %{}) do
    cond do
      # Ultra-fast for simple interactions
      matches_simple_pattern?(input_text) ->
        %{
          model: "anthropic.claude-3-5-haiku-20241022-v1:0",
          reasoning: "Simple interaction pattern detected",
          expected_latency: "80-120ms"
        }
      
      # Extended thinking for complex reasoning
      matches_complex_pattern?(input_text) && context[:allow_extended_thinking] ->
        %{
          model: "anthropic.claude-3-7-sonnet-20250219-v1:0",
          reasoning: "Complex reasoning required",
          expected_latency: "500-2000ms",
          thinking_budget: 3000
        }
      
      # Tool use required
      context[:requires_tools] ->
        %{
          model: "anthropic.claude-3-5-sonnet-20241022-v2:0",
          reasoning: "Tool use capability required",
          expected_latency: "150-200ms"
        }
      
      # Cost optimization mode
      context[:cost_optimize] ->
        %{
          model: "anthropic.claude-3-haiku-20240307-v1:0",
          reasoning: "Cost optimization requested",
          expected_latency: "100-150ms"
        }
      
      # Default balanced choice
      true ->
        %{
          model: "anthropic.claude-3-5-sonnet-20241022-v2:0",
          reasoning: "Balanced performance and capability",
          expected_latency: "150-200ms"
        }
    end
  end
  
  defp matches_simple_pattern?(text) do
    Enum.any?(@simple_patterns, &Regex.match?(&1, text))
  end
  
  defp matches_complex_pattern?(text) do
    Enum.any?(@complex_patterns, &Regex.match?(&1, text))
  end
end
```

## Streaming Implementation for Ultra-Low Latency

### Enhanced Streaming with Sentence-Level Processing
```elixir
defmodule Aybiza.VoicePipeline.StreamProcessor do
  def process_streaming_response(model_id, messages, options \\ %{}) do
    payload = %{
      "anthropic_version" => "bedrock-2023-05-31",
      "max_tokens" => Map.get(options, :max_tokens, 4096),
      "messages" => messages,
      "temperature" => Map.get(options, :temperature, 0.3),
      "stream" => true
    }
    
    # Add extended thinking if Claude 3.7 Sonnet
    payload = if String.contains?(model_id, "claude-3-7-sonnet") do
      Map.merge(payload, %{
        "thinking" => Map.get(options, :thinking, false),
        "budget_tokens" => Map.get(options, :thinking_budget, 3000)
      })
    else
      payload
    end
    
    stream = ExAws.Bedrock.invoke_model_with_response_stream(model_id, payload)
    
    # Process chunks and send to TTS immediately
    stream
    |> Stream.map(&decode_bedrock_chunk/1)
    |> Stream.each(&handle_content_chunk/1)
    |> Stream.run()
  end
  
  defp handle_content_chunk(%{"type" => "content_block_delta", "delta" => %{"text" => text}}) do
    # Accumulate text and detect sentence boundaries
    case extract_complete_sentences(text) do
      {[], _partial} -> :continue_accumulating
      {sentences, partial} ->
        # Send complete sentences to TTS immediately
        Enum.each(sentences, fn sentence ->
          Task.start(fn ->
            Aybiza.VoicePipeline.DeepgramTTS.synthesize_streaming(sentence)
          end)
        end)
        :sentences_processed
    end
  end
  
  defp extract_complete_sentences(text) do
    # Split on sentence boundaries
    sentences = Regex.split(~r/[.!?]\s+/, text, include_captures: true)
    
    case sentences do
      [] -> {[], text}
      [incomplete] -> {[], incomplete}
      sentences ->
        complete = Enum.take(sentences, length(sentences) - 1)
        |> Enum.chunk_every(2)
        |> Enum.map(fn [sentence, punct] -> sentence <> punct end)
        
        remaining = List.last(sentences)
        {complete, remaining}
    end
  end
end
```

## Cost Optimization Strategies

### Token Usage Optimization
```elixir
defmodule Aybiza.CostOptimizer do
  @cost_per_1k_tokens %{
    "anthropic.claude-3-7-sonnet-20250219-v1:0" => %{input: 3.00, output: 15.00},
    "anthropic.claude-3-5-sonnet-20241022-v2:0" => %{input: 3.00, output: 15.00},
    "anthropic.claude-3-5-haiku-20241022-v1:0" => %{input: 0.80, output: 4.00},
    "anthropic.claude-3-haiku-20240307-v1:0" => %{input: 0.25, output: 1.25}
  }
  
  def calculate_cost(model_id, input_tokens, output_tokens) do
    costs = @cost_per_1k_tokens[model_id]
    
    input_cost = (input_tokens / 1000) * costs.input
    output_cost = (output_tokens / 1000) * costs.output
    
    %{
      input_cost: input_cost,
      output_cost: output_cost,
      total_cost: input_cost + output_cost,
      cost_per_call: input_cost + output_cost
    }
  end
  
  def optimize_model_selection(conversation_history, user_input) do
    # Estimate complexity
    complexity = estimate_complexity(user_input, conversation_history)
    
    case complexity do
      :simple -> 
        # 80% cost reduction vs Sonnet
        {"anthropic.claude-3-haiku-20240307-v1:0", "Maximum cost efficiency"}
      
      :moderate ->
        # 73% cost reduction vs Sonnet, much better intelligence
        {"anthropic.claude-3-5-haiku-20241022-v1:0", "Balanced cost and intelligence"}
      
      :complex ->
        # Full capability, accept higher cost
        {"anthropic.claude-3-5-sonnet-20241022-v2:0", "Full intelligence required"}
      
      :very_complex ->
        # Extended thinking justified
        {"anthropic.claude-3-7-sonnet-20250219-v1:0", "Complex reasoning required"}
    end
  end
  
  defp estimate_complexity(input, history) do
    word_count = String.split(input) |> length()
    has_context = length(history) > 3
    
    cond do
      word_count < 5 && !has_context -> :simple
      word_count < 15 && !has_context -> :moderate
      word_count > 30 || has_context -> :complex
      contains_reasoning_keywords?(input) -> :very_complex
      true -> :moderate
    end
  end
  
  defp contains_reasoning_keywords?(input) do
    keywords = ["analyze", "compare", "explain why", "reasoning", "logic", "step by step"]
    text = String.downcase(input)
    Enum.any?(keywords, &String.contains?(text, &1))
  end
end
```

## System Prompts for Voice Agents (Enhanced)

### Voice-Optimized System Prompts
```elixir
defmodule Aybiza.Prompts.VoiceOptimized do
  @base_voice_prompt """
  You are an intelligent voice assistant. Your responses will be converted to speech, so optimize for spoken language.
  
  VOICE GUIDELINES:
  - Use natural, conversational language
  - Keep responses under 50 words when possible
  - Use contractions (don't, can't, I'll) for natural flow
  - Avoid complex punctuation that sounds awkward when spoken
  - End with clear closure or question when appropriate
  
  INTERACTION STYLE:
  - Be warm, professional, and helpful
  - Acknowledge the user's request before responding
  - Ask one clarifying question at a time
  - Confirm important information before proceeding
  """
  
  def get_system_prompt(agent_type, industry \\ nil) do
    base = @base_voice_prompt
    
    case agent_type do
      :customer_service ->
        base <> """
        
        CUSTOMER SERVICE FOCUS:
        - Prioritize resolving the customer's issue quickly
        - Show empathy for customer concerns
        - Escalate to human agent when needed
        - Always confirm next steps
        """
      
      :sales ->
        base <> """
        
        SALES FOCUS:
        - Listen actively to understand customer needs
        - Ask qualifying questions naturally
        - Focus on value, not features
        - Guide toward appropriate solutions
        """
      
      :technical_support ->
        base <> """
        
        TECHNICAL SUPPORT FOCUS:
        - Gather relevant technical information systematically
        - Explain technical concepts in simple terms
        - Provide step-by-step guidance
        - Confirm each step before proceeding
        """
    end
    |> add_industry_context(industry)
  end
  
  defp add_industry_context(prompt, industry) do
    case industry do
      :healthcare ->
        prompt <> """
        
        HEALTHCARE CONTEXT:
        - Maintain HIPAA compliance in all interactions
        - Never provide medical advice or diagnosis
        - Use appropriate medical terminology when needed
        - Show empathy for health-related concerns
        """
      
      :financial ->
        prompt <> """
        
        FINANCIAL CONTEXT:
        - Maintain strict confidentiality of financial information
        - Use precise language for financial terms
        - Never provide investment advice
        - Confirm account details securely
        """
      
      _ -> prompt
    end
  end
end
```

## Error Handling and Resilience

### Comprehensive Error Handling Strategy
```elixir
defmodule Aybiza.ErrorHandler.Bedrock do
  @fallback_models [
    "anthropic.claude-3-5-sonnet-20241022-v2:0",
    "anthropic.claude-3-5-haiku-20241022-v1:0",
    "anthropic.claude-3-haiku-20240307-v1:0"
  ]
  
  def handle_bedrock_error(error, context) do
    case error do
      {:error, %{"type" => "ThrottlingException"}} ->
        handle_throttling(context)
      
      {:error, %{"type" => "ModelNotReadyException"}} ->
        use_fallback_model(context)
      
      {:error, %{"type" => "ValidationException"}} ->
        fix_validation_error(context)
      
      {:error, %{"type" => "ServiceUnavailableException"}} ->
        switch_region(context)
      
      {:error, :timeout} ->
        use_faster_model(context)
      
      _ ->
        generic_fallback(context)
    end
  end
  
  defp handle_throttling(context) do
    # Implement exponential backoff with jitter
    delay = calculate_backoff_delay(context.retry_count)
    Process.sleep(delay)
    
    # Try with faster model to reduce load
    faster_model = get_faster_model(context.current_model)
    retry_with_model(faster_model, context)
  end
  
  defp use_fallback_model(context) do
    fallback = get_next_available_model(context.current_model)
    
    Logger.warning("Model #{context.current_model} not available, falling back to #{fallback}")
    
    retry_with_model(fallback, context)
  end
  
  defp switch_region(context) do
    # Try different region if current is unavailable
    alternative_region = get_alternative_region(context.region)
    
    context
    |> Map.put(:region, alternative_region)
    |> retry_request()
  end
  
  defp get_faster_model(current_model) do
    case current_model do
      "anthropic.claude-3-7-sonnet-20250219-v1:0" ->
        "anthropic.claude-3-5-sonnet-20241022-v2:0"
      
      "anthropic.claude-3-5-sonnet-20241022-v2:0" ->
        "anthropic.claude-3-5-haiku-20241022-v1:0"
      
      _ ->
        "anthropic.claude-3-haiku-20240307-v1:0"
    end
  end
  
  defp calculate_backoff_delay(retry_count) do
    # Exponential backoff with jitter
    base_delay = :math.pow(2, retry_count) * 1000
    jitter = :rand.uniform(500)
    trunc(base_delay + jitter)
  end
end
```

## Monitoring and Analytics

### Comprehensive Metrics Collection
```elixir
defmodule Aybiza.Metrics.Claude do
  def record_model_performance(model_id, metrics) do
    # Record to both CloudWatch and TimescaleDB
    
    # CloudWatch for AWS monitoring
    CloudWatch.put_metric_data("AYBIZA/Claude", [
      %{
        metric_name: "FirstTokenLatency",
        value: metrics.first_token_latency,
        unit: "Milliseconds",
        dimensions: [
          %{name: "ModelId", value: model_id},
          %{name: "Region", value: metrics.region},
          %{name: "EdgeRegion", value: metrics.edge_region}
        ]
      },
      %{
        metric_name: "TokensPerSecond",
        value: metrics.tokens_per_second,
        unit: "Count/Second",
        dimensions: [
          %{name: "ModelId", value: model_id}
        ]
      },
      %{
        metric_name: "CostPerCall",
        value: metrics.cost_usd,
        unit: "None",
        dimensions: [
          %{name: "ModelId", value: model_id}
        ]
      }
    ])
    
    # TimescaleDB for detailed analytics
    Aybiza.Repo.insert(%DetailedMetrics{
      timestamp: DateTime.utc_now(),
      call_sid: metrics.call_sid,
      tenant_id: metrics.tenant_id,
      metric_type: "llm_performance",
      metric_name: "model_usage",
      metric_value: metrics.total_latency,
      metadata: %{
        model_id: model_id,
        input_tokens: metrics.input_tokens,
        output_tokens: metrics.output_tokens,
        cost_usd: metrics.cost_usd,
        thinking_tokens: metrics.thinking_tokens,
        region: metrics.region,
        edge_region: metrics.edge_region
      }
    })
  end
  
  def get_model_performance_summary(period \\ "24h") do
    # Query TimescaleDB for performance insights
    query = """
    SELECT 
      metadata->>'model_id' as model,
      COUNT(*) as usage_count,
      AVG(metric_value) as avg_latency,
      PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY metric_value) as p95_latency,
      AVG((metadata->>'cost_usd')::float) as avg_cost,
      SUM((metadata->>'cost_usd')::float) as total_cost
    FROM detailed_metrics 
    WHERE metric_name = 'model_usage'
      AND timestamp > NOW() - INTERVAL '#{period}'
    GROUP BY metadata->>'model_id'
    ORDER BY usage_count DESC
    """
    
    Aybiza.Repo.query!(query)
  end
end
```

## Best Practices Summary (Updated 2025)

### 1. Model Selection Strategy
- **Default to Claude 3.5 Sonnet v2** for balanced performance
- **Use Claude 3.5 Haiku** for ultra-low latency (<150ms total)
- **Reserve Claude 3.7 Sonnet** for complex reasoning requiring extended thinking
- **Use Claude 3 Haiku** only for cost-critical simple applications
- **Avoid Claude 3 Opus** - migrate to Claude 3.7 Sonnet for better value

### 2. Latency Optimization
- **Always use streaming** for voice applications
- **Implement sentence-level TTS** triggering from streaming responses
- **Choose optimal AWS regions** based on user geography via Cloudflare edge
- **Pre-warm connections** and use connection pooling
- **Monitor first token latency** religiously

### 3. Cost Management
- **Implement dynamic model switching** based on complexity
- **Monitor token usage patterns** and optimize context management
- **Use Claude 3.5 Haiku** for 73% cost reduction on simple queries
- **Track cost per conversation** and set budgets
- **Optimize system prompts** to reduce unnecessary token usage

### 4. Quality Assurance
- **Use appropriate models** for task complexity
- **Implement proper system prompts** optimized for voice
- **Monitor conversation quality** through user feedback
- **A/B test different models** for specific use cases
- **Maintain context efficiently** for long conversations

### 5. Reliability and Monitoring
- **Implement comprehensive error handling** with model fallbacks
- **Use multi-region deployment** for high availability
- **Monitor all key metrics** (latency, cost, quality, errors)
- **Set up intelligent alerting** for performance degradation
- **Plan for capacity** and rate limit management

### 6. Security and Compliance
- **Validate all inputs** before sending to models
- **Implement prompt injection detection** for Claude 3.7's advanced capabilities
- **Monitor for sensitive data** in conversations
- **Use proper authentication** for Bedrock API calls
- **Maintain audit trails** for all model usage

This comprehensive reference ensures your AYBIZA voice agents leverage the full capabilities of the latest Claude models while maintaining optimal performance, cost-efficiency, and user experience in the hybrid Cloudflare+AWS architecture.