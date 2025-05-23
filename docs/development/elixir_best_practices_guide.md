# AYBIZA Elixir Best Practices Guide - Updated

## Overview
This guide provides comprehensive best practices for developing the AYBIZA platform using Elixir/OTP with our hybrid Cloudflare+AWS architecture. These practices ensure code quality, performance, reliability, and maintainability across all applications.

## Technology Stack (Current)
- **Elixir**: 1.18.3+ (with experimental features enabled)
- **Erlang/OTP**: 28.0+ (with JIT compilation)
- **Phoenix**: 1.7.21+ (with LiveView 1.1.0+)
- **PostgreSQL**: 16.9+ (with TimescaleDB 2.20.0+)
- **Redis**: 7.4+ (for session management and caching)
- **Cloudflare Workers**: Edge computing runtime
- **AWS Services**: Bedrock, Lambda, S3, RDS, etc.

## 1. Application Structure and Organization

### 1.1 Umbrella Project Structure
```elixir
# apps/voice_gateway/lib/voice_gateway/application.ex
defmodule VoiceGateway.Application do
  use Application
  
  @impl true
  def start(_type, _args) do
    children = [
      # Edge-first supervision strategy
      {VoiceGateway.EdgeSupervisor, []},
      {VoiceGateway.TwilioSupervisor, []},
      {VoiceGateway.WebSocketSupervisor, []},
      {VoiceGateway.HealthCheck, []},
      
      # Telemetry and monitoring
      {VoiceGateway.Telemetry, []},
      {VoiceGateway.MetricsCollector, []}
    ]
    
    opts = [strategy: :one_for_one, name: VoiceGateway.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

### 1.2 Module Organization Best Practices
```elixir
# Good: Clear module hierarchy with proper namespacing
defmodule Aybiza.ConversationEngine.Claude do
  @moduledoc """
  Enhanced Claude integration with 3.7 Sonnet support and extended thinking.
  
  Provides intelligent model selection based on conversation complexity,
  latency requirements, and regional optimization for hybrid architecture.
  """
  
  alias Aybiza.AWS.BedrockClient
  alias Aybiza.ConversationEngine.Context
  alias Aybiza.Analytics.MetricsCollector
  
  @type conversation_mode :: :standard | :extended_thinking | :low_latency
  @type model_selection :: :auto | :claude_3_7_sonnet | :claude_3_5_sonnet | :claude_3_5_haiku
  
  @spec generate_response(String.t(), Context.t(), keyword()) :: 
    {:ok, String.t()} | {:error, term()}
  def generate_response(prompt, context, opts \\ []) do
    with {:ok, model_config} <- select_optimal_model(context, opts),
         {:ok, response} <- BedrockClient.invoke_model(model_config),
         :ok <- MetricsCollector.record_conversation_metrics(context, response) do
      {:ok, response.content}
    end
  end
  
  # Private functions for internal logic
  defp select_optimal_model(%Context{} = context, opts) do
    complexity = analyze_conversation_complexity(context)
    latency_requirement = Keyword.get(opts, :latency_target, :standard)
    region = determine_optimal_region(context)
    
    case {complexity, latency_requirement, region} do
      {:high, _, _} -> {:ok, build_model_config(:claude_3_7_sonnet, region)}
      {:medium, :low_latency, _} -> {:ok, build_model_config(:claude_3_5_haiku, region)}
      {:medium, _, _} -> {:ok, build_model_config(:claude_3_5_sonnet, region)}
      {:low, _, _} -> {:ok, build_model_config(:claude_3_5_haiku, region)}
    end
  end
end
```

## 2. Supervision and Fault Tolerance

### 2.1 Hybrid Architecture Supervision Strategy
```elixir
# Enhanced supervision for edge-cloud hybrid processing
defmodule Aybiza.HybridSupervisor do
  use Supervisor
  
  def start_link(init_arg) do
    Supervisor.start_link(__MODULE__, init_arg, name: __MODULE__)
  end
  
  @impl true
  def init(_init_arg) do
    children = [
      # Edge processing tier (Cloudflare Workers coordination)
      {Aybiza.Edge.WorkerCoordinator, []},
      {Aybiza.Edge.HealthMonitor, []},
      
      # Core processing tier
      {Aybiza.VoicePipeline.ProcessingSupervisor, []},
      {Aybiza.ConversationEngine.Supervisor, []},
      
      # Backend service tier
      {Aybiza.AWS.ServicesSupervisor, []},
      {Aybiza.Analytics.ProcessingSupervisor, []},
      
      # Circuit breakers for hybrid coordination
      {Aybiza.Reliability.CircuitBreakerSupervisor, []}
    ]
    
    # Use rest_for_one to ensure proper cascading restarts
    Supervisor.init(children, strategy: :rest_for_one, max_restarts: 3, max_seconds: 10)
  end
end
```

### 2.2 GenServer Best Practices for Voice Processing
```elixir
defmodule Aybiza.VoicePipeline.AudioProcessor do
  use GenServer
  require Logger
  
  @initial_state %{
    processing_queue: :queue.new(),
    active_streams: %{},
    performance_metrics: %{},
    circuit_breaker_state: :closed
  }
  
  # Client API with proper timeout handling
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  @spec process_audio_chunk(binary(), String.t()) :: {:ok, binary()} | {:error, term()}
  def process_audio_chunk(audio_data, stream_id) do
    # Use shorter timeout for real-time processing
    GenServer.call(__MODULE__, {:process_chunk, audio_data, stream_id}, 1_000)
  catch
    :exit, {:timeout, _} ->
      Logger.warn("Audio processing timeout for stream #{stream_id}")
      {:error, :timeout}
  end
  
  # Server callbacks with comprehensive error handling
  @impl true
  def init(opts) do
    # Initialize with performance monitoring
    :timer.send_interval(5_000, :collect_metrics)
    :timer.send_interval(30_000, :health_check)
    
    {:ok, Map.merge(@initial_state, Map.new(opts))}
  end
  
  @impl true
  def handle_call({:process_chunk, audio_data, stream_id}, _from, state) do
    case state.circuit_breaker_state do
      :open ->
        {:reply, {:error, :circuit_breaker_open}, state}
      
      _ ->
        case process_audio_internal(audio_data, stream_id, state) do
          {:ok, processed_audio, new_state} ->
            {:reply, {:ok, processed_audio}, new_state}
          
          {:error, reason, new_state} ->
            updated_state = handle_processing_error(reason, new_state)
            {:reply, {:error, reason}, updated_state}
        end
    end
  end
  
  @impl true
  def handle_info(:collect_metrics, state) do
    metrics = collect_performance_metrics(state)
    Aybiza.Analytics.MetricsCollector.record_voice_processing_metrics(metrics)
    {:noreply, %{state | performance_metrics: metrics}}
  end
  
  @impl true
  def handle_info(:health_check, state) do
    case perform_health_check(state) do
      :healthy ->
        {:noreply, %{state | circuit_breaker_state: :closed}}
      
      :unhealthy ->
        Logger.warn("Voice processor health check failed, opening circuit breaker")
        {:noreply, %{state | circuit_breaker_state: :open}}
    end
  end
  
  # Private helper functions
  defp process_audio_internal(audio_data, stream_id, state) do
    start_time = System.monotonic_time(:millisecond)
    
    try do
      processed = apply_audio_processing(audio_data)
      processing_time = System.monotonic_time(:millisecond) - start_time
      
      updated_metrics = update_performance_metrics(state.performance_metrics, processing_time)
      new_state = %{state | performance_metrics: updated_metrics}
      
      {:ok, processed, new_state}
    rescue
      error ->
        Logger.error("Audio processing failed: #{inspect(error)}")
        {:error, :processing_failed, state}
    end
  end
end
```

## 3. Performance Optimization for Real-Time Voice

### 3.1 Memory Management and Garbage Collection
```elixir
# Optimize for low-latency voice processing
defmodule Aybiza.Performance.MemoryOptimizer do
  @moduledoc """
  Memory optimization utilities for ultra-low latency voice processing.
  Target: <50ms for US/UK, <100ms globally.
  """
  
  # Use binary pattern matching for efficient audio chunk processing
  def process_audio_chunks(<<chunk::binary-size(1024), rest::binary>>) do
    # Process in 1KB chunks to minimize GC pressure
    processed_chunk = process_single_chunk(chunk)
    [processed_chunk | process_audio_chunks(rest)]
  end
  
  def process_audio_chunks(<<final_chunk::binary>>) when byte_size(final_chunk) > 0 do
    [process_single_chunk(final_chunk)]
  end
  
  def process_audio_chunks(<<>>) do
    []
  end
  
  # Minimize allocations in hot paths
  defp process_single_chunk(chunk) do
    # Use iodata to avoid unnecessary binary concatenation
    [apply_filter(chunk), apply_normalization(chunk)]
    |> IO.iodata_to_binary()
  end
  
  # Configure GC for real-time processing
  def configure_process_for_realtime(pid \\ self()) do
    # Reduce GC frequency for real-time processes
    :erlang.process_flag(:min_heap_size, 16_384)
    :erlang.process_flag(:min_bin_vheap_size, 8_192)
    
    # Set higher priority for voice processing
    Process.flag(:priority, :high)
  end
end
```

### 3.2 Streaming and Backpressure Management
```elixir
defmodule Aybiza.VoicePipeline.StreamProcessor do
  use GenStage
  require Logger
  
  @max_demand 10
  @buffer_size 100
  
  def start_link(opts \\ []) do
    GenStage.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  # Producer-consumer for audio streaming
  @impl true
  def init(opts) do
    buffer_size = Keyword.get(opts, :buffer_size, @buffer_size)
    
    state = %{
      buffer: :queue.new(),
      buffer_size: buffer_size,
      demand: 0,
      processing_times: []
    }
    
    {:producer_consumer, state}
  end
  
  @impl true
  def handle_subscribe(:consumer, _opts, _from, state) do
    {:automatic, state}
  end
  
  @impl true
  def handle_demand(incoming_demand, state) do
    %{buffer: buffer, demand: current_demand} = state
    new_demand = current_demand + incoming_demand
    
    {events, remaining_buffer} = take_events_from_buffer(buffer, new_demand)
    new_state = %{state | buffer: remaining_buffer, demand: new_demand - length(events)}
    
    {:noreply, events, new_state}
  end
  
  @impl true
  def handle_events(events, _from, state) do
    start_time = System.monotonic_time(:microsecond)
    
    # Process events with performance tracking
    processed_events = 
      events
      |> Enum.map(&process_audio_event/1)
      |> Enum.filter(&filter_valid_events/1)
    
    processing_time = System.monotonic_time(:microsecond) - start_time
    updated_state = track_performance(state, processing_time)
    
    # Apply backpressure if processing is too slow
    case should_apply_backpressure?(updated_state) do
      true ->
        Logger.warn("Applying backpressure due to slow processing")
        {:noreply, [], updated_state}
      
      false ->
        {:noreply, processed_events, updated_state}
    end
  end
  
  defp should_apply_backpressure?(state) do
    avg_processing_time = calculate_average_processing_time(state.processing_times)
    avg_processing_time > 45_000  # 45ms threshold for US/UK
  end
end
```

## 4. Database Patterns with PostgreSQL 16.9+ and TimescaleDB

### 4.1 Ecto Schema Best Practices
```elixir
defmodule Aybiza.Schema.CallRecord do
  use Ecto.Schema
  import Ecto.Changeset
  
  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
  @timestamps_opts [type: :utc_datetime_usec]
  
  schema "call_records" do
    field :call_sid, :string, null: false
    field :phone_number, :string, null: false
    field :direction, Ecto.Enum, values: [:inbound, :outbound], null: false
    field :status, Ecto.Enum, 
      values: [:initiated, :ringing, :answered, :completed, :failed], 
      null: false
    
    # Performance metrics with microsecond precision
    field :latency_ms, :integer
    field :processing_time_us, :integer
    field :audio_quality_score, :decimal, precision: 5, scale: 3
    
    # Hybrid architecture tracking
    field :edge_region, :string
    field :processing_region, :string
    field :cost_optimization_applied, :boolean, default: false
    
    # TimescaleDB timestamp for time-series queries
    field :call_timestamp, :utc_datetime_usec, null: false
    
    # Relationships
    belongs_to :agent, Aybiza.Schema.Agent
    has_many :conversation_turns, Aybiza.Schema.ConversationTurn
    has_many :analytics_events, Aybiza.Schema.AnalyticsEvent
    
    timestamps()
  end
  
  @required_fields [:call_sid, :phone_number, :direction, :status, :call_timestamp]
  @optional_fields [:latency_ms, :processing_time_us, :audio_quality_score, 
                   :edge_region, :processing_region, :cost_optimization_applied, :agent_id]
  
  def changeset(call_record, attrs) do
    call_record
    |> cast(attrs, @required_fields ++ @optional_fields)
    |> validate_required(@required_fields)
    |> validate_inclusion(:direction, [:inbound, :outbound])
    |> validate_inclusion(:status, [:initiated, :ringing, :answered, :completed, :failed])
    |> validate_number(:latency_ms, greater_than: 0, less_than: 10_000)
    |> validate_number(:audio_quality_score, greater_than_or_equal_to: 0, less_than_or_equal_to: 1)
    |> validate_format(:phone_number, ~r/^\+[1-9]\d{1,14}$/)
    |> unique_constraint(:call_sid)
    |> foreign_key_constraint(:agent_id)
  end
  
  # Helper function for TimescaleDB time-bucket queries
  def time_bucket_query(bucket_size \\ "1 minute") do
    from cr in __MODULE__,
      select: %{
        time_bucket: fragment("time_bucket(?, ?)", ^bucket_size, cr.call_timestamp),
        call_count: count(cr.id),
        avg_latency: avg(cr.latency_ms),
        avg_quality: avg(cr.audio_quality_score)
      },
      group_by: fragment("time_bucket(?, ?)", ^bucket_size, cr.call_timestamp),
      order_by: fragment("time_bucket(?, ?)", ^bucket_size, cr.call_timestamp)
  end
end
```

### 4.2 Repository Patterns with Performance Optimization
```elixir
defmodule Aybiza.Repo.CallRecords do
  import Ecto.Query
  alias Aybiza.Repo
  alias Aybiza.Schema.CallRecord
  
  @doc """
  Optimized bulk insert for high-volume call records.
  Uses PostgreSQL COPY for maximum performance.
  """
  def bulk_insert_call_records(records) when is_list(records) do
    # Validate and prepare records
    changesets = Enum.map(records, &CallRecord.changeset(%CallRecord{}, &1))
    
    case Enum.find(changesets, &(!&1.valid?)) do
      nil ->
        # All valid, proceed with bulk insert
        records_data = Enum.map(changesets, &Ecto.Changeset.apply_changes/1)
        
        Repo.transaction(fn ->
          {count, _} = Repo.insert_all(CallRecord, records_data, 
            on_conflict: :nothing,
            conflict_target: :call_sid,
            returning: [:id]
          )
          count
        end)
      
      invalid_changeset ->
        {:error, invalid_changeset}
    end
  end
  
  @doc """
  Real-time analytics query optimized for TimescaleDB.
  Returns call metrics for the last hour with 1-minute resolution.
  """
  def get_realtime_call_metrics(agent_id, opts \\ []) do
    time_range = Keyword.get(opts, :time_range, "1 hour")
    bucket_size = Keyword.get(opts, :bucket_size, "1 minute")
    
    query = 
      from cr in CallRecord,
        where: cr.agent_id == ^agent_id and 
               cr.call_timestamp > ago(^time_range, "hour"),
        select: %{
          time_bucket: fragment("time_bucket(?, ?)", ^bucket_size, cr.call_timestamp),
          total_calls: count(),
          completed_calls: filter(count(), cr.status == :completed),
          avg_latency_ms: avg(cr.latency_ms),
          avg_quality_score: avg(cr.audio_quality_score),
          cost_optimized_calls: filter(count(), cr.cost_optimization_applied == true)
        },
        group_by: fragment("time_bucket(?, ?)", ^bucket_size, cr.call_timestamp),
        order_by: fragment("time_bucket(?, ?)", ^bucket_size, cr.call_timestamp)
    
    Repo.all(query)
  end
  
  @doc """
  Hybrid architecture performance analysis.
  Compares edge vs cloud processing performance.
  """
  def analyze_hybrid_performance(date_range) do
    query = 
      from cr in CallRecord,
        where: cr.call_timestamp >= ^date_range.start and 
               cr.call_timestamp <= ^date_range.end,
        select: %{
          edge_region: cr.edge_region,
          processing_region: cr.processing_region,
          call_count: count(),
          avg_latency_ms: avg(cr.latency_ms),
          p95_latency: fragment("percentile_cont(0.95) WITHIN GROUP (ORDER BY ?)", cr.latency_ms),
          avg_quality: avg(cr.audio_quality_score),
          cost_savings: avg(fragment("CASE WHEN ? THEN 1 ELSE 0 END", cr.cost_optimization_applied))
        },
        group_by: [cr.edge_region, cr.processing_region],
        order_by: [desc: :avg_latency_ms]
    
    Repo.all(query)
  end
end
```

## 5. Error Handling and Logging

### 5.1 Comprehensive Error Handling Patterns
```elixir
defmodule Aybiza.ErrorHandler do
  require Logger
  
  @doc """
  Standardized error handling for voice processing pipeline.
  Includes circuit breaker logic and automatic retries.
  """
  def handle_voice_processing_error(error, context) do
    case classify_error(error) do
      :transient ->
        Logger.warn("Transient voice processing error: #{inspect(error)}", context: context)
        schedule_retry(context)
        {:error, :retry_scheduled}
      
      :permanent ->
        Logger.error("Permanent voice processing error: #{inspect(error)}", context: context)
        notify_monitoring_system(error, context)
        {:error, :permanent_failure}
      
      :rate_limit ->
        Logger.warn("Rate limit exceeded: #{inspect(error)}", context: context)
        apply_backpressure(context)
        {:error, :rate_limited}
      
      :circuit_breaker ->
        Logger.error("Circuit breaker triggered: #{inspect(error)}", context: context)
        switch_to_fallback_processing(context)
        {:error, :circuit_breaker_open}
    end
  end
  
  @doc """
  Structured logging for hybrid architecture debugging.
  """
  def log_hybrid_processing_event(event_type, metadata) do
    base_metadata = %{
      timestamp: DateTime.utc_now(),
      node: Node.self(),
      environment: Application.get_env(:aybiza, :environment)
    }
    
    full_metadata = Map.merge(base_metadata, metadata)
    
    case event_type do
      :edge_processing_started ->
        Logger.info("Edge processing initiated", full_metadata)
      
      :edge_to_cloud_handoff ->
        Logger.info("Handoff from edge to cloud processing", full_metadata)
      
      :processing_completed ->
        Logger.info("Voice processing completed", full_metadata)
      
      :performance_threshold_exceeded ->
        Logger.warn("Performance threshold exceeded", full_metadata)
      
      :cost_optimization_applied ->
        Logger.info("Cost optimization strategy applied", full_metadata)
    end
  end
  
  defp classify_error(%{status_code: code}) when code in [429, 503, 504], do: :transient
  defp classify_error(%{status_code: code}) when code in [400, 401, 403], do: :permanent
  defp classify_error(%{reason: :timeout}), do: :transient
  defp classify_error(%{reason: :circuit_breaker_open}), do: :circuit_breaker
  defp classify_error(_), do: :permanent
end
```

### 5.2 Telemetry and Observability
```elixir
defmodule Aybiza.Telemetry do
  use Supervisor
  require Logger
  
  def start_link(init_arg) do
    Supervisor.start_link(__MODULE__, init_arg, name: __MODULE__)
  end
  
  @impl true
  def init(_init_arg) do
    children = [
      # Telemetry metrics
      {TelemetryMetrics.Supervisor, metrics()},
      
      # Custom telemetry handlers
      {Aybiza.Telemetry.VoiceMetricsHandler, []},
      {Aybiza.Telemetry.HybridArchitectureHandler, []},
      {Aybiza.Telemetry.PerformanceMonitor, []}
    ]
    
    Supervisor.init(children, strategy: :one_for_one)
  end
  
  defp metrics do
    [
      # Voice processing latency
      Metrics.summary("aybiza.voice.processing_latency",
        unit: {:native, :millisecond},
        tags: [:region, :model_type, :complexity]
      ),
      
      # Audio quality metrics
      Metrics.distribution("aybiza.audio.quality_score",
        tags: [:codec, :sample_rate, :processing_method]
      ),
      
      # Hybrid architecture performance
      Metrics.counter("aybiza.hybrid.edge_requests_total",
        tags: [:edge_region, :outcome]
      ),
      
      Metrics.summary("aybiza.hybrid.handoff_latency",
        unit: {:native, :millisecond},
        tags: [:source_region, :target_region]
      ),
      
      # Cost optimization
      Metrics.counter("aybiza.cost.optimization_applied_total",
        tags: [:strategy, :region, :model]
      ),
      
      # Claude model performance
      Metrics.histogram("aybiza.claude.response_time",
        unit: {:native, :millisecond},
        tags: [:model_version, :thinking_mode, :region],
        buckets: [10, 50, 100, 250, 500, 1000, 2500, 5000]
      )
    ]
  end
  
  # Emit custom telemetry events
  def emit_voice_processing_event(event_name, measurements, metadata \\ %{}) do
    :telemetry.execute(
      [:aybiza, :voice, event_name],
      measurements,
      Map.merge(metadata, %{timestamp: System.system_time(:microsecond)})
    )
  end
  
  def emit_hybrid_architecture_event(event_name, measurements, metadata \\ %{}) do
    :telemetry.execute(
      [:aybiza, :hybrid, event_name],
      measurements,
      Map.merge(metadata, %{
        timestamp: System.system_time(:microsecond),
        node: Node.self()
      })
    )
  end
end
```

## 6. Testing Strategies

### 6.1 Property-Based Testing for Voice Processing
```elixir
defmodule Aybiza.VoicePipeline.PropertyTest do
  use ExUnit.Case
  use PropCheck
  
  alias Aybiza.VoicePipeline.AudioProcessor
  
  property "audio processing maintains data integrity" do
    forall {sample_rate, bit_depth, audio_data} <- audio_generator() do
      case AudioProcessor.process_audio(audio_data, %{
        sample_rate: sample_rate,
        bit_depth: bit_depth
      }) do
        {:ok, processed_audio} ->
          # Verify audio properties are maintained
          original_duration = calculate_duration(audio_data, sample_rate)
          processed_duration = calculate_duration(processed_audio, sample_rate)
          
          # Duration should be preserved within 1ms tolerance
          abs(original_duration - processed_duration) <= 1
        
        {:error, _reason} ->
          # Acceptable for invalid input
          true
      end
    end
  end
  
  property "processing latency meets SLA requirements" do
    forall audio_chunk <- audio_chunk_generator() do
      start_time = System.monotonic_time(:millisecond)
      
      result = AudioProcessor.process_audio_chunk(audio_chunk, "test_stream")
      
      processing_time = System.monotonic_time(:millisecond) - start_time
      
      # Must complete within latency SLA
      case result do
        {:ok, _} -> processing_time <= 45  # 45ms for US/UK
        {:error, _} -> true  # Errors don't count against SLA
      end
    end
  end
  
  # Generators for property testing
  defp audio_generator do
    {choose(8000, 48000), oneof([8, 16, 24, 32]), binary()}
  end
  
  defp audio_chunk_generator do
    let size <- choose(512, 4096) do
      <<random_bytes::binary-size(size)>>
    end
  end
end
```

### 6.2 Integration Testing with Hybrid Architecture
```elixir
defmodule Aybiza.Integration.HybridArchitectureTest do
  use ExUnit.Case, async: false
  
  alias Aybiza.Test.Helpers.{MockCloudflare, MockAWS, VoiceTestHelper}
  
  setup do
    # Setup test environment with mocked services
    MockCloudflare.start_link()
    MockAWS.start_link()
    
    on_exit(fn ->
      MockCloudflare.stop()
      MockAWS.stop()
    end)
    
    :ok
  end
  
  describe "edge-to-cloud processing handoff" do
    test "seamless handoff for complex conversations" do
      # Simulate conversation that exceeds edge processing capabilities
      complex_conversation = VoiceTestHelper.generate_complex_conversation()
      
      # Start processing at edge
      {:ok, session_id} = Aybiza.Edge.ProcessingCoordinator.start_session(
        phone_number: "+1234567890",
        edge_region: "lax1"
      )
      
      # Process initial turns at edge
      initial_turns = Enum.take(complex_conversation, 3)
      
      for turn <- initial_turns do
        {:ok, response} = Aybiza.Edge.ProcessingCoordinator.process_turn(
          session_id,
          turn
        )
        
        assert response.processing_location == :edge
        assert response.latency_ms < 30  # Edge processing should be very fast
      end
      
      # Next turn should trigger handoff to cloud
      handoff_turn = Enum.at(complex_conversation, 3)
      
      {:ok, response} = Aybiza.Edge.ProcessingCoordinator.process_turn(
        session_id,
        handoff_turn
      )
      
      assert response.processing_location == :cloud
      assert response.handoff_triggered == true
      assert response.latency_ms < 100  # Still within SLA after handoff
      
      # Verify session state is properly transferred
      {:ok, session_state} = Aybiza.ConversationEngine.get_session_state(session_id)
      assert length(session_state.conversation_history) == 4
      assert session_state.processing_location == :cloud
    end
    
    test "cost optimization is applied appropriately" do
      # Test various scenarios for cost optimization
      scenarios = [
        %{complexity: :low, region: "us-east-1", expected_model: :claude_3_5_haiku},
        %{complexity: :medium, region: "eu-west-1", expected_model: :claude_3_5_sonnet},
        %{complexity: :high, region: "ap-southeast-1", expected_model: :claude_3_7_sonnet}
      ]
      
      for scenario <- scenarios do
        {:ok, session_id} = start_test_session(scenario.region)
        
        conversation = VoiceTestHelper.generate_conversation(scenario.complexity)
        
        {:ok, response} = Aybiza.ConversationEngine.process_conversation(
          session_id,
          conversation
        )
        
        assert response.model_used == scenario.expected_model
        assert response.cost_optimization_applied == true
        
        # Verify cost savings
        estimated_savings = calculate_cost_savings(response)
        assert estimated_savings > 0.15  # At least 15% savings
      end
    end
  end
  
  describe "performance under load" do
    @tag :load_testing
    test "maintains latency SLA under concurrent load" do
      # Simulate 100 concurrent voice sessions
      concurrent_sessions = 100
      
      tasks = 
        1..concurrent_sessions
        |> Enum.map(fn i ->
          Task.async(fn ->
            session_id = "load_test_#{i}"
            
            # Process a standard conversation
            conversation = VoiceTestHelper.generate_standard_conversation()
            
            start_time = System.monotonic_time(:millisecond)
            
            {:ok, response} = Aybiza.VoicePipeline.process_full_conversation(
              session_id,
              conversation
            )
            
            end_time = System.monotonic_time(:millisecond)
            total_time = end_time - start_time
            
            %{
              session_id: session_id,
              total_time: total_time,
              avg_turn_latency: response.avg_turn_latency,
              success: true
            }
          end)
        end)
      
      results = Task.await_many(tasks, 30_000)  # 30 second timeout
      
      # Analyze results
      successful_sessions = Enum.filter(results, & &1.success)
      assert length(successful_sessions) == concurrent_sessions
      
      avg_latencies = Enum.map(successful_sessions, & &1.avg_turn_latency)
      p95_latency = Enum.at(Enum.sort(avg_latencies), round(length(avg_latencies) * 0.95))
      
      # P95 latency should still meet SLA under load
      assert p95_latency <= 75  # Allow some degradation under load
    end
  end
  
  defp start_test_session(region) do
    Aybiza.SessionManager.start_session(%{
      phone_number: "+1234567890",
      region: region,
      test_mode: true
    })
  end
end
```

## 7. Security Best Practices

### 7.1 Input Validation and Sanitization
```elixir
defmodule Aybiza.Security.InputValidator do
  @moduledoc """
  Comprehensive input validation for voice processing pipeline.
  Prevents injection attacks and ensures data integrity.
  """
  
  # Phone number validation with international support
  @phone_regex ~r/^\+[1-9]\d{1,14}$/
  
  # Audio format validation
  @supported_audio_formats [:wav, :mp3, :ogg, :flac]
  @max_audio_size 10 * 1024 * 1024  # 10MB
  
  def validate_phone_number(phone_number) when is_binary(phone_number) do
    case Regex.match?(@phone_regex, phone_number) do
      true -> {:ok, phone_number}
      false -> {:error, :invalid_phone_number}
    end
  end
  
  def validate_audio_input(audio_data, metadata) do
    with :ok <- validate_audio_size(audio_data),
         :ok <- validate_audio_format(metadata),
         :ok <- validate_audio_content(audio_data) do
      {:ok, audio_data}
    end
  end
  
  def sanitize_conversation_input(input) when is_binary(input) do
    input
    |> String.trim()
    |> remove_control_characters()
    |> limit_length(10_000)  # Prevent memory exhaustion
    |> HtmlSanitizeEx.strip_tags()
  end
  
  def validate_api_key(api_key) when is_binary(api_key) do
    case String.length(api_key) do
      len when len >= 32 and len <= 128 ->
        case Base.decode64(api_key) do
          {:ok, _} -> {:ok, api_key}
          :error -> {:error, :invalid_api_key_format}
        end
      
      _ -> {:error, :invalid_api_key_length}
    end
  end
  
  # Private validation functions
  defp validate_audio_size(audio_data) do
    case byte_size(audio_data) do
      size when size <= @max_audio_size -> :ok
      _ -> {:error, :audio_too_large}
    end
  end
  
  defp validate_audio_format(%{format: format}) when format in @supported_audio_formats do
    :ok
  end
  
  defp validate_audio_format(_), do: {:error, :unsupported_audio_format}
  
  defp validate_audio_content(audio_data) do
    # Check for audio header signatures to prevent malicious uploads
    case audio_data do
      <<"RIFF", _::binary-size(4), "WAVE", _::binary>> -> :ok
      <<"ID3", _::binary>> -> :ok  # MP3
      <<"OggS", _::binary>> -> :ok  # OGG
      <<"fLaC", _::binary>> -> :ok  # FLAC
      _ -> {:error, :invalid_audio_content}
    end
  end
  
  defp remove_control_characters(input) do
    String.replace(input, ~r/[\x00-\x1F\x7F]/, "")
  end
  
  defp limit_length(input, max_length) do
    case String.length(input) do
      len when len <= max_length -> input
      _ -> String.slice(input, 0, max_length)
    end
  end
end
```

### 7.2 Encryption and Secrets Management
```elixir
defmodule Aybiza.Security.EncryptionManager do
  @moduledoc """
  Handles encryption for sensitive data in voice processing pipeline.
  Uses AWS KMS for key management and AES-256-GCM for data encryption.
  """
  
  alias Aybiza.AWS.KMSClient
  
  @algorithm :aes_256_gcm
  @key_spec "AES_256"
  
  def encrypt_sensitive_data(data, context \\ %{}) when is_binary(data) do
    with {:ok, data_key} <- KMSClient.generate_data_key(@key_spec, context),
         {:ok, encrypted_data} <- encrypt_with_key(data, data_key.plaintext),
         {:ok, encrypted_key} <- encrypt_data_key(data_key.plaintext) do
      
      # Zero out plaintext key from memory
      :crypto.strong_rand_bytes(byte_size(data_key.plaintext))
      
      {:ok, %{
        encrypted_data: encrypted_data,
        encrypted_key: encrypted_key,
        algorithm: @algorithm
      }}
    end
  end
  
  def decrypt_sensitive_data(%{encrypted_data: enc_data, encrypted_key: enc_key}) do
    with {:ok, plaintext_key} <- KMSClient.decrypt(enc_key),
         {:ok, decrypted_data} <- decrypt_with_key(enc_data, plaintext_key) do
      
      # Zero out key from memory
      :crypto.strong_rand_bytes(byte_size(plaintext_key))
      
      {:ok, decrypted_data}
    end
  end
  
  def encrypt_audio_stream(audio_stream, session_id) do
    # Stream encryption for real-time audio
    session_key = generate_session_key(session_id)
    
    Stream.map(audio_stream, fn chunk ->
      {:ok, encrypted_chunk} = encrypt_with_key(chunk, session_key)
      encrypted_chunk
    end)
  end
  
  # Private encryption functions
  defp encrypt_with_key(data, key) do
    iv = :crypto.strong_rand_bytes(12)  # 96-bit IV for GCM
    
    case :crypto.crypto_one_time_aead(@algorithm, key, iv, data, "", true) do
      {ciphertext, tag} -> 
        {:ok, %{
          iv: iv,
          ciphertext: ciphertext,
          tag: tag
        }}
      
      error -> {:error, error}
    end
  end
  
  defp decrypt_with_key(%{iv: iv, ciphertext: ciphertext, tag: tag}, key) do
    case :crypto.crypto_one_time_aead(@algorithm, key, iv, ciphertext, "", tag, false) do
      plaintext when is_binary(plaintext) -> {:ok, plaintext}
      error -> {:error, error}
    end
  end
  
  defp generate_session_key(session_id) do
    # Generate deterministic session key for audio stream
    :crypto.hash(:sha256, "aybiza_audio_key_#{session_id}")
  end
end
```

## 8. Configuration Management

### 8.1 Environment-Specific Configuration
```elixir
# config/config.exs
import Config

# Global configuration
config :aybiza,
  # Application metadata
  app_name: "AYBIZA Voice Platform",
  version: "1.0.0",
  
  # Hybrid architecture settings
  primary_regions: ["us-east-1", "us-west-2", "eu-west-1"],
  edge_regions: ["lax1", "dfw1", "lhr1", "fra1", "nrt1"],
  
  # Performance targets
  latency_targets: %{
    "us" => 50,      # 50ms for US
    "uk" => 50,      # 50ms for UK  
    "eu" => 75,      # 75ms for EU
    "global" => 100  # 100ms globally
  },
  
  # Cost optimization
  cost_optimization: %{
    enabled: true,
    target_savings: 0.286,  # 28.6% savings target
    model_selection_strategy: :adaptive
  }

# Database configuration
config :aybiza, Aybiza.Repo,
  # Use read replicas for analytics queries
  read_repo: Aybiza.ReadRepo,
  pool_size: 20,
  queue_target: 5_000,
  queue_interval: 5_000,
  
  # PostgreSQL 16.9+ specific settings
  parameters: [
    shared_preload_libraries: "timescaledb",
    max_connections: "200",
    shared_buffers: "256MB",
    effective_cache_size: "1GB",
    work_mem: "4MB",
    
    # Performance tuning for voice workloads
    random_page_cost: "1.1",
    effective_io_concurrency: "200",
    max_worker_processes: "8",
    max_parallel_workers_per_gather: "4"
  ]

# Phoenix configuration
config :aybiza, AybizaWeb.Endpoint,
  url: [host: "localhost"],
  render_errors: [view: AybizaWeb.ErrorView, accepts: ~w(html json)],
  pubsub_server: Aybiza.PubSub,
  
  # WebSocket configuration for real-time audio
  socket: [
    "/socket",
    AybizaWeb.UserSocket,
    websocket: [
      timeout: 45_000,
      compress: false,  # Disable compression for audio
      max_frame_size: 1_048_576  # 1MB frames for audio
    ]
  ]

# Telemetry configuration
config :aybiza, Aybiza.Telemetry,
  # Custom metrics collection
  voice_metrics_interval: 1_000,      # 1 second
  performance_metrics_interval: 5_000, # 5 seconds
  cost_metrics_interval: 60_000,      # 1 minute
  
  # Export to monitoring systems
  exporters: [
    {TelemetryMetricsPrometheus, [port: 9568]},
    {Aybiza.Telemetry.DatadogExporter, []}
  ]

# Environment-specific configurations
import_config "#{config_env()}.exs"
```

### 8.2 Production Configuration with Secrets
```elixir
# config/prod.exs
import Config

# Production-specific settings
config :aybiza,
  environment: :production,
  
  # Hybrid architecture production settings
  edge_processing_enabled: true,
  cost_optimization_aggressive: true,
  
  # Enhanced security in production
  security: %{
    require_ssl: true,
    hsts_max_age: 31_536_000,  # 1 year
    content_security_policy: true,
    rate_limiting: %{
      requests_per_minute: 300,
      burst_limit: 50
    }
  }

# Database production configuration
config :aybiza, Aybiza.Repo,
  # Connection from environment or AWS Secrets Manager
  url: {:system, "DATABASE_URL"},
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "20"),
  
  # Production-specific settings
  ssl: true,
  ssl_opts: [
    verify: :verify_peer,
    cacerts: :public_key.cacerts_get(),
    server_name_indication: 'aybiza-prod.cluster-xyz.region.rds.amazonaws.com',
    customize_hostname_check: [
      match_fun: :public_key.pkix_verify_hostname_match_fun(:https)
    ]
  ],
  
  # Connection pooling for high load
  queue_target: 2_000,
  queue_interval: 10_000

# Phoenix production configuration  
config :aybiza, AybizaWeb.Endpoint,
  # Load from environment variables
  http: [
    port: String.to_integer(System.get_env("PORT") || "4000"),
    transport_options: [socket_opts: [:inet6]]
  ],
  url: [
    host: System.get_env("APP_HOSTNAME") || "api.aybiza.ai",
    port: 443,
    scheme: "https"
  ],
  
  # SSL configuration
  https: [
    port: 443,
    cipher_suite: :strong,
    keyfile: System.get_env("SSL_KEY_PATH"),
    certfile: System.get_env("SSL_CERT_PATH"),
    transport_options: [socket_opts: [:inet6]]
  ],
  
  # Production optimizations
  cache_static_manifest: "priv/static/cache_manifest.json",
  server: true,
  root: ".",
  version: Application.spec(:aybiza, :vsn)

# AWS services configuration
config :aybiza, Aybiza.AWS,
  # Credentials from IAM roles or environment
  access_key_id: {:system, "AWS_ACCESS_KEY_ID"},
  secret_access_key: {:system, "AWS_SECRET_ACCESS_KEY"},
  region: System.get_env("AWS_DEFAULT_REGION") || "us-east-1",
  
  # Service-specific settings
  bedrock: %{
    model_region_mapping: %{
      "anthropic.claude-3-7-sonnet-20250219-v1:0" => ["us-east-1", "us-west-2"],
      "anthropic.claude-3-5-sonnet-20241022-v2:0" => ["us-east-1", "us-west-2", "eu-west-1"],
      "anthropic.claude-3-5-haiku-20241022-v1:0" => ["us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1"]
    },
    default_max_tokens: 4096,
    streaming_enabled: true
  },
  
  s3: %{
    bucket: System.get_env("S3_BUCKET") || "aybiza-prod-audio-storage",
    encryption: "AES256"
  }

# Logging configuration
config :logger,
  level: :info,
  backends: [
    :console,
    {LoggerFileBackend, :info_log},
    {LoggerFileBackend, :error_log}
  ]

config :logger, :console,
  format: "$time $metadata[$level] $message\n",
  metadata: [:request_id, :session_id, :user_id, :region]

config :logger, :info_log,
  path: "log/info.log",
  level: :info,
  format: "$time $metadata[$level] $message\n",
  metadata: [:request_id, :session_id, :user_id, :region]

config :logger, :error_log,
  path: "log/error.log", 
  level: :error,
  format: "$time $metadata[$level] $message\n",
  metadata: [:request_id, :session_id, :user_id, :region, :stacktrace]

# Runtime configuration from secrets
if System.get_env("RUNTIME_CONFIG") == "true" do
  config :aybiza, Aybiza.Repo,
    # Load database URL from AWS Secrets Manager
    url: {Aybiza.AWS.SecretsManager, :get_database_url, []},
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "20")
  
  config :aybiza, :twilio,
    account_sid: {Aybiza.AWS.SecretsManager, :get_secret, ["twilio_account_sid"]},
    auth_token: {Aybiza.AWS.SecretsManager, :get_secret, ["twilio_auth_token"]}
  
  config :aybiza, :deepgram,
    api_key: {Aybiza.AWS.SecretsManager, :get_secret, ["deepgram_api_key"]}
end
```

## 9. Deployment and DevOps Patterns

### 9.1 Release and Deployment Configuration
```elixir
# rel/env.sh.eex
#!/bin/sh

# Production environment configuration
export RELEASE_DISTRIBUTION=name
export RELEASE_NODE=<%= @release.name %>@${HOSTNAME}

# Memory and scheduler settings for voice processing
export ERL_MAX_PORTS=65536
export ERL_MAX_ETS_TABLES=32768

# Optimize for low-latency voice processing
export ERL_FLAGS="+MBas ageffcbf +MHas ageffcbf +MMscs 1024 +P 2097152 +Q 1048576"

# JIT compilation for performance
export ERL_FLAGS="${ERL_FLAGS} +JMsingle true"

# Memory allocator tuning for real-time performance
export ERL_FLAGS="${ERL_FLAGS} +MBsmbcs 512 +MHsmbcs 512 +MBlmbcs 1024 +MHlmbcs 1024"

# Scheduler configuration for multi-core systems
SCHEDULERS=$(nproc)
export ERL_FLAGS="${ERL_FLAGS} +S ${SCHEDULERS}:${SCHEDULERS}"

# Garbage collection tuning for low latency
export ERL_FLAGS="${ERL_FLAGS} +hms 8192 +hmbs 16384"

# Set application-specific environment variables
export POOL_SIZE=${POOL_SIZE:-20}
export PORT=${PORT:-4000}
export DATABASE_URL=${DATABASE_URL}
export SECRET_KEY_BASE=${SECRET_KEY_BASE}

# AWS configuration
export AWS_REGION=${AWS_REGION:-us-east-1}
export AWS_DEFAULT_REGION=${AWS_REGION}

# Voice processing configuration
export VOICE_PROCESSING_CONCURRENCY=${VOICE_PROCESSING_CONCURRENCY:-100}
export AUDIO_BUFFER_SIZE=${AUDIO_BUFFER_SIZE:-4096}

# Cloudflare edge configuration
export EDGE_PROCESSING_ENABLED=${EDGE_PROCESSING_ENABLED:-true}
export EDGE_REGIONS=${EDGE_REGIONS:-"lax1,dfw1,lhr1,fra1"}

# Health check configuration
export HEALTH_CHECK_INTERVAL=${HEALTH_CHECK_INTERVAL:-30}
export READINESS_CHECK_TIMEOUT=${READINESS_CHECK_TIMEOUT:-5}
```

## 10. Standardized Error Handling

### 10.1 Error Handling Patterns
```elixir
defmodule Aybiza.ErrorHandler do
  @moduledoc """
  Standardized error handling patterns for consistent error management
  """
  
  require Logger
  
  # Define standard error types
  @type error_reason :: :invalid_input | :not_found | :unauthorized | 
                        :rate_limited | :external_service_error | 
                        :timeout | :internal_error
  
  @type error_result :: {:error, error_reason()} | 
                        {:error, error_reason(), binary()} |
                        {:error, error_reason(), binary(), map()}
  
  @doc """
  Standardized error handling with consistent logging and formatting
  """
  def handle_error({:error, reason} = error, context \\ %{}) do
    log_error(reason, context)
    normalize_error(error)
  end
  
  def handle_error({:error, reason, message} = error, context \\ %{}) do
    log_error(reason, message, context)
    normalize_error(error)
  end
  
  def handle_error({:error, reason, message, details} = error, context \\ %{}) do
    log_error(reason, message, Map.merge(context, details))
    normalize_error(error)
  end
  
  @doc """
  Wrap functions with error handling
  """
  defmacro with_error_handling(context \\ %{}, do: block) do
    quote do
      try do
        unquote(block)
      rescue
        error in [ArgumentError, KeyError] ->
          Aybiza.ErrorHandler.handle_error(
            {:error, :invalid_input, Exception.message(error)},
            unquote(context)
          )
        
        error in [Ecto.NoResultsError] ->
          Aybiza.ErrorHandler.handle_error(
            {:error, :not_found, "Resource not found"},
            unquote(context)
          )
        
        error ->
          Aybiza.ErrorHandler.handle_error(
            {:error, :internal_error, Exception.message(error), %{stacktrace: __STACKTRACE__}},
            unquote(context)
          )
      catch
        :exit, reason ->
          Aybiza.ErrorHandler.handle_error(
            {:error, :internal_error, "Process exited", %{exit_reason: reason}},
            unquote(context)
          )
      end
    end
  end
  
  @doc """
  Chain operations with error handling
  """
  def chain(value, functions) do
    Enum.reduce_while(functions, {:ok, value}, fn fun, {:ok, acc} ->
      case fun.(acc) do
        {:ok, result} -> {:cont, {:ok, result}}
        {:error, _} = error -> {:halt, handle_error(error)}
      end
    end)
  end
  
  # Private functions
  defp log_error(reason, context) do
    Logger.error("Error occurred",
      reason: reason,
      context: sanitize_context(context)
    )
  end
  
  defp log_error(reason, message, context) do
    Logger.error(message,
      reason: reason,
      context: sanitize_context(context)
    )
  end
  
  defp normalize_error({:error, reason}) when is_atom(reason) do
    {:error, %{
      type: reason,
      message: humanize_error(reason)
    }}
  end
  
  defp normalize_error({:error, reason, message}) do
    {:error, %{
      type: reason,
      message: message
    }}
  end
  
  defp normalize_error({:error, reason, message, details}) do
    {:error, %{
      type: reason,
      message: message,
      details: sanitize_details(details)
    }}
  end
  
  defp humanize_error(:invalid_input), do: "Invalid input provided"
  defp humanize_error(:not_found), do: "Resource not found"
  defp humanize_error(:unauthorized), do: "Unauthorized access"
  defp humanize_error(:rate_limited), do: "Rate limit exceeded"
  defp humanize_error(:external_service_error), do: "External service error"
  defp humanize_error(:timeout), do: "Operation timed out"
  defp humanize_error(:internal_error), do: "Internal server error"
  defp humanize_error(_), do: "An error occurred"
  
  defp sanitize_context(context) do
    Map.drop(context, [:password, :token, :secret, :api_key])
  end
  
  defp sanitize_details(details) do
    details
    |> Map.drop([:password, :token, :secret, :api_key])
    |> Map.update(:stacktrace, nil, &format_stacktrace/1)
  end
  
  defp format_stacktrace(stacktrace) when is_list(stacktrace) do
    Enum.take(stacktrace, 5)
    |> Enum.map(&Exception.format_stacktrace_entry/1)
  end
  
  defp format_stacktrace(stacktrace), do: stacktrace
end
```

### 10.2 Error Handling in Practice
```elixir
defmodule Aybiza.VoiceProcessing do
  import Aybiza.ErrorHandler
  
  @doc """
  Example of using standardized error handling
  """
  def process_audio(audio_data, opts \\ %{}) do
    context = %{
      operation: :process_audio,
      audio_size: byte_size(audio_data),
      opts: opts
    }
    
    with_error_handling context do
      with {:ok, validated} <- validate_audio(audio_data),
           {:ok, processed} <- apply_processing(validated, opts),
           {:ok, result} <- save_result(processed) do
        {:ok, result}
      end
    end
  end
  
  @doc """
  Example with chain
  """
  def process_call_transcript(transcript) do
    context = %{operation: :process_transcript, length: String.length(transcript)}
    
    ErrorHandler.chain(transcript, [
      &validate_transcript/1,
      &detect_language/1,
      &extract_entities/1,
      &analyze_sentiment/1,
      &save_analysis/1
    ])
    |> case do
      {:ok, result} -> {:ok, result}
      error -> ErrorHandler.handle_error(error, context)
    end
  end
  
  # Consistent error returns
  defp validate_audio(data) when byte_size(data) == 0 do
    {:error, :invalid_input, "Audio data cannot be empty"}
  end
  
  defp validate_audio(data) when byte_size(data) > 10_485_760 do
    {:error, :invalid_input, "Audio data exceeds 10MB limit"}
  end
  
  defp validate_audio(data), do: {:ok, data}
end
```

### 10.3 Controller Error Handling
```elixir
defmodule AybizaWeb.ErrorHelpers do
  @moduledoc """
  Standardized error responses for controllers
  """
  
  import Plug.Conn
  import Phoenix.Controller
  
  def handle_errors(conn, {:error, %{type: type, message: message} = error}) do
    conn
    |> put_status(error_to_status(type))
    |> put_view(AybizaWeb.ErrorView)
    |> render("error.json", error: error)
  end
  
  def handle_errors(conn, {:error, reason}) when is_atom(reason) do
    handle_errors(conn, {:error, %{type: reason, message: humanize_error(reason)}})
  end
  
  defp error_to_status(:invalid_input), do: :bad_request
  defp error_to_status(:not_found), do: :not_found
  defp error_to_status(:unauthorized), do: :unauthorized
  defp error_to_status(:rate_limited), do: :too_many_requests
  defp error_to_status(:timeout), do: :request_timeout
  defp error_to_status(:external_service_error), do: :bad_gateway
  defp error_to_status(_), do: :internal_server_error
  
  defp humanize_error(error) do
    error
    |> Atom.to_string()
    |> String.replace("_", " ")
    |> String.capitalize()
  end
end

# Usage in controller
defmodule AybizaWeb.AgentController do
  use AybizaWeb, :controller
  import AybizaWeb.ErrorHelpers
  
  def show(conn, %{"id" => id}) do
    case Aybiza.Agents.get_agent(id) do
      {:ok, agent} ->
        render(conn, "show.json", agent: agent)
      
      error ->
        handle_errors(conn, error)
    end
  end
end
```

### 9.2 Hot Code Deployment Patterns
```elixir
defmodule Aybiza.Deployment.HotUpgrade do
  @moduledoc """
  Handles hot code upgrades for zero-downtime deployments.
  Critical for voice processing systems that cannot tolerate interruptions.
  """
  
  require Logger
  
  @doc """
  Performs a hot upgrade with proper session migration.
  """
  def perform_hot_upgrade(release_version) do
    Logger.info("Starting hot upgrade to version #{release_version}")
    
    with :ok <- validate_upgrade_safety(),
         :ok <- prepare_session_migration(),
         :ok <- perform_code_upgrade(release_version),
         :ok <- migrate_active_sessions(),
         :ok <- validate_upgrade_success() do
      
      Logger.info("Hot upgrade completed successfully")
      cleanup_old_code()
      {:ok, :upgrade_completed}
    else
      {:error, reason} ->
        Logger.error("Hot upgrade failed: #{inspect(reason)}")
        rollback_upgrade()
        {:error, reason}
    end
  end
  
  @doc """
  Validates that a hot upgrade can be performed safely.
  """
  def validate_upgrade_safety do
    checks = [
      &check_active_call_count/0,
      &check_system_health/0,
      &check_resource_utilization/0,
      &check_upgrade_compatibility/0
    ]
    
    case Enum.find_value(checks, &run_safety_check/1) do
      nil -> :ok
      error -> {:error, error}
    end
  end
  
  defp check_active_call_count do
    active_calls = Aybiza.CallManager.get_active_call_count()
    
    case active_calls do
      count when count > 1000 ->
        {:error, "Too many active calls (#{count}) for safe upgrade"}
      
      _ -> nil
    end
  end
  
  defp check_system_health do
    health_status = Aybiza.HealthCheck.system_status()
    
    case health_status.overall_status do
      :healthy -> nil
      status -> {:error, "System unhealthy: #{status}"}
    end
  end
  
  defp check_resource_utilization do
    %{cpu: cpu_usage, memory: memory_usage} = :observer_backend.sys_info()
    
    cond do
      cpu_usage > 80 -> {:error, "CPU usage too high: #{cpu_usage}%"}
      memory_usage > 85 -> {:error, "Memory usage too high: #{memory_usage}%"}
      true -> nil
    end
  end
  
  defp prepare_session_migration do
    # Prepare active voice sessions for migration
    active_sessions = Aybiza.SessionManager.list_active_sessions()
    
    Logger.info("Preparing #{length(active_sessions)} sessions for migration")
    
    Enum.each(active_sessions, fn session ->
      Aybiza.SessionManager.mark_for_migration(session.id)
    end)
    
    :ok
  end
  
  defp perform_code_upgrade(release_version) do
    case :release_handler.upgrade_app(:aybiza, release_version) do
      :ok -> 
        Logger.info("Code upgrade successful")
        :ok
      
      {:error, reason} ->
        Logger.error("Code upgrade failed: #{inspect(reason)}")
        {:error, reason}
    end
  end
  
  defp migrate_active_sessions do
    # Gradually migrate sessions to new code
    sessions_to_migrate = Aybiza.SessionManager.get_sessions_marked_for_migration()
    
    Logger.info("Migrating #{length(sessions_to_migrate)} active sessions")
    
    # Migrate in batches to avoid overwhelming the system
    sessions_to_migrate
    |> Enum.chunk_every(10)
    |> Enum.each(fn batch ->
      migrate_session_batch(batch)
      # Brief pause between batches
      Process.sleep(100)
    end)
    
    :ok
  end
  
  defp migrate_session_batch(sessions) do
    tasks = Enum.map(sessions, fn session ->
      Task.async(fn ->
        case Aybiza.SessionManager.migrate_session(session.id) do
          :ok -> {:ok, session.id}
          error -> {:error, session.id, error}
        end
      end)
    end)
    
    results = Task.await_many(tasks, 5000)
    
    # Log any migration failures
    Enum.each(results, fn
      {:ok, session_id} ->
        Logger.debug("Successfully migrated session #{session_id}")
      
      {:error, session_id, reason} ->
        Logger.warn("Failed to migrate session #{session_id}: #{inspect(reason)}")
    end)
  end
end
```

## 10. Monitoring and Observability

### 10.1 Custom Metrics and Health Checks
```elixir
defmodule Aybiza.HealthCheck do
  use GenServer
  require Logger
  
  @check_interval 30_000  # 30 seconds
  @unhealthy_threshold 3  # 3 consecutive failures
  
  defstruct [
    :voice_pipeline_status,
    :database_status,
    :aws_services_status,
    :edge_connectivity_status,
    :overall_status,
    :last_check_time,
    :consecutive_failures
  ]
  
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def system_status do
    GenServer.call(__MODULE__, :get_status)
  end
  
  def force_health_check do
    GenServer.cast(__MODULE__, :force_check)
  end
  
  @impl true
  def init(_opts) do
    schedule_health_check()
    
    initial_state = %__MODULE__{
      voice_pipeline_status: :unknown,
      database_status: :unknown,
      aws_services_status: :unknown,
      edge_connectivity_status: :unknown,
      overall_status: :unknown,
      last_check_time: DateTime.utc_now(),
      consecutive_failures: 0
    }
    
    {:ok, initial_state}
  end
  
  @impl true
  def handle_call(:get_status, _from, state) do
    {:reply, state, state}
  end
  
  @impl true
  def handle_cast(:force_check, state) do
    new_state = perform_health_checks(state)
    {:noreply, new_state}
  end
  
  @impl true
  def handle_info(:health_check, state) do
    new_state = perform_health_checks(state)
    schedule_health_check()
    {:noreply, new_state}
  end
  
  defp perform_health_checks(state) do
    Logger.debug("Performing health checks")
    start_time = System.monotonic_time(:millisecond)
    
    # Perform all health checks concurrently
    checks = [
      Task.async(&check_voice_pipeline/0),
      Task.async(&check_database_connectivity/0),
      Task.async(&check_aws_services/0),
      Task.async(&check_edge_connectivity/0)
    ]
    
    results = Task.await_many(checks, 10_000)  # 10 second timeout
    
    [voice_status, db_status, aws_status, edge_status] = results
    
    overall_status = determine_overall_status([voice_status, db_status, aws_status, edge_status])
    
    consecutive_failures = case overall_status do
      :healthy -> 0
      _ -> state.consecutive_failures + 1
    end
    
    new_state = %{state |
      voice_pipeline_status: voice_status,
      database_status: db_status,
      aws_services_status: aws_status,
      edge_connectivity_status: edge_status,
      overall_status: overall_status,
      last_check_time: DateTime.utc_now(),
      consecutive_failures: consecutive_failures
    }
    
    check_time = System.monotonic_time(:millisecond) - start_time
    
    # Emit telemetry
    :telemetry.execute([:aybiza, :health_check], %{
      duration_ms: check_time,
      consecutive_failures: consecutive_failures
    }, %{
      overall_status: overall_status,
      voice_status: voice_status,
      database_status: db_status,
      aws_status: aws_status,
      edge_status: edge_status
    })
    
    # Alert if system is consistently unhealthy
    if consecutive_failures >= @unhealthy_threshold do
      Logger.error("System unhealthy for #{consecutive_failures} consecutive checks")
      send_alert(:system_unhealthy, new_state)
    end
    
    new_state
  end
  
  defp check_voice_pipeline do
    try do
      # Test audio processing pipeline
      test_audio = generate_test_audio()
      
      case Aybiza.VoicePipeline.AudioProcessor.process_audio_chunk(test_audio, "health_check") do
        {:ok, _processed} -> :healthy
        {:error, _reason} -> :unhealthy
      end
    rescue
      _ -> :unhealthy
    end
  end
  
  defp check_database_connectivity do
    try do
      case Aybiza.Repo.query("SELECT 1", [], timeout: 5_000) do
        {:ok, _result} -> :healthy
        {:error, _reason} -> :unhealthy
      end
    rescue
      _ -> :unhealthy
    end
  end
  
  defp check_aws_services do
    # Check critical AWS services
    aws_checks = [
      &check_bedrock_connectivity/0,
      &check_s3_connectivity/0,
      &check_kms_connectivity/0
    ]
    
    results = Enum.map(aws_checks, & &1.())
    
    case Enum.all?(results, &(&1 == :healthy)) do
      true -> :healthy
      false -> :degraded
    end
  end
  
  defp check_edge_connectivity do
    # Test connectivity to Cloudflare edge locations
    edge_regions = Application.get_env(:aybiza, :edge_regions, [])
    
    healthy_regions = 
      edge_regions
      |> Enum.map(&test_edge_region/1)
      |> Enum.count(&(&1 == :healthy))
    
    case healthy_regions / length(edge_regions) do
      ratio when ratio >= 0.8 -> :healthy    # 80% of regions healthy
      ratio when ratio >= 0.5 -> :degraded   # 50% of regions healthy
      _ -> :unhealthy
    end
  end
  
  defp determine_overall_status(statuses) do
    cond do
      Enum.all?(statuses, &(&1 == :healthy)) -> :healthy
      Enum.any?(statuses, &(&1 == :unhealthy)) -> :unhealthy
      true -> :degraded
    end
  end
  
  defp schedule_health_check do
    Process.send_after(self(), :health_check, @check_interval)
  end
end
```

This comprehensive guide provides the foundation for building and maintaining the AYBIZA platform with Elixir/OTP best practices, optimized for our hybrid Cloudflare+AWS architecture and ultra-low latency voice processing requirements.

Remember to:
1. Always follow these patterns for consistency across the codebase
2. Adapt patterns to specific use cases while maintaining core principles
3. Continuously monitor and optimize based on real-world performance data
4. Keep security and compliance requirements at the forefront of all development decisions
5. Test thoroughly, especially the real-time voice processing components
6. Document any deviations from these patterns with clear justifications

The patterns in this guide are specifically designed for the AYBIZA platform's unique requirements of ultra-low latency voice processing, hybrid edge-cloud architecture, and enterprise-grade reliability.