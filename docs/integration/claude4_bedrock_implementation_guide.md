# Claude 4 on AWS Bedrock Implementation Guide

## Overview
This guide provides comprehensive implementation details for using Claude 4 models (Opus and Sonnet) on AWS Bedrock for the AYBIZA voice agent platform. Claude 4 models are now available on AWS Bedrock with tool calling support through the Converse API.

## Model Availability and Features

### Claude 4 Opus (anthropic.claude-4-opus-20250120)
- **Availability**: ✅ Available on AWS Bedrock (January 2025)
- **Key Features**:
  - Extended thinking (1024-64000 token budget)
  - Parallel tool execution via Converse API
  - Advanced reasoning and complex analysis
  - 200K context window, 64K max output
- **Voice Agent Use**: Complex troubleshooting, multi-step workflows

### Claude 4 Sonnet (anthropic.claude-4-sonnet-20250120)
- **Availability**: ✅ Available on AWS Bedrock (January 2025)
- **Key Features**:
  - Balanced performance with tool support
  - <200ms response times
  - 200K context window, 32K max output
  - Cost-effective for production
- **Voice Agent Use**: Standard customer service, real-time interactions

## Implementation Patterns

**Note**: These examples use the [aws-elixir](https://github.com/aws-beam/aws-elixir) library for AWS SDK functionality. Add to your mix.exs:
```elixir
{:aws, "~> 1.0"},
{:hackney, "~> 1.20"},
{:jason, "~> 1.4"},
{:redix, "~> 1.5"}
```

### 1. Basic Setup
```elixir
# config/config.exs
config :aybiza, :bedrock,
  models: [
    "anthropic.claude-4-opus-20250120",
    "anthropic.claude-4-sonnet-20250120",
    "anthropic.claude-3-5-sonnet-20241022-v2:0",
    "anthropic.claude-3-5-haiku-20241022-v1:0"
  ],
  default_model: "anthropic.claude-4-sonnet-20250120",
  region: "us-east-1",
  fallback_region: "us-west-2"
```

### 2. Tool Calling with Converse API
```elixir
defmodule Aybiza.Claude4.ToolCalling do
  @doc """
  Execute tools with Claude 4 on Bedrock
  """
  def execute_with_tools(prompt, tools) do
    request = %{
      "modelId" => "anthropic.claude-4-sonnet-20250120",
      "messages" => [
        %{
          "role" => "user",
          "content" => [%{"text" => prompt}]
        }
      ],
      "toolConfig" => %{
        "tools" => format_tools_for_bedrock(tools),
        "toolChoice" => %{"auto" => %{}}
      },
      "inferenceConfig" => %{
        "maxTokens" => 4096,
        "temperature" => 0.0
      }
    }
    
    # Using aws_elixir or custom Bedrock client
    client = AWS.Client.create(region: "us-east-1")
    
    AWS.BedrockRuntime.converse(client, request)
    |> handle_tool_response()
  end
end
```

### 3. Extended Thinking Implementation
```elixir
defmodule Aybiza.Claude4.ExtendedThinking do
  @doc """
  Use Claude 4's extended thinking for complex queries
  """
  def think_deeply(prompt, thinking_budget \\ 32_000) do
    request = %{
      "modelId" => "anthropic.claude-4-opus-20250120",
      "messages" => format_messages(prompt),
      "system" => [%{"text" => "You are a helpful AI assistant. Think step by step."}],
      "inferenceConfig" => %{
        "maxTokens" => 8000,
        "temperature" => 0.1
      },
      "additionalModelRequestFields" => %{
        "thinking_enabled" => true,
        "thinking_budget_tokens" => thinking_budget
      }
    }
    
    # Stream the response using aws_elixir
    client = AWS.Client.create(region: "us-east-1")
    
    AWS.BedrockRuntime.converse_stream(client, request)
    |> process_thinking_stream()
  end
end
```

### 4. Memory Implementation
```elixir
defmodule Aybiza.Claude4.Memory do
  @doc """
  Conversation memory using DynamoDB and Redis
  """
  use GenServer
  
  def remember(call_id, message) do
    # Store in Redis for fast access using Redix
    {:ok, conn} = Redix.start_link()
    Redix.pipeline(conn, [
      ["LPUSH", "conv:#{call_id}", Jason.encode!(message)],
      ["EXPIRE", "conv:#{call_id}", 3600]
    ])
    
    # Persist to DynamoDB
    # Using ex_aws_dynamo library
    ExAws.Dynamo.put_item("conversations", %{
      "call_id" => %{"S" => call_id},
      "timestamp" => %{"S" => DateTime.to_iso8601(DateTime.utc_now())},
      "message" => %{"S" => Jason.encode!(message)}
    })
    |> ExAws.request()
  end
  
  def get_context(call_id, last_n \\ 10) do
    # Try Redis first using Redix
    {:ok, conn} = Redix.start_link()
    case Redix.command(conn, ["LRANGE", "conv:#{call_id}", 0, last_n - 1]) do
      {:ok, messages} -> Enum.map(messages, &Jason.decode!/1)
      _ -> fetch_from_dynamo(call_id, last_n)
    end
  end
end
```

## Feature Workarounds on Bedrock

### 1. Code Execution (Use Lambda)
```elixir
defmodule Aybiza.Claude4.CodeExecution do
  @doc """
  Execute code via Lambda instead of direct execution
  """
  def execute_code(code, language \\ "python") do
    # Create a Lambda-based tool
    code_tool = %{
      "toolSpec" => %{
        "name" => "execute_code",
        "description" => "Execute code in a sandboxed environment",
        "inputSchema" => %{
          "json" => %{
            "type" => "object",
            "properties" => %{
              "code" => %{"type" => "string"},
              "language" => %{"type" => "string"}
            }
          }
        }
      }
    }
    
    # Let Claude 4 call the tool
    request = build_tool_request([code_tool], 
      "Execute this code and return the results: #{code}")
    
    # The tool execution triggers Lambda
    execute_with_lambda_backend(request)
  end
end
```

### 2. Web Search (Use AWS Kendra)
```elixir
defmodule Aybiza.Claude4.WebSearch do
  @doc """
  Web search using AWS Kendra instead of native search
  """
  def search(query) do
    search_tool = %{
      "toolSpec" => %{
        "name" => "search_knowledge",
        "description" => "Search for information",
        "inputSchema" => %{
          "json" => %{
            "type" => "object",
            "properties" => %{
              "query" => %{"type" => "string"}
            }
          }
        }
      }
    }
    
    # Use Kendra as the search backend
    search_with_kendra(query)
  end
end
```

### 3. MCP Connectors (Use Standard Tools)
```elixir
defmodule Aybiza.Claude4.Connectors do
  @doc """
  Implement MCP-like functionality with standard tools
  """
  def create_connector(type, config) do
    # Define as a standard tool
    connector_tool = %{
      "toolSpec" => %{
        "name" => "#{type}_connector",
        "description" => "Connect to #{type} system",
        "inputSchema" => build_connector_schema(type)
      }
    }
    
    # Register the tool
    ToolRegistry.register(connector_tool, config)
  end
end
```

## Voice Agent Integration

### 1. Real-time Streaming
```elixir
defmodule Aybiza.VoiceAgent.Claude4 do
  def handle_voice_input(transcript, context) do
    # Select model based on complexity
    model = select_optimal_model(transcript, context)
    
    # Stream response for low latency
    stream = stream_claude4_response(model, transcript, context)
    
    # Process sentences as they arrive
    stream
    |> Stream.transform("", &extract_sentences/2)
    |> Stream.each(&send_to_tts/1)
    |> Stream.run()
  end
  
  defp select_optimal_model(transcript, context) do
    cond do
      needs_deep_thinking?(transcript) -> "anthropic.claude-4-opus-20250120"
      has_tool_requirements?(transcript) -> "anthropic.claude-4-sonnet-20250120"
      true -> "anthropic.claude-3-5-sonnet-20241022-v2:0"
    end
  end
end
```

### 2. Tool Integration Examples
```elixir
defmodule Aybiza.Tools.Claude4Tools do
  def voice_agent_tools() do
    [
      crm_lookup_tool(),
      appointment_booking_tool(),
      knowledge_search_tool(),
      call_transfer_tool(),
      email_tool()
    ]
  end
  
  defp crm_lookup_tool() do
    %{
      "toolSpec" => %{
        "name" => "lookup_customer",
        "description" => "Look up customer in CRM",
        "inputSchema" => %{
          "json" => %{
            "type" => "object",
            "properties" => %{
              "identifier" => %{
                "type" => "string",
                "description" => "Customer ID or phone"
              }
            },
            "required" => ["identifier"]
          }
        }
      }
    }
  end
end
```

## Performance Optimization

### 1. Model Selection Strategy
```elixir
def optimize_model_selection(request) do
  %{
    complexity: assess_complexity(request),
    latency_requirement: get_latency_requirement(request),
    has_tools: requires_tools?(request),
    cost_sensitive: is_cost_sensitive?(request)
  }
  |> select_best_model()
end

defp select_best_model(criteria) do
  case criteria do
    %{complexity: c} when c > 0.8 -> "anthropic.claude-4-opus-20250120"
    %{has_tools: true} -> "anthropic.claude-4-sonnet-20250120"
    %{latency_requirement: l} when l < 150 -> "anthropic.claude-3-5-haiku-20241022-v1:0"
    _ -> "anthropic.claude-3-5-sonnet-20241022-v2:0"
  end
end
```

### 2. Caching Strategy
```elixir
def with_caching(request) do
  cache_key = generate_cache_key(request)
  
  {:ok, conn} = Redix.start_link()
  case Redix.command(conn, ["GET", cache_key]) do
    {:ok, cached} when not is_nil(cached) -> 
      Logger.info("Cache hit for Claude 4 request")
      {:cached, Jason.decode!(cached)}
    
    _ ->
      response = execute_claude4_request(request)
      Redix.command(conn, ["SETEX", cache_key, 300, Jason.encode!(response)])
      {:fresh, response}
  end
end
```

## Cost Management

### Pricing Overview
- **Claude 4 Opus**: $15/MTok input, $75/MTok output
- **Claude 4 Sonnet**: $3/MTok input, $15/MTok output
- **Claude 3.5 Sonnet**: $3/MTok input, $15/MTok output
- **Claude 3.5 Haiku**: $0.80/MTok input, $4/MTok output

### Cost Optimization
```elixir
def calculate_request_cost(model, input_tokens, output_tokens) do
  prices = %{
    "anthropic.claude-4-opus-20250120" => {15.00, 75.00},
    "anthropic.claude-4-sonnet-20250120" => {3.00, 15.00},
    "anthropic.claude-3-5-sonnet-20241022-v2:0" => {3.00, 15.00},
    "anthropic.claude-3-5-haiku-20241022-v1:0" => {0.80, 4.00}
  }
  
  {input_price, output_price} = prices[model]
  
  input_cost = (input_tokens / 1_000_000) * input_price
  output_cost = (output_tokens / 1_000_000) * output_price
  
  input_cost + output_cost
end
```

## Monitoring and Debugging

### 1. Performance Metrics
```elixir
def track_claude4_performance(operation, metadata) do
  metrics = %{
    model: metadata.model,
    latency_ms: metadata.duration,
    input_tokens: metadata.input_tokens,
    output_tokens: metadata.output_tokens,
    thinking_tokens: metadata.thinking_tokens,
    tools_called: metadata.tools_count,
    cost_usd: calculate_request_cost(metadata)
  }
  
  # Send to CloudWatch
  CloudWatch.put_metric("Claude4Performance", metrics)
  
  # Log for analysis
  Logger.info("Claude 4 operation completed", metrics)
end
```

### 2. Error Handling
```elixir
def handle_claude4_errors(error) do
  case error do
    {:error, "ThrottlingException"} ->
      # Switch to fallback model
      use_fallback_model()
    
    {:error, "ModelNotReadyException"} ->
      # Retry with different region
      retry_in_alternate_region()
    
    {:error, "ValidationException"} ->
      # Log and fix request
      log_validation_error(error)
      fix_and_retry()
    
    _ ->
      # Generic fallback
      use_cached_response_or_default()
  end
end
```

## Best Practices

### 1. Voice-Optimized Prompts
```elixir
def enhance_for_voice(prompt) do
  """
  #{prompt}
  
  VOICE INTERACTION GUIDELINES:
  - Keep responses under 50 words for quick delivery
  - Use natural, conversational language
  - End with a clear question or next step
  - Avoid complex punctuation
  - Use contractions for natural flow
  """
end
```

### 2. Streaming Optimization
```elixir
def optimize_streaming(stream) do
  stream
  |> Stream.chunk_by(&sentence_boundary?/1)
  |> Stream.map(&join_chunks/1)
  |> Stream.filter(&complete_sentence?/1)
  |> Stream.map(&prepare_for_tts/1)
end
```

## Migration Guide

### From Claude 3.x to Claude 4
1. Update model IDs in configuration
2. Add tool configuration for Converse API
3. Implement memory management
4. Update error handling for new exceptions
5. Adjust token limits for larger outputs
6. Implement cost tracking

### Testing Claude 4 Features
```elixir
defmodule Claude4Test do
  use ExUnit.Case
  
  test "tool calling works correctly" do
    tools = [test_calculator_tool()]
    result = Claude4.execute_with_tools("What is 15% of 200?", tools)
    assert result == {:ok, "30"}
  end
  
  test "extended thinking produces detailed analysis" do
    result = Claude4.think_deeply("Analyze the pros and cons of microservices")
    assert String.length(result) > 1000
    assert result =~ "advantages"
    assert result =~ "disadvantages"
  end
end
```

## Conclusion

Claude 4 models on AWS Bedrock provide powerful capabilities for AYBIZA voice agents:
- ✅ Tool calling via Converse API
- ✅ Extended thinking for complex queries
- ✅ Streaming for low-latency responses
- ✅ Memory management with external storage
- ⚠️ Some features require workarounds (code execution, MCP)

The combination of Claude 4's advanced capabilities with AWS Bedrock's infrastructure provides a robust foundation for sophisticated voice agent interactions while maintaining the security and scalability benefits of the AWS ecosystem.