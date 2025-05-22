# Twilio Integration Enhancements

This document describes enhanced Twilio integration features to improve monitoring, control, and metadata handling in the AYBIZA platform's hybrid Cloudflare+AWS architecture, optimized for ultra-low latency (<50ms US/UK) and seamless integration with Claude 3.7 Sonnet's extended thinking capabilities.

## 1. TwiML Generation with Track Selection

Enhance the TwiML generator for hybrid architecture deployment, with explicit track selection for better control over audio streaming and edge-optimized routing:

```elixir
defmodule Aybiza.VoiceGateway.TwiMLGenerator do
  @doc """
  Generates TwiML for streaming with explicit track selection
  """
  def generate_stream_twiml(websocket_url, options \\ %{}) do
    track = Map.get(options, :track, "both_tracks")
    stream_name = Map.get(options, :stream_name, generate_stream_name())
    custom_params = Map.get(options, :custom_params, %{})
    
    {:safe, twiml} = TwiML.response() do
      TwiML.Start do
        TwiML.Stream(url: websocket_url, 
                     track: track,
                     name: stream_name) do
          # Add custom parameters
          Enum.each(custom_params, fn {key, value} ->
            TwiML.Parameter(name: key, value: value)
          end)
        end
      end
    end
    
    twiml
  end
  
  @doc """
  Generates TwiML for bidirectional streaming
  """
  def generate_bidirectional_stream_twiml(websocket_url, options \\ %{}) do
    stream_name = Map.get(options, :stream_name, generate_stream_name())
    custom_params = Map.get(options, :custom_params, %{})
    status_callback = Map.get(options, :status_callback)
    
    {:safe, twiml} = TwiML.response() do
      TwiML.Connect do
        TwiML.Stream(
          url: websocket_url,
          name: stream_name,
          statusCallback: status_callback
        ) do
          # Add custom parameters
          Enum.each(custom_params, fn {key, value} ->
            TwiML.Parameter(name: key, value: value)
          end)
        end
      end
    end
    
    twiml
  end
  
  defp generate_stream_name do
    "stream_#{System.unique_integer([:positive])}"
  end
end
```

## 2. Stream Status Callbacks Implementation

Implement comprehensive status callback handling for better monitoring and debugging:

```elixir
defmodule Aybiza.VoiceGateway.StreamStatusHandler do
  require Logger
  
  @doc """
  Handles stream status callbacks from Twilio
  """
  def handle_status_callback(params) do
    %{
      "StreamSid" => stream_sid,
      "StreamStatus" => status,
      "StreamName" => stream_name,
      "CallSid" => call_sid,
      "AccountSid" => account_sid,
      "Timestamp" => timestamp
    } = params
    
    # Log the status change
    Logger.info("Stream status update", %{
      stream_sid: stream_sid,
      status: status,
      stream_name: stream_name,
      call_sid: call_sid
    })
    
    # Handle specific status changes
    case status do
      "connected" ->
        handle_stream_connected(params)
      
      "disconnected" ->
        handle_stream_disconnected(params)
      
      "started" ->
        handle_stream_started(params)
      
      "stopped" ->
        handle_stream_stopped(params)
      
      "failed" ->
        handle_stream_failed(params)
      
      _ ->
        Logger.warning("Unknown stream status", %{status: status})
    end
    
    # Record metrics
    record_stream_metrics(status, params)
    
    :ok
  end
  
  defp handle_stream_connected(params) do
    # Update call state to streaming
    Aybiza.VoicePipeline.CallManager.update_stream_status(
      params["CallSid"],
      :connected,
      params
    )
  end
  
  defp handle_stream_disconnected(params) do
    # Handle graceful disconnection
    Aybiza.VoicePipeline.CallManager.handle_stream_disconnect(
      params["CallSid"],
      params["Reason"]
    )
  end
  
  defp handle_stream_failed(params) do
    # Handle stream failure
    Logger.error("Stream failed", %{
      call_sid: params["CallSid"],
      error_code: params["ErrorCode"],
      error_message: params["ErrorMessage"]
    })
    
    # Trigger error recovery
    Aybiza.VoicePipeline.ErrorRecovery.handle_stream_failure(
      params["CallSid"],
      params
    )
  end
  
  defp record_stream_metrics(status, params) do
    :telemetry.execute(
      [:aybiza, :twilio, :stream, status],
      %{count: 1},
      %{
        call_sid: params["CallSid"],
        stream_name: params["StreamName"]
      }
    )
  end
end
```

## 3. Custom Parameters for Metadata

Utilize custom parameters to pass metadata through the WebSocket connection:

```elixir
defmodule Aybiza.VoiceGateway.StreamMetadata do
  @doc """
  Prepares custom parameters for stream metadata
  """
  def prepare_stream_metadata(call_id, agent_id, tenant_id, options \\ %{}) do
    %{
      # Core identifiers
      call_id: call_id,
      agent_id: agent_id,
      tenant_id: tenant_id,
      
      # Session information
      session_id: generate_session_id(),
      timestamp: DateTime.utc_now() |> DateTime.to_iso8601(),
      
      # Agent configuration
      agent_name: get_agent_name(agent_id),
      agent_version: get_agent_version(agent_id),
      
      # Call context
      caller_region: Map.get(options, :caller_region),
      call_type: Map.get(options, :call_type, "inbound"),
      priority: Map.get(options, :priority, "normal"),
      
      # Feature flags
      features: %{
        transcription_enabled: true,
        recording_enabled: Map.get(options, :recording_enabled, false),
        analytics_enabled: true
      }
    }
  end
  
  @doc """
  Extracts metadata from WebSocket connection parameters
  """
  def extract_stream_metadata(params) do
    %{
      call_id: params["call_id"],
      agent_id: params["agent_id"],
      tenant_id: params["tenant_id"],
      session_id: params["session_id"],
      timestamp: params["timestamp"],
      features: parse_features(params["features"])
    }
  end
  
  defp generate_session_id do
    "session_#{System.unique_integer([:positive])}_#{System.os_time(:microsecond)}"
  end
  
  defp get_agent_name(agent_id) do
    # Fetch from agent configuration
    case Aybiza.AgentManager.get_agent(agent_id) do
      {:ok, agent} -> agent.name
      _ -> "unknown"
    end
  end
  
  defp get_agent_version(agent_id) do
    # Fetch agent version
    case Aybiza.AgentManager.get_agent(agent_id) do
      {:ok, agent} -> agent.version
      _ -> "1.0.0"
    end
  end
  
  defp parse_features(features) when is_map(features), do: features
  defp parse_features(features) when is_binary(features) do
    case Jason.decode(features) do
      {:ok, parsed} -> parsed
      _ -> %{}
    end
  end
  defp parse_features(_), do: %{}
end
```

## 4. Enhanced Keep-Alive Patterns

Implement robust keep-alive mechanisms for long-running connections:

```elixir
defmodule Aybiza.VoiceGateway.KeepAlive do
  use GenServer
  require Logger
  
  @keepalive_interval 30_000  # 30 seconds
  @timeout_threshold 90_000   # 90 seconds
  
  defmodule State do
    defstruct [
      :connection_id,
      :socket,
      :last_activity,
      :keepalive_timer,
      :timeout_timer,
      :retry_count,
      :max_retries
    ]
  end
  
  def start_link(connection_id, socket, opts \\ []) do
    GenServer.start_link(__MODULE__, 
      {connection_id, socket, opts}, 
      name: via_tuple(connection_id))
  end
  
  @impl true
  def init({connection_id, socket, opts}) do
    state = %State{
      connection_id: connection_id,
      socket: socket,
      last_activity: System.monotonic_time(:millisecond),
      retry_count: 0,
      max_retries: Keyword.get(opts, :max_retries, 3)
    }
    
    # Schedule initial keepalive
    state = schedule_keepalive(state)
    state = schedule_timeout_check(state)
    
    {:ok, state}
  end
  
  # Update activity timestamp when receiving data
  def update_activity(connection_id) do
    GenServer.cast(via_tuple(connection_id), :update_activity)
  end
  
  @impl true
  def handle_cast(:update_activity, state) do
    {:noreply, %{state | last_activity: System.monotonic_time(:millisecond)}}
  end
  
  @impl true
  def handle_info(:send_keepalive, state) do
    # Send keepalive frame
    case send_keepalive_frame(state.socket) do
      :ok ->
        Logger.debug("Keepalive sent", %{connection_id: state.connection_id})
        {:noreply, schedule_keepalive(state)}
      
      {:error, reason} ->
        Logger.warning("Keepalive failed", %{
          connection_id: state.connection_id,
          reason: reason,
          retry_count: state.retry_count
        })
        
        if state.retry_count < state.max_retries do
          # Retry with backoff
          Process.send_after(self(), :retry_keepalive, 5_000)
          {:noreply, %{state | retry_count: state.retry_count + 1}}
        else
          # Max retries reached, disconnect
          handle_keepalive_failure(state)
          {:stop, :normal, state}
        end
    end
  end
  
  @impl true
  def handle_info(:retry_keepalive, state) do
    send(self(), :send_keepalive)
    {:noreply, state}
  end
  
  @impl true
  def handle_info(:check_timeout, state) do
    time_since_activity = System.monotonic_time(:millisecond) - state.last_activity
    
    if time_since_activity > @timeout_threshold do
      Logger.warning("Connection timeout", %{
        connection_id: state.connection_id,
        time_since_activity: time_since_activity
      })
      
      handle_connection_timeout(state)
      {:stop, :normal, state}
    else
      {:noreply, schedule_timeout_check(state)}
    end
  end
  
  defp schedule_keepalive(state) do
    if state.keepalive_timer do
      Process.cancel_timer(state.keepalive_timer)
    end
    
    timer = Process.send_after(self(), :send_keepalive, @keepalive_interval)
    %{state | keepalive_timer: timer, retry_count: 0}
  end
  
  defp schedule_timeout_check(state) do
    if state.timeout_timer do
      Process.cancel_timer(state.timeout_timer)
    end
    
    timer = Process.send_after(self(), :check_timeout, @keepalive_interval)
    %{state | timeout_timer: timer}
  end
  
  defp send_keepalive_frame(socket) do
    # Send WebSocket ping frame
    case socket.send(socket, :ping) do
      :ok -> :ok
      error -> {:error, error}
    end
  end
  
  defp handle_keepalive_failure(state) do
    # Notify call manager of connection failure
    Aybiza.VoicePipeline.CallManager.handle_connection_failure(
      state.connection_id,
      :keepalive_failure
    )
  end
  
  defp handle_connection_timeout(state) do
    # Notify call manager of timeout
    Aybiza.VoicePipeline.CallManager.handle_connection_timeout(
      state.connection_id
    )
  end
  
  defp via_tuple(connection_id) do
    {:via, Registry, {Aybiza.ConnectionRegistry, connection_id}}
  end
end
```

## 5. Stream Naming and Control

Implement stream naming for better control and management:

```elixir
defmodule Aybiza.VoiceGateway.StreamControl do
  @moduledoc """
  Manages named streams for better control and monitoring
  """
  
  # Stream name format: "{prefix}_{call_sid}_{timestamp}"
  @stream_prefix "aybiza"
  
  @doc """
  Generates a unique stream name for a call
  """
  def generate_stream_name(call_sid, options \\ %{}) do
    prefix = Map.get(options, :prefix, @stream_prefix)
    timestamp = System.os_time(:microsecond)
    
    "#{prefix}_#{call_sid}_#{timestamp}"
  end
  
  @doc """
  Stops a named stream
  """
  def stop_stream(stream_name) do
    {:safe, twiml} = TwiML.response() do
      TwiML.Stop do
        TwiML.Stream(name: stream_name)
      end
    end
    
    twiml
  end
  
  @doc """
  Updates stream parameters
  """
  def update_stream(stream_name, params) do
    {:safe, twiml} = TwiML.response() do
      TwiML.Update do
        TwiML.Stream(name: stream_name) do
          Enum.each(params, fn {key, value} ->
            TwiML.Parameter(name: key, value: value)
          end)
        end
      end
    end
    
    twiml
  end
  
  @doc """
  Manages stream lifecycle
  """
  def manage_stream_lifecycle(call_sid, action, options \\ %{}) do
    stream_name = get_or_create_stream_name(call_sid)
    
    case action do
      :start ->
        start_named_stream(call_sid, stream_name, options)
      
      :stop ->
        stop_stream(stream_name)
      
      :pause ->
        pause_stream(stream_name)
      
      :resume ->
        resume_stream(stream_name)
      
      :update ->
        update_stream(stream_name, Map.get(options, :params, %{}))
    end
  end
  
  defp start_named_stream(call_sid, stream_name, options) do
    websocket_url = Map.get(options, :websocket_url)
    track = Map.get(options, :track, "both_tracks")
    custom_params = prepare_custom_params(call_sid, options)
    
    {:safe, twiml} = TwiML.response() do
      TwiML.Start do
        TwiML.Stream(
          url: websocket_url,
          name: stream_name,
          track: track
        ) do
          Enum.each(custom_params, fn {key, value} ->
            TwiML.Parameter(name: key, value: value)
          end)
        end
      end
    end
    
    # Store stream name mapping
    store_stream_mapping(call_sid, stream_name)
    
    twiml
  end
  
  defp get_or_create_stream_name(call_sid) do
    case get_stream_name(call_sid) do
      {:ok, stream_name} -> stream_name
      :error -> generate_stream_name(call_sid)
    end
  end
  
  defp store_stream_mapping(call_sid, stream_name) do
    :ets.insert(:stream_names, {call_sid, stream_name})
  end
  
  defp get_stream_name(call_sid) do
    case :ets.lookup(:stream_names, call_sid) do
      [{^call_sid, stream_name}] -> {:ok, stream_name}
      [] -> :error
    end
  end
  
  defp prepare_custom_params(call_sid, options) do
    Aybiza.VoiceGateway.StreamMetadata.prepare_stream_metadata(
      call_sid,
      Map.get(options, :agent_id),
      Map.get(options, :tenant_id),
      options
    )
  end
end
```

## Integration with Existing Pipeline

Update the existing TwilioController to use these enhancements:

```elixir
defmodule AybizaWeb.TwilioController do
  use AybizaWeb, :controller
  
  alias Aybiza.VoiceGateway.{TwiMLGenerator, StreamControl, StreamMetadata}
  
  def voice_webhook(conn, params) do
    call_sid = params["CallSid"]
    
    # Prepare stream with all enhancements
    twiml = case params["CallStatus"] do
      "ringing" ->
        handle_incoming_call(call_sid, params)
      
      "completed" ->
        handle_call_completed(call_sid, params)
      
      _ ->
        handle_call_status_update(call_sid, params)
    end
    
    conn
    |> put_resp_content_type("text/xml")
    |> send_resp(200, twiml)
  end
  
  defp handle_incoming_call(call_sid, params) do
    # Get agent and tenant information
    {:ok, agent_id} = resolve_agent_for_call(params)
    {:ok, tenant_id} = resolve_tenant_for_call(params)
    
    # Generate WebSocket URL
    websocket_url = generate_websocket_url()
    
    # Generate TwiML with all enhancements
    TwiMLGenerator.generate_stream_twiml(websocket_url, %{
      track: "both_tracks",
      stream_name: StreamControl.generate_stream_name(call_sid),
      custom_params: StreamMetadata.prepare_stream_metadata(
        call_sid, 
        agent_id, 
        tenant_id,
        %{
          caller_region: params["CallerCountry"],
          caller_number: params["From"],
          called_number: params["To"]
        }
      ),
      status_callback: Routes.twilio_path(@endpoint, :stream_status_callback)
    })
  end
  
  def stream_status_callback(conn, params) do
    # Handle stream status updates
    Aybiza.VoiceGateway.StreamStatusHandler.handle_status_callback(params)
    
    conn
    |> put_resp_content_type("text/xml")
    |> send_resp(200, "")
  end
  
  defp generate_websocket_url do
    # Generate secure WebSocket URL for this environment
    scheme = if Mix.env() == :prod, do: "wss", else: "ws"
    host = Application.get_env(:aybiza, :websocket_host)
    port = Application.get_env(:aybiza, :websocket_port)
    
    "#{scheme}://#{host}:#{port}/socket/websocket"
  end
end
```

## Configuration

Add these configuration options to your config files:

```elixir
# config/config.exs
config :aybiza, :twilio,
  stream_status_callback_path: "/api/twilio/stream-status",
  default_track: "both_tracks",
  enable_stream_naming: true,
  enable_custom_parameters: true,
  enable_status_callbacks: true,
  keepalive_interval: 30_000,
  connection_timeout: 90_000,
  max_retry_attempts: 3

# config/runtime.exs
config :aybiza, :websocket,
  host: System.get_env("WEBSOCKET_HOST", "localhost"),
  port: String.to_integer(System.get_env("WEBSOCKET_PORT", "4000")),
  secure: System.get_env("WEBSOCKET_SECURE", "false") == "true"
```

## Testing

Add tests for the new functionality:

```elixir
defmodule Aybiza.VoiceGateway.TwiMLGeneratorTest do
  use ExUnit.Case, async: true
  
  alias Aybiza.VoiceGateway.TwiMLGenerator
  
  test "generates TwiML with track selection" do
    twiml = TwiMLGenerator.generate_stream_twiml("wss://example.com", %{
      track: "inbound_track"
    })
    
    assert twiml =~ ~r/track="inbound_track"/
  end
  
  test "generates TwiML with stream name" do
    twiml = TwiMLGenerator.generate_stream_twiml("wss://example.com", %{
      stream_name: "test_stream_123"
    })
    
    assert twiml =~ ~r/name="test_stream_123"/
  end
  
  test "includes custom parameters" do
    twiml = TwiMLGenerator.generate_stream_twiml("wss://example.com", %{
      custom_params: %{agent_id: "agent_123", tenant_id: "tenant_456"}
    })
    
    assert twiml =~ ~r/<Parameter name="agent_id" value="agent_123"/
    assert twiml =~ ~r/<Parameter name="tenant_id" value="tenant_456"/
  end
end
```

## Monitoring and Metrics

Add monitoring for the new features:

```elixir
defmodule Aybiza.VoiceGateway.Metrics do
  def setup_metrics do
    # Stream lifecycle metrics
    :telemetry.attach_many(
      "twilio-stream-metrics",
      [
        [:aybiza, :twilio, :stream, :connected],
        [:aybiza, :twilio, :stream, :disconnected],
        [:aybiza, :twilio, :stream, :failed]
      ],
      &handle_stream_event/4,
      nil
    )
    
    # Keep-alive metrics
    :telemetry.attach_many(
      "keepalive-metrics",
      [
        [:aybiza, :twilio, :keepalive, :sent],
        [:aybiza, :twilio, :keepalive, :failed],
        [:aybiza, :twilio, :keepalive, :timeout]
      ],
      &handle_keepalive_event/4,
      nil
    )
  end
  
  defp handle_stream_event([:aybiza, :twilio, :stream, status], measurements, metadata, _config) do
    # Record stream status metrics
    Prometheus.Counter.inc([
      name: :twilio_stream_events_total,
      labels: [status, metadata.tenant_id]
    ])
  end
  
  defp handle_keepalive_event([:aybiza, :twilio, :keepalive, status], measurements, metadata, _config) do
    # Record keepalive metrics
    Prometheus.Counter.inc([
      name: :twilio_keepalive_events_total,
      labels: [status, metadata.connection_id]
    ])
  end
end
```

## Summary

These enhancements provide:

1. **Better Control**: Explicit track selection and stream naming
2. **Improved Monitoring**: Stream status callbacks and lifecycle tracking
3. **Rich Metadata**: Custom parameters for context passing
4. **Connection Stability**: Enhanced keep-alive patterns
5. **Operational Excellence**: Better debugging and troubleshooting capabilities

All enhancements maintain backward compatibility while adding powerful new features for production deployments.