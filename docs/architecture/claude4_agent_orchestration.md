# AYBIZA AI Voice Agent Platform - Claude 4 Agent Orchestration

## Overview
This document details the implementation of Claude 4 Opus and Sonnet models within AYBIZA's billion-scale voice agent platform. The orchestration layer manages extended thinking, parallel tool execution, MCP connectors, and intelligent model selection while maintaining <100ms latency for voice interactions.

## Architecture Overview

### Agent Orchestration Flow
```
Voice Input → Model Selection → Claude 4 Processing → Tool Execution → Response Generation
     ↓              ↓                    ↓                  ↓                ↓
  Context      Complexity         Extended Thinking    Parallel Tools    Streaming TTS
  Manager      Analysis          & Prompt Caching      & MCP/Code       & Natural Flow
```

## Claude 4 Model Capabilities

### Model Specifications
| Model | ID | Best For | Latency Target | Cost |
|-------|-----|----------|----------------|------|
| Claude 4 Opus | anthropic.claude-opus-4-20250514-v1:0 | Extended thinking, code execution | 500ms | $15/$75 per M tokens |
| Claude 4 Sonnet | anthropic.claude-sonnet-4-20250514-v1:0 | Balanced performance, parallel tools | 200ms | $3/$15 per M tokens |
| Claude 3.7 Sonnet | anthropic.claude-3-7-sonnet-20250219-v1:0 | Complex reasoning | 300ms | $3/$15 per M tokens |
| Claude 3.5 Sonnet v2 | anthropic.claude-3-5-sonnet-20241022-v2:0 | Standard queries | 150ms | $3/$15 per M tokens |
| Claude 3.5 Haiku | anthropic.claude-3-5-haiku-20241022-v1:0 | Ultra-fast responses | 50ms | $0.25/$1.25 per M tokens |

### Key Features
1. **Extended Thinking**: Token-based budget (1024-64000 tokens) - requires beta header
2. **Parallel Tool Use**: Execute multiple tools concurrently (no documented limit)
3. **Prompt Caching**: 5-minute default TTL (1-hour in beta) with 90% cost reduction
4. **Code Execution**: Python 3.11.12 (1 GiB RAM, 5 GiB storage) - requires beta header
5. **MCP Integration**: Not available on AWS Bedrock (API/SDK only)
6. **Files API**: 32 MB limit, not available on AWS Bedrock
7. **Web Search**: $10 per 1000 searches (API/SDK only)

## Implementation

### 1. Model Selection Engine
```elixir
defmodule Aybiza.Agents.ModelSelector do
  @moduledoc """
  Intelligent model selection based on query complexity and requirements
  """
  
  @model_configs %{
    claude_opus_4: %{
      id: "anthropic.claude-opus-4-20250514-v1:0",
      thinking_budget_tokens: 64_000,  # Max extended thinking tokens
      supports_code: false,  # Not on Bedrock
      supports_mcp: false,   # Not on Bedrock
      supports_extended_thinking: true,
      cost_factor: 5.0
    },
    claude_sonnet_4: %{
      id: "anthropic.claude-sonnet-4-20250514-v1:0",
      thinking_budget_tokens: 32_000,  # Moderate thinking budget
      supports_code: false,  # Not on Bedrock
      supports_mcp: false,   # Not on Bedrock
      supports_extended_thinking: true,
      cost_factor: 1.0
    },
    claude_3_7_sonnet: %{
      id: "anthropic.claude-3-7-sonnet-20250219-v1:0",
      thinking_budget_tokens: 0,  # No extended thinking
      supports_code: false,
      supports_mcp: false,
      supports_extended_thinking: false,
      cost_factor: 1.0
    },
    claude_3_5_sonnet: %{
      id: "anthropic.claude-3-5-sonnet-20241022-v2:0",
      thinking_budget_tokens: 0,  # No extended thinking
      supports_code: false,
      supports_mcp: false,
      supports_extended_thinking: false,
      cost_factor: 1.0
    },
    claude_3_5_haiku: %{
      id: "anthropic.claude-3-5-haiku-20241022-v1:0",
      thinking_budget_tokens: 0,  # No extended thinking
      supports_code: false,
      supports_mcp: false,
      supports_extended_thinking: false,
      cost_factor: 0.08
    }
  }
  
  def select_model(context) do
    complexity = analyze_complexity(context)
    requirements = extract_requirements(context)
    
    model = cond do
      # Extended thinking required
      complexity.requires_extended_thinking ->
        :claude_opus_4
      
      # Code execution needed
      requirements.code_execution ->
        :claude_opus_4
      
      # Multiple tools or MCP
      length(requirements.tools) > 3 or requirements.mcp_required ->
        :claude_sonnet_4
      
      # Complex reasoning without tools
      complexity.score > 0.7 ->
        :claude_3_7_sonnet
      
      # Standard queries
      complexity.score > 0.3 ->
        :claude_3_5_sonnet
      
      # Simple, fast responses
      true ->
        :claude_3_5_haiku
    end
    
    {model, @model_configs[model]}
  end
  
  defp analyze_complexity(context) do
    %{
      score: calculate_complexity_score(context),
      requires_extended_thinking: needs_extended_thinking?(context),
      estimated_tokens: estimate_token_count(context),
      conversation_depth: length(context.history)
    }
  end
  
  defp needs_extended_thinking?(context) do
    keywords = ["analyze", "explain in detail", "step by step", "compare", "evaluate"]
    prompt = String.downcase(context.current_prompt)
    
    Enum.any?(keywords, &String.contains?(prompt, &1)) or
    String.length(prompt) > 500 or
    context.explicit_thinking_request
  end
end
```

### 2. Claude 4 Client with Bedrock
```elixir
defmodule Aybiza.Agents.Claude4Client do
  @moduledoc """
  AWS Bedrock client for Claude 4 models with enhanced features
  """
  
  alias AWS.Client
  alias AWS.BedrockRuntime
  
  @cache_ttl 300  # 5 minutes default (use beta header for 1 hour)
  
  def converse_with_claude(model_config, messages, options \\ %{}) do
    # Build request with caching
    request = build_converse_request(model_config, messages, options)
    
    # Check prompt cache
    cache_key = generate_cache_key(request)
    case PromptCache.get(cache_key) do
      {:hit, cached_response} ->
        {:ok, %{cached_response | cached: true}}
      
      :miss ->
        # Make API call
        response = BedrockRuntime.converse(
          client(),
          model_config.id,
          request
        )
        
        # Cache successful responses
        case response do
          {:ok, result} ->
            PromptCache.put(cache_key, result, ttl: @cache_ttl)
            {:ok, result}
          
          error -> error
        end
    end
  end
  
  defp build_converse_request(model_config, messages, options) do
    %{
      messages: format_messages(messages),
      system: build_system_prompt(options),
      inferenceConfig: %{
        maxTokens: options[:max_tokens] || 4000,
        temperature: options[:temperature] || 0.3,
        topP: options[:top_p] || 0.95
      },
      toolConfig: build_tool_config(options[:tools])
    }
    |> maybe_add_thinking_config(model_config, options)
    |> maybe_add_cache_control(options)
  end
  
  defp maybe_add_thinking_config(request, model_config, options) do
    if options[:enable_thinking] and model_config.supports_extended_thinking do
      # Extended thinking uses token budget, not time
      # Note: This feature requires beta header on API/SDK
      Map.put(request, :thinkingConfig, %{
        enabled: true,
        budgetTokens: options[:thinking_budget] || model_config.thinking_budget_tokens
      })
    else
      request
    end
  end
  
  defp maybe_add_cache_control(request, options) do
    if options[:enable_caching] do
      Map.put(request, :cacheControl, %{
        enabled: true,
        ttlSeconds: @cache_ttl
      })
    else
      request
    end
  end
  
  defp build_tool_config(nil), do: nil
  defp build_tool_config(tools) do
    %{
      tools: Enum.map(tools, &format_tool_spec/1),
      toolChoice: %{type: "auto"}
    }
  end
end
```

### 3. Extended Thinking Handler
```elixir
defmodule Aybiza.Agents.ExtendedThinking do
  @moduledoc """
  Manages extended thinking mode for complex reasoning
  """
  
  use GenServer
  
  def process_with_thinking(agent_id, prompt, context, options) do
    GenServer.call(
      via_tuple(agent_id),
      {:process_with_thinking, prompt, context, options},
      35_000  # 35s timeout for 30s thinking
    )
  end
  
  @impl true
  def handle_call({:process_with_thinking, prompt, context, options}, _from, state) do
    # Acknowledge thinking to user
    send_to_voice("I need to think about this for a moment...")
    
    # Start thinking process
    thinking_task = Task.async(fn ->
      execute_extended_thinking(prompt, context, options, state)
    end)
    
    # Monitor thinking progress (token-based, not time-based)
    result = monitor_thinking_progress(thinking_task, options[:thinking_budget] || 64_000)
    
    {:reply, result, update_state_with_result(state, result)}
  end
  
  defp execute_extended_thinking(prompt, context, options, state) do
    # Build enhanced prompt for thinking
    thinking_prompt = """
    <thinking>
    You have up to 30 seconds to think through this problem step by step.
    Break down complex problems into smaller parts.
    Consider multiple approaches before settling on a solution.
    Show your work and reasoning.
    </thinking>
    
    User Query: #{prompt}
    
    Context:
    #{format_context(context)}
    """
    
    # Process with Claude 4 Opus
    Claude4Client.converse_with_claude(
      %{id: "anthropic.claude-opus-4-20250514-v1:0"},
      [%{role: "user", content: thinking_prompt}],
      Map.merge(options, %{
        enable_thinking: true,
        enable_caching: true,
        stream: true,
        stream_callback: &handle_thinking_stream/2
      })
    )
  end
  
  defp handle_thinking_stream(chunk, state) do
    case chunk do
      %{thinking: thinking_content} ->
        # Internal thinking, don't send to user
        Logger.debug("Thinking: #{thinking_content}")
        {:continue, state}
      
      %{content: content} ->
        # Actual response, send to TTS
        send_to_voice_streaming(content)
        {:continue, state}
      
      %{tool_use: tool_request} ->
        # Execute tool during thinking
        Task.start(fn -> execute_tool_async(tool_request) end)
        {:continue, state}
    end
  end
end
```

### 4. Parallel Tool Orchestrator
```elixir
defmodule Aybiza.Agents.ParallelToolOrchestrator do
  @moduledoc """
  Manages parallel execution of multiple tools
  """
  
  # No documented limit on parallel tools in official docs
  @max_parallel_tools 20  # Conservative limit for performance
  @tool_timeout 5_000
  
  def execute_tools_parallel(tool_requests, context) do
    # Validate tool requests
    validated_requests = validate_and_prepare_tools(tool_requests, context)
    
    # Group by execution priority
    grouped = group_by_priority(validated_requests)
    
    # Execute in waves
    results = Enum.reduce(grouped, [], fn {_priority, tools}, acc ->
      wave_results = execute_tool_wave(tools, context)
      acc ++ wave_results
    end)
    
    aggregate_results(results)
  end
  
  defp execute_tool_wave(tools, context) do
    tools
    |> Enum.take(@max_parallel_tools)
    |> Task.async_stream(
      fn tool -> execute_single_tool(tool, context) end,
      timeout: @tool_timeout,
      on_timeout: :kill_task,
      max_concurrency: @max_parallel_tools
    )
    |> Enum.map(fn
      {:ok, result} -> result
      {:exit, :timeout} -> %{tool: tool, error: :timeout}
      {:exit, reason} -> %{tool: tool, error: reason}
    end)
  end
  
  defp execute_single_tool(tool, context) do
    start_time = System.monotonic_time(:millisecond)
    
    result = case tool.type do
      # Note: Web search costs $10 per 1000 searches (API only)
      :web_search -> 
        Logger.warn("Web search not available on Bedrock")
        {:error, :not_on_bedrock}
      
      # Code execution requires beta header (API only)
      :code_execution -> 
        Logger.warn("Code execution not available on Bedrock")
        {:error, :not_on_bedrock}
      
      # MCP requires API/SDK
      :mcp_connector -> 
        Logger.warn("MCP not available on Bedrock")
        {:error, :not_on_bedrock}
      
      # Files API requires API/SDK
      :files_api -> 
        Logger.warn("Files API not available on Bedrock, using S3 fallback")
        S3FilesAdapter.execute(tool.params, context)
      
      :database -> DatabaseTool.execute(tool.params, context)
      _ -> GenericTool.execute(tool)
    end
    
    %{
      tool: tool,
      result: result,
      duration_ms: System.monotonic_time(:millisecond) - start_time,
      timestamp: DateTime.utc_now()
    }
  end
end
```

### 5. MCP Connector Integration
```elixir
defmodule Aybiza.Agents.MCPConnector do
  @moduledoc """
  Model Context Protocol connector for external system integration
  """
  
  @supported_connectors %{
    "hubspot" => Aybiza.MCP.HubSpotConnector,
    "postgresql" => Aybiza.MCP.PostgreSQLConnector,
    "google_drive" => Aybiza.MCP.GoogleDriveConnector,
    "slack" => Aybiza.MCP.SlackConnector,
    "github" => Aybiza.MCP.GitHubConnector
  }
  
  def connect(connector_type, config) do
    case @supported_connectors[connector_type] do
      nil -> 
        {:error, :unsupported_connector}
      
      connector_module ->
        connector_module.establish_connection(config)
    end
  end
  
  def execute_operation(connector, operation, params) do
    # Validate operation permissions
    with :ok <- validate_permissions(connector, operation),
         :ok <- validate_params(operation, params) do
      
      # Execute with timeout
      Task.async(fn ->
        connector.execute(operation, params)
      end)
      |> Task.await(5000)
    end
  end
  
  defmodule HubSpotConnector do
    @behaviour Aybiza.MCP.ConnectorBehaviour
    
    def establish_connection(config) do
      # Initialize HubSpot client with credentials
      client = HubSpot.Client.new(
        api_key: config.api_key,
        timeout: 5000
      )
      
      {:ok, %{client: client, config: config}}
    end
    
    def execute("search_contacts", params, connector) do
      connector.client
      |> HubSpot.Contacts.search(params.query, params.options)
      |> format_response()
    end
    
    def execute("create_deal", params, connector) do
      connector.client
      |> HubSpot.Deals.create(params.deal_data)
      |> format_response()
    end
  end
end
```

### 6. Code Execution Sandbox
```elixir
defmodule Aybiza.Agents.CodeExecutionSandbox do
  @moduledoc """
  Secure Python code execution for Claude 4 agents
  """
  
  @python_runtime "python3.11"
  @max_execution_time 3000
  @max_memory "256MB"
  
  def execute_python(code, options \\ %{}) do
    # Prepare sandboxed environment
    sandbox_id = generate_sandbox_id()
    
    # Create isolated container
    {:ok, container} = create_sandbox_container(sandbox_id, options)
    
    try do
      # Execute code with limits
      result = Docker.Container.exec(container, [
        @python_runtime,
        "-c",
        wrap_code_with_safety(code)
      ], [
        timeout: options[:timeout] || @max_execution_time,
        memory: options[:memory] || @max_memory,
        network: false  # No network access
      ])
      
      parse_execution_result(result)
    after
      # Clean up container
      Docker.Container.stop(container)
      Docker.Container.remove(container)
    end
  end
  
  defp wrap_code_with_safety(code) do
    """
    import sys
    import json
    import resource
    
    # Set resource limits
    resource.setrlimit(resource.RLIMIT_CPU, (3, 3))
    resource.setrlimit(resource.RLIMIT_AS, (256 * 1024 * 1024, 256 * 1024 * 1024))
    
    # Disable dangerous imports
    sys.modules['os'] = None
    sys.modules['subprocess'] = None
    sys.modules['socket'] = None
    
    # User code execution
    try:
        exec('''
    #{code}
    ''')
        print(json.dumps({"status": "success", "output": locals()}))
    except Exception as e:
        print(json.dumps({"status": "error", "error": str(e)}))
    """
  end
end
```

### 7. Files API Integration
```elixir
defmodule Aybiza.Agents.FilesAPI do
  @moduledoc """
  Persistent file storage for agent sessions
  """
  
  def store_file(agent_id, file_data, metadata \\ %{}) do
    file_key = generate_file_key(agent_id, metadata)
    
    # Store in S3
    AWS.S3.put_object(
      client(),
      bucket(),
      file_key,
      %{
        body: file_data,
        metadata: metadata,
        server_side_encryption: "AES256"
      }
    )
    
    # Store metadata in database
    %AgentFile{
      agent_id: agent_id,
      file_key: file_key,
      file_name: metadata.file_name,
      file_type: metadata.file_type,
      file_size_bytes: byte_size(file_data),
      content_hash: :crypto.hash(:sha256, file_data) |> Base.encode16()
    }
    |> Repo.insert()
  end
  
  def retrieve_file(agent_id, file_id) do
    case Repo.get_by(AgentFile, agent_id: agent_id, id: file_id) do
      nil -> 
        {:error, :not_found}
      
      file ->
        # Get from S3
        AWS.S3.get_object(client(), bucket(), file.file_key)
        |> case do
          {:ok, %{body: data}} -> 
            {:ok, %{file: file, data: data}}
          error -> 
            error
        end
    end
  end
  
  def list_files(agent_id, options \\ %{}) do
    AgentFile
    |> where(agent_id: ^agent_id)
    |> apply_filters(options)
    |> Repo.all()
  end
end
```

### 8. Voice Integration Layer
```elixir
defmodule Aybiza.Agents.VoiceIntegration do
  @moduledoc """
  Integrates Claude 4 agents with voice pipeline
  """
  
  def handle_voice_turn(call_context, user_input) do
    # 1. Select model based on context
    {model, config} = ModelSelector.select_model(%{
      current_prompt: user_input,
      history: call_context.conversation_history,
      explicit_thinking_request: contains_thinking_request?(user_input)
    })
    
    # 2. Process with appropriate handler
    response = case model do
      :claude_opus_4 when config.supports_extended_thinking ->
        ExtendedThinking.process_with_thinking(
          call_context.agent_id,
          user_input,
          call_context,
          %{model_config: config}
        )
      
      _ ->
        Claude4Client.converse_with_claude(
          config,
          build_messages(call_context, user_input),
          %{
            stream: true,
            enable_caching: true,
            tools: get_available_tools(call_context)
          }
        )
    end
    
    # 3. Handle response with streaming
    handle_streaming_response(response, call_context)
  end
  
  defp handle_streaming_response({:ok, stream}, call_context) do
    stream
    |> Stream.each(fn chunk ->
      case chunk do
        %{content: text} ->
          # Send to TTS immediately
          VoicePipeline.send_to_tts(call_context.call_id, text)
        
        %{tool_use: tools} ->
          # Acknowledge and execute
          VoicePipeline.speak(call_context.call_id, "Let me check that for you...")
          ParallelToolOrchestrator.execute_tools_parallel(tools, call_context)
        
        _ -> :ok
      end
    end)
    |> Stream.run()
  end
end
```

## Performance Optimization

### 1. Prompt Caching Strategy
```elixir
defmodule Aybiza.Agents.PromptCache do
  @moduledoc """
  1-hour TTL cache for 90% cost reduction
  """
  
  use GenServer
  
  @ttl 300_000  # 5 minutes default (API limit without beta header)
  @max_cache_size 10_000
  
  def get(key) do
    case :ets.lookup(:prompt_cache, key) do
      [{^key, value, expiry}] when expiry > System.monotonic_time(:millisecond) ->
        {:hit, value}
      _ ->
        :miss
    end
  end
  
  def put(key, value, opts \\ []) do
    ttl = Keyword.get(opts, :ttl, @ttl)
    expiry = System.monotonic_time(:millisecond) + ttl
    
    :ets.insert(:prompt_cache, {key, value, expiry})
    
    # Cleanup old entries if needed
    if :ets.info(:prompt_cache, :size) > @max_cache_size do
      cleanup_expired()
    end
  end
end
```

### 2. Latency Monitoring
```elixir
defmodule Aybiza.Agents.LatencyMonitor do
  @moduledoc """
  Tracks and optimizes agent response latency
  """
  
  def track_latency(operation, metadata) do
    start_time = System.monotonic_time(:millisecond)
    
    result = yield()
    
    duration = System.monotonic_time(:millisecond) - start_time
    
    # Store metric
    %LatencyMetric{
      operation: operation,
      duration_ms: duration,
      metadata: metadata,
      timestamp: DateTime.utc_now()
    }
    |> Repo.insert_async()
    
    # Alert if over threshold
    if duration > metadata.threshold_ms do
      AlertManager.send_latency_alert(operation, duration, metadata)
    end
    
    result
  end
end
```

## Cost Management

### Dynamic Model Selection for Cost
```elixir
defmodule Aybiza.Agents.CostOptimizer do
  @moduledoc """
  Optimizes model selection based on cost constraints
  """
  
  @model_costs %{
    claude_opus_4: %{input: 0.015, output: 0.075},
    claude_sonnet_4: %{input: 0.003, output: 0.015},
    claude_3_7_sonnet: %{input: 0.003, output: 0.015},
    claude_3_5_sonnet: %{input: 0.003, output: 0.015},
    claude_3_5_haiku: %{input: 0.00025, output: 0.00125}
  }
  
  def select_cost_optimal_model(requirements, budget_constraints) do
    eligible_models = filter_by_requirements(requirements)
    
    # Calculate estimated cost for each model
    model_costs = Enum.map(eligible_models, fn model ->
      estimated_cost = calculate_estimated_cost(
        model,
        requirements.estimated_input_tokens,
        requirements.estimated_output_tokens
      )
      
      {model, estimated_cost}
    end)
    
    # Select cheapest that meets requirements
    model_costs
    |> Enum.filter(fn {_model, cost} -> cost <= budget_constraints.max_cost end)
    |> Enum.min_by(fn {_model, cost} -> cost end, fn -> {:claude_3_5_haiku, 0} end)
    |> elem(0)
  end
end
```

## Monitoring & Observability

### Agent Performance Dashboard
```elixir
defmodule Aybiza.Agents.PerformanceDashboard do
  @moduledoc """
  Real-time monitoring of Claude 4 agent performance
  """
  
  def collect_metrics(agent_id) do
    %{
      model_usage: get_model_usage_stats(agent_id),
      latency_percentiles: calculate_latency_percentiles(agent_id),
      tool_execution_stats: get_tool_stats(agent_id),
      cost_breakdown: calculate_cost_breakdown(agent_id),
      cache_hit_rate: calculate_cache_hit_rate(agent_id),
      error_rates: get_error_rates(agent_id)
    }
  end
  
  defp calculate_latency_percentiles(agent_id) do
    LatencyMetric
    |> where(agent_id: ^agent_id)
    |> where([m], m.timestamp > ago(1, "hour"))
    |> select([m], m.duration_ms)
    |> Repo.all()
    |> Statistics.percentiles([50, 90, 95, 99])
  end
end
```

## Security Considerations

### 1. Tool Execution Security
- All tools run in isolated environments
- Network access disabled for code execution
- Customer credentials stored in KMS with tenant isolation
- Audit logging for all tool executions

### 2. Prompt Injection Protection
- Input validation before Claude 4 processing
- Output filtering for sensitive data
- Rate limiting per tenant
- Emergency kill switch for runaway agents

## Best Practices

### 1. Model Selection
- Use Haiku for simple, fast responses
- Use Sonnet 4 for standard queries with tools
- Use Opus 4 only for complex reasoning or code
- Always enable prompt caching

### 2. Tool Design
- Keep tool execution under 500ms
- Use parallel execution for independent tools
- Cache tool results when appropriate
- Provide clear tool descriptions for Claude

### 3. Voice Integration
- Stream responses at sentence boundaries
- Acknowledge tool use naturally
- Keep extended thinking under 5 seconds for voice
- Use appropriate voice feedback during processing

## Future Enhancements

### 1. Advanced Features
- Multi-agent collaboration
- Persistent agent memory across calls
- Custom model fine-tuning
- Advanced reasoning chains

### 2. Integration Expansions
- Additional MCP connectors
- Custom tool development SDK
- WebAssembly code execution
- Real-time data streaming

## AWS SDK Configuration

This module uses the aws-elixir library for AWS SDK functionality. Helper function:

```elixir
defp client do
  AWS.Client.create(
    Application.get_env(:aybiza, :aws_access_key_id),
    Application.get_env(:aybiza, :aws_secret_access_key),
    Application.get_env(:aybiza, :aws_region, "us-east-1")
  )
end
```

This orchestration system enables AYBIZA to leverage Claude 4's advanced capabilities while maintaining the ultra-low latency required for natural voice conversations at billion-user scale.