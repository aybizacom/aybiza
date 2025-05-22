# Infinite Loop Prevention and DoS Protection Guide

## Overview

This document extends AYBIZA's security framework with specific guidance on preventing infinite loops and protecting against DoS attacks that could be caused by uncontrolled iterations, recursive calls, or malicious inputs.

## Infinite Loop Prevention Strategies

### 1. AI Conversation Loop Detection

```elixir
defmodule Aybiza.Security.ConversationLoopDetector do
  @moduledoc """
  Detects and prevents infinite loops in AI conversations.
  """
  
  @max_conversation_turns 100
  @max_identical_responses 3
  @similarity_threshold 0.95
  @max_token_growth_rate 2.0
  
  def check_conversation_health(conversation_id, messages) do
    checks = [
      check_turn_limit(messages),
      check_repetitive_content(messages),
      check_token_explosion(messages),
      check_circular_prompting(messages)
    ]
    
    case Enum.find(checks, &match?({:error, _}, &1)) do
      nil -> :ok
      error -> error
    end
  end
  
  defp check_turn_limit(messages) do
    if length(messages) > @max_conversation_turns do
      {:error, :conversation_too_long}
    else
      :ok
    end
  end
  
  defp check_repetitive_content(messages) do
    recent_messages = Enum.take(messages, @max_identical_responses)
    
    if has_identical_responses?(recent_messages) do
      {:error, :repetitive_responses}
    else
      :ok
    end
  end
  
  defp check_token_explosion(messages) do
    if length(messages) >= 10 do
      recent_tokens = messages
                     |> Enum.take(10)
                     |> Enum.map(&count_tokens/1)
      
      growth_rate = calculate_token_growth_rate(recent_tokens)
      
      if growth_rate > @max_token_growth_rate do
        {:error, :token_explosion}
      else
        :ok
      end
    else
      :ok
    end
  end
  
  defp check_circular_prompting(messages) do
    # Detect if user is trying to manipulate the AI into circular responses
    user_messages = Enum.filter(messages, &(&1.role == "user"))
    
    if has_prompt_injection_loop?(user_messages) do
      {:error, :prompt_injection_loop}
    else
      :ok
    end
  end
  
  defp has_identical_responses?(messages) do
    contents = Enum.map(messages, & &1.content)
    length(Enum.uniq(contents)) == 1 and length(contents) >= @max_identical_responses
  end
  
  defp calculate_token_growth_rate(token_counts) do
    if length(token_counts) < 2 do
      1.0
    else
      recent = Enum.take(token_counts, 5) |> Enum.sum()
      older = Enum.drop(token_counts, 5) |> Enum.sum()
      
      if older == 0, do: 1.0, else: recent / older
    end
  end
  
  defp has_prompt_injection_loop?(user_messages) do
    # Check for patterns that might cause AI to repeat indefinitely
    injection_patterns = [
      ~r/repeat.*indefinitely/i,
      ~r/keep saying.*forever/i,
      ~r/don't stop.*until/i,
      ~r/infinite.*loop/i
    ]
    
    Enum.any?(user_messages, fn message ->
      Enum.any?(injection_patterns, &Regex.match?(&1, message.content))
    end)
  end
  
  defp count_tokens(message) do
    # Approximate token count (real implementation would use tokenizer)
    String.length(message.content) |> div(4)
  end
end
```

### 2. Process Execution Guards

```elixir
defmodule Aybiza.Security.ExecutionGuard do
  @moduledoc """
  Prevents infinite loops in process execution and external service calls.
  """
  
  def with_iteration_guard(fun, opts \\ []) do
    max_iterations = opts[:max_iterations] || 1000
    timeout = opts[:timeout] || 30_000
    memory_limit = opts[:memory_limit_mb] || 100
    
    task = Task.async(fn ->
      # Set process memory limit
      :erlang.process_flag(:max_heap_size, memory_limit * 1024 * 1024)
      
      # Execute with iteration counting
      execute_with_guards(fun, max_iterations, 0, System.monotonic_time())
    end)
    
    case Task.yield(task, timeout) || Task.shutdown(task) do
      {:ok, result} -> result
      nil -> {:error, :execution_timeout}
    end
  end
  
  defp execute_with_guards(fun, max_iterations, current_iteration, start_time) do
    # Check iteration limit
    if current_iteration >= max_iterations do
      {:error, :max_iterations_exceeded}
    else
      # Check execution time
      elapsed = System.monotonic_time() - start_time
      if System.convert_time_unit(elapsed, :native, :millisecond) > 25_000 do
        {:error, :execution_timeout}
      else
        case fun.() do
          {:continue, new_fun} ->
            execute_with_guards(new_fun, max_iterations, current_iteration + 1, start_time)
          {:continue, new_fun, new_opts} ->
            updated_max = new_opts[:max_iterations] || max_iterations
            execute_with_guards(new_fun, updated_max, current_iteration + 1, start_time)
          result ->
            result
        end
      end
    end
  end
end
```

### 3. External Service Loop Prevention

```elixir
defmodule Aybiza.Security.ServiceLoopPrevention do
  @moduledoc """
  Prevents infinite loops when calling external services.
  """
  
  def with_service_guard(service_name, fun, opts \\ []) do
    max_retries = opts[:max_retries] || 3
    backoff_base = opts[:backoff_base] || 1000
    max_backoff = opts[:max_backoff] || 30_000
    
    execute_with_backoff(service_name, fun, 0, max_retries, backoff_base, max_backoff)
  end
  
  defp execute_with_backoff(service_name, fun, attempt, max_retries, backoff_base, max_backoff) do
    case fun.() do
      {:ok, result} ->
        {:ok, result}
      
      {:error, :rate_limited} when attempt < max_retries ->
        backoff_time = min(backoff_base * :math.pow(2, attempt), max_backoff)
        
        Logger.warn("Service #{service_name} rate limited, backing off for #{backoff_time}ms")
        Process.sleep(round(backoff_time))
        
        execute_with_backoff(service_name, fun, attempt + 1, max_retries, backoff_base, max_backoff)
      
      {:error, :timeout} when attempt < max_retries ->
        Logger.warn("Service #{service_name} timeout, retrying (attempt #{attempt + 1}/#{max_retries})")
        execute_with_backoff(service_name, fun, attempt + 1, max_retries, backoff_base, max_backoff)
      
      {:error, reason} = error ->
        Logger.error("Service #{service_name} failed: #{inspect(reason)}")
        error
    end
  end
end
```

### 4. Memory and CPU Protection

```elixir
defmodule Aybiza.Security.ResourceProtection do
  @moduledoc """
  Protects against resource exhaustion that could lead to DoS.
  """
  
  def with_resource_limits(fun, opts \\ []) do
    memory_limit_mb = opts[:memory_limit_mb] || 50
    cpu_time_limit_ms = opts[:cpu_time_limit_ms] || 10_000
    
    # Start monitored process
    {pid, ref} = spawn_monitor(fn ->
      # Set memory limit
      :erlang.process_flag(:max_heap_size, memory_limit_mb * 1024 * 1024)
      
      # Execute with CPU time monitoring
      start_time = :erlang.monotonic_time(:millisecond)
      
      result = try do
        fun.()
      catch
        :exit, :kill -> {:error, :resource_limit_exceeded}
        error -> {:error, error}
      end
      
      end_time = :erlang.monotonic_time(:millisecond)
      cpu_time = end_time - start_time
      
      if cpu_time > cpu_time_limit_ms do
        {:error, :cpu_time_exceeded}
      else
        result
      end
    end)
    
    # Wait for completion with timeout
    receive do
      {:DOWN, ^ref, :process, ^pid, :normal} -> :ok
      {:DOWN, ^ref, :process, ^pid, reason} -> {:error, reason}
    after
      opts[:total_timeout] || 30_000 ->
        Process.exit(pid, :kill)
        {:error, :total_timeout}
    end
  end
end
```

### 5. Database Query Protection

```elixir
defmodule Aybiza.Security.DatabaseProtection do
  @moduledoc """
  Prevents infinite loops and resource exhaustion in database queries.
  """
  
  def safe_query(query, opts \\ []) do
    max_rows = opts[:max_rows] || 10_000
    timeout = opts[:timeout] || 30_000
    
    # Add LIMIT clause if not present
    limited_query = ensure_limit(query, max_rows)
    
    # Execute with timeout
    Task.async(fn ->
      Aybiza.Repo.all(limited_query)
    end)
    |> Task.await(timeout)
  rescue
    Task.TimeoutError ->
      {:error, :query_timeout}
  end
  
  def safe_recursive_query(initial_query, iteration_fun, opts \\ []) do
    max_iterations = opts[:max_iterations] || 100
    max_total_rows = opts[:max_total_rows] || 50_000
    
    execute_recursive_query(initial_query, iteration_fun, [], 0, max_iterations, max_total_rows)
  end
  
  defp execute_recursive_query(_query, _fun, results, iteration, max_iterations, _max_rows) 
       when iteration >= max_iterations do
    {:error, :max_iterations_exceeded, results}
  end
  
  defp execute_recursive_query(_query, _fun, results, _iteration, _max_iterations, max_rows) 
       when length(results) >= max_rows do
    {:error, :max_rows_exceeded, results}
  end
  
  defp execute_recursive_query(query, iteration_fun, results, iteration, max_iterations, max_rows) do
    case Aybiza.Repo.all(query) do
      [] ->
        {:ok, results}
      
      new_results ->
        all_results = results ++ new_results
        next_query = iteration_fun.(new_results, iteration)
        
        execute_recursive_query(
          next_query, 
          iteration_fun, 
          all_results, 
          iteration + 1, 
          max_iterations, 
          max_rows
        )
    end
  end
  
  defp ensure_limit(query, max_rows) do
    # Check if query already has a LIMIT
    if has_limit?(query) do
      query
    else
      from(q in query, limit: ^max_rows)
    end
  end
  
  defp has_limit?(%Ecto.Query{limit: nil}), do: false
  defp has_limit?(%Ecto.Query{limit: _}), do: true
end
```

## Integration with Existing Security Framework

### 1. Enhanced Circuit Breaker

```elixir
defmodule Aybiza.Security.EnhancedCircuitBreaker do
  @moduledoc """
  Enhanced circuit breaker with loop detection.
  """
  
  use Aybiza.CircuitBreaker
  
  def call_with_loop_detection(name, fun, opts \\ []) do
    loop_detector = opts[:loop_detector] || &default_loop_detector/2
    
    case call(name, fun) do
      {:ok, result} ->
        # Check for loop patterns in the result
        case loop_detector.(name, result) do
          :no_loop -> {:ok, result}
          {:loop_detected, reason} ->
            # Force circuit open
            force_open(name, reason)
            {:error, :loop_detected}
        end
      
      error -> error
    end
  end
  
  defp default_loop_detector(service_name, result) do
    # Check if we're getting identical responses repeatedly
    cache_key = "circuit_breaker_responses:#{service_name}"
    recent_responses = Aybiza.Cache.get(cache_key) || []
    
    updated_responses = [result | recent_responses] |> Enum.take(5)
    Aybiza.Cache.put(cache_key, updated_responses, ttl: 300)
    
    if length(updated_responses) >= 3 and length(Enum.uniq(updated_responses)) == 1 do
      {:loop_detected, :identical_responses}
    else
      :no_loop
    end
  end
end
```

### 2. Rate Limiter Enhancement

```elixir
defmodule Aybiza.Security.EnhancedRateLimiter do
  @moduledoc """
  Enhanced rate limiter with burst detection and loop prevention.
  """
  
  def check_rate_with_burst_detection(key, window_ms, limit, opts \\ []) do
    burst_threshold = opts[:burst_threshold] || limit * 0.8
    burst_window = opts[:burst_window] || window_ms / 10
    
    # Check normal rate limit
    case Hammer.check_rate(key, window_ms, limit) do
      {:allow, count} ->
        # Also check for burst patterns
        burst_key = "burst:#{key}"
        case Hammer.check_rate(burst_key, burst_window, round(burst_threshold)) do
          {:allow, _} -> {:allow, count}
          {:deny, _} -> {:deny, :burst_detected}
        end
      
      {:deny, reason} -> {:deny, reason}
    end
  end
  
  def check_rate_with_pattern_detection(key, window_ms, limit, opts \\ []) do
    # Check for suspicious patterns
    pattern_key = "pattern:#{key}"
    request_times = Aybiza.Cache.get(pattern_key) || []
    
    current_time = System.system_time(:millisecond)
    updated_times = [current_time | request_times]
                   |> Enum.filter(&(&1 > current_time - window_ms))
                   |> Enum.take(limit * 2)
    
    Aybiza.Cache.put(pattern_key, updated_times, ttl: window_ms)
    
    case detect_suspicious_pattern(updated_times) do
      :normal_pattern ->
        Hammer.check_rate(key, window_ms, limit)
      
      {:suspicious_pattern, reason} ->
        Logger.warn("Suspicious rate limit pattern detected: #{reason}")
        {:deny, :suspicious_pattern}
    end
  end
  
  defp detect_suspicious_pattern(request_times) when length(request_times) < 10 do
    :normal_pattern
  end
  
  defp detect_suspicious_pattern(request_times) do
    # Check for perfectly regular intervals (bot-like behavior)
    intervals = request_times
               |> Enum.chunk_every(2, 1, :discard)
               |> Enum.map(fn [a, b] -> abs(a - b) end)
    
    # If all intervals are nearly identical, it's suspicious
    if length(Enum.uniq_by(intervals, &round(&1 / 100))) == 1 do
      {:suspicious_pattern, :regular_intervals}
    else
      :normal_pattern
    end
  end
end
```

## Implementation Guidelines

### 1. AI Service Integration

```elixir
defmodule Aybiza.AI.SafeBedrockClient do
  @moduledoc """
  Bedrock client with infinite loop protection.
  """
  
  alias Aybiza.Security.ConversationLoopDetector
  alias Aybiza.Security.ExecutionGuard
  
  def invoke_with_protection(client, request, conversation_history \\ []) do
    # Check conversation health first
    case ConversationLoopDetector.check_conversation_health(request.conversation_id, conversation_history) do
      :ok ->
        # Execute with guards
        ExecutionGuard.with_iteration_guard(fn ->
          Aybiza.AWS.BedrockClient.invoke(client, request)
        end, max_iterations: 1, timeout: 30_000)
      
      {:error, reason} ->
        Logger.warn("Conversation loop detected: #{reason}")
        {:error, :conversation_loop_detected}
    end
  end
end
```

### 2. Background Job Protection

```elixir
defmodule Aybiza.Jobs.SafeJob do
  @moduledoc """
  Base job with infinite loop protection.
  """
  
  defmacro __using__(opts) do
    quote do
      use Oban.Worker
      alias Aybiza.Security.ExecutionGuard
      
      @impl Oban.Worker
      def perform(%Oban.Job{args: args} = job) do
        max_iterations = unquote(opts)[:max_iterations] || 1000
        timeout = unquote(opts)[:timeout] || 300_000  # 5 minutes
        
        ExecutionGuard.with_iteration_guard(
          fn -> execute_job(args) end,
          max_iterations: max_iterations,
          timeout: timeout
        )
      end
      
      def execute_job(args) do
        {:error, :not_implemented}
      end
      
      defoverridable execute_job: 1
    end
  end
end
```

## Testing Infinite Loop Protection

### 1. Conversation Loop Tests

```elixir
defmodule Aybiza.Security.ConversationLoopDetectorTest do
  use ExUnit.Case
  
  alias Aybiza.Security.ConversationLoopDetector
  
  test "detects repetitive responses" do
    messages = [
      %{role: "user", content: "Hello"},
      %{role: "assistant", content: "Hello there!"},
      %{role: "user", content: "How are you?"},
      %{role: "assistant", content: "Hello there!"},
      %{role: "user", content: "What's your name?"},
      %{role: "assistant", content: "Hello there!"}
    ]
    
    assert {:error, :repetitive_responses} = 
      ConversationLoopDetector.check_conversation_health("conv_1", messages)
  end
  
  test "detects token explosion" do
    # Create messages with exponentially growing content
    messages = 1..15
               |> Enum.map(fn i ->
                 content = String.duplicate("word ", round(:math.pow(2, i)))
                 %{role: "assistant", content: content}
               end)
    
    assert {:error, :token_explosion} = 
      ConversationLoopDetector.check_conversation_health("conv_2", messages)
  end
end
```

### 2. Resource Protection Tests

```elixir
defmodule Aybiza.Security.ResourceProtectionTest do
  use ExUnit.Case
  
  alias Aybiza.Security.ResourceProtection
  
  test "prevents memory exhaustion" do
    # Function that tries to allocate too much memory
    memory_hog = fn ->
      Enum.reduce(1..1_000_000, [], fn i, acc -> [i | acc] end)
    end
    
    result = ResourceProtection.with_resource_limits(
      memory_hog,
      memory_limit_mb: 10,
      total_timeout: 5000
    )
    
    assert {:error, :resource_limit_exceeded} = result
  end
  
  test "prevents CPU time exhaustion" do
    # Function that uses too much CPU time
    cpu_hog = fn ->
      Enum.reduce(1..10_000_000, 0, fn i, acc -> acc + i end)
    end
    
    result = ResourceProtection.with_resource_limits(
      cpu_hog,
      cpu_time_limit_ms: 100
    )
    
    assert {:error, :cpu_time_exceeded} = result
  end
end
```

## Monitoring and Alerting

### 1. Loop Detection Metrics

```elixir
defmodule Aybiza.Monitoring.LoopMetrics do
  @moduledoc """
  Metrics collection for loop detection events.
  """
  
  def record_loop_detection(type, tenant_id, metadata \\ %{}) do
    # Record to metrics system
    Aybiza.Metrics.increment("security.loop_detection", %{
      type: type,
      tenant_id: tenant_id
    })
    
    # Log for analysis
    Logger.warn("Loop detected", 
      type: type, 
      tenant_id: tenant_id, 
      metadata: metadata
    )
    
    # Alert if critical
    if type in [:conversation_loop, :infinite_recursion] do
      Aybiza.Alerts.send_alert(%{
        severity: :high,
        service: "loop_detection",
        message: "Critical loop detected: #{type}",
        tenant_id: tenant_id,
        metadata: metadata
      })
    end
  end
end
```

## Conclusion

These infinite loop prevention measures provide comprehensive protection against:

1. **AI Conversation Loops**: Repetitive responses, token explosion, circular prompting
2. **Process Execution Loops**: Uncontrolled iterations, recursive calls
3. **Resource Exhaustion**: Memory and CPU limits
4. **Database Query Loops**: Recursive queries, unlimited result sets
5. **External Service Loops**: Retry storms, cascading failures

The implementation builds on your existing security framework while adding specific protections against infinite loops and DoS attacks through malicious or malformed inputs.