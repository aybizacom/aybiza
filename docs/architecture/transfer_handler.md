# AYBIZA Call Transfer Handler

## Overview

The Transfer Handler is a critical component of the AYBIZA AI Voice Business Agent platform that manages the transfer of calls between AI agents and human operators. This document outlines the architecture and implementation of the Transfer Handler.

## Architecture

The Transfer Handler is designed with the following key capabilities:

1. **Smart Routing**: Intelligently routes calls to the appropriate destination based on context
2. **Context Preservation**: Maintains conversation context during transfers
3. **Queue Management**: Manages transfer queues with priorities and estimated wait times
4. **Warm Transfers**: Provides context to receiving agents or operators before connecting
5. **Agent-to-Agent Transfer**: Seamlessly transfers between specialized AI agents
6. **Human Escalation**: Gracefully transfers to human operators when needed
7. **Transfer Analytics**: Tracks and analyzes transfer patterns and outcomes

## Implementation

### Transfer Handler Core

```elixir
defmodule Aybiza.TransferHandler do
  @moduledoc """
  Manages call transfers between AI agents and human operators
  """
  
  use GenServer
  
  alias Aybiza.VoicePipeline.CallManager
  alias Aybiza.AgentManager.AgentRegistry
  alias Aybiza.MemoryService
  
  # Start the transfer handler
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  # Initialize the handler
  @impl true
  def init(opts) do
    {:ok, %{
      transfer_registry: %{},
      transfer_queues: %{},
      active_transfers: %{},
      settings: load_settings(opts)
    }}
  end
  
  @doc """
  Transfer a call to a human agent or another AI agent
  """
  def transfer_call(call_id, params) do
    GenServer.call(__MODULE__, {:transfer_call, call_id, params})
  end
  
  @doc """
  Check the status of a transfer
  """
  def check_transfer_status(transfer_id) do
    GenServer.call(__MODULE__, {:check_status, transfer_id})
  end
  
  @doc """
  Cancel a pending transfer
  """
  def cancel_transfer(transfer_id) do
    GenServer.call(__MODULE__, {:cancel_transfer, transfer_id})
  end
  
  # Handle transfer request
  @impl true
  def handle_call({:transfer_call, call_id, params}, _from, state) do
    # Generate unique transfer ID
    transfer_id = generate_transfer_id()
    
    # Determine transfer type
    case params.transfer_type do
      "human" ->
        handle_human_transfer(call_id, params, transfer_id, state)
        
      "ai_agent" ->
        handle_ai_transfer(call_id, params, transfer_id, state)
    end
  end
  
  # Handle human transfer
  defp handle_human_transfer(call_id, params, transfer_id, state) do
    # Get call details
    {:ok, call} = Aybiza.CallAnalytics.get_call(call_id)
    
    # Determine destination queue
    destination = resolve_human_destination(params.destination)
    
    # Generate context summary for human agent
    context_summary = generate_context_summary(call_id, params)
    
    # Calculate estimated wait time
    wait_time = calculate_wait_time(destination, call.tenant_id)
    
    # Create transfer record
    transfer = %{
      id: transfer_id,
      call_id: call_id,
      source_type: "ai_agent",
      source_id: call.agent_id,
      destination_type: "human",
      destination: destination,
      reason: params.reason,
      status: "pending",
      context_summary: context_summary,
      requested_at: DateTime.utc_now(),
      estimated_wait_time: wait_time,
      gather_information: params.gather_information,
      information_to_gather: params.information_to_gather
    }
    
    # Register transfer
    updated_state = register_transfer(transfer, state)
    
    # Add to queue
    updated_state = add_to_queue(destination, transfer, updated_state)
    
    # Gather additional information if needed
    if params.gather_information do
      gather_information(call_id, params.information_to_gather)
    end
    
    # Notify caller of transfer
    notify_caller_of_transfer(call_id, destination, wait_time)
    
    # Return transfer details
    {:reply, {:ok, %{
      success: true,
      message: "Transfer initiated to #{destination}",
      transfer_id: transfer_id,
      estimated_wait_time: wait_time
    }}, updated_state}
  end
  
  # Handle AI-to-AI transfer
  defp handle_ai_transfer(call_id, params, transfer_id, state) do
    # Get call details
    {:ok, call} = Aybiza.CallAnalytics.get_call(call_id)
    
    # Resolve destination agent
    {:ok, destination_agent} = resolve_ai_destination(params.destination)
    
    # Create transfer record
    transfer = %{
      id: transfer_id,
      call_id: call_id,
      source_type: "ai_agent",
      source_id: call.agent_id,
      destination_type: "ai_agent",
      destination: destination_agent.id,
      reason: params.reason,
      status: "in_progress",
      context_summary: params.context_summary,
      requested_at: DateTime.utc_now(),
      estimated_wait_time: 0
    }
    
    # Register transfer
    updated_state = register_transfer(transfer, state)
    
    # Execute immediate transfer to AI agent
    execute_ai_transfer(call_id, destination_agent, params.context_summary)
    
    # Return transfer details
    {:reply, {:ok, %{
      success: true,
      message: "Call transferred to #{destination_agent.name}",
      transfer_id: transfer_id,
      estimated_wait_time: 0
    }}, updated_state}
  end
  
  # Execute AI-to-AI transfer (immediate)
  defp execute_ai_transfer(call_id, destination_agent, context_summary) do
    # Update call record with new agent
    Aybiza.CallAnalytics.update_call(call_id, %{
      agent_id: destination_agent.id,
      agent_version_id: destination_agent.current_version_id
    })
    
    # Store transfer in memory
    MemoryService.store_memory(call_id, %{
      content: "Call transferred to #{destination_agent.name}",
      memory_type: "interaction",
      importance: 4,
      tags: ["transfer", "ai_agent"]
    })
    
    # Create initial prompt with context
    initial_prompt = """
    This call has been transferred to you. 
    
    Context from previous agent:
    #{context_summary}
    
    Please continue the conversation appropriately.
    """
    
    # Update call context with transfer information
    CallManager.update_call_context(call_id, %{
      transferred: true,
      previous_agent_id: destination_agent.id,
      transfer_context: context_summary,
      initial_prompt: initial_prompt
    })
  end
  
  # Register transfer in state
  defp register_transfer(transfer, state) do
    %{state | 
      transfer_registry: Map.put(state.transfer_registry, transfer.id, transfer)
    }
  end
  
  # Add transfer to appropriate queue
  defp add_to_queue(destination, transfer, state) do
    # Get or create queue
    queue = Map.get(state.transfer_queues, destination, [])
    
    # Add transfer to queue
    updated_queue = queue ++ [transfer.id]
    
    # Update state
    %{state |
      transfer_queues: Map.put(state.transfer_queues, destination, updated_queue)
    }
  end
  
  # Generate context summary for transfer
  defp generate_context_summary(call_id, params) do
    if params.context_summary do
      params.context_summary
    else
      # Get call transcript
      {:ok, transcript} = Aybiza.CallAnalytics.get_call_transcript(call_id)
      
      # Generate summary from transcript
      Aybiza.MemoryService.Summarizer.summarize_transcript(transcript.full_text)
    end
  end
  
  # Resolve human destination queue
  defp resolve_human_destination(destination) do
    # Map destination to actual queue name
    # e.g., "sales" -> "sales_team_queue"
    # Implementation depends on configuration
    destination
  end
  
  # Resolve AI agent destination
  defp resolve_ai_destination(destination) do
    # Get agent by ID or name
    AgentRegistry.get_agent(destination)
  end
  
  # Calculate estimated wait time based on queue
  defp calculate_wait_time(destination, tenant_id) do
    # Get queue stats
    queue_stats = get_queue_stats(destination, tenant_id)
    
    # Calculate wait time in seconds
    round(queue_stats.average_wait_time * queue_stats.queue_length)
  end
  
  # Get stats for a queue
  defp get_queue_stats(queue, tenant_id) do
    # Implementation depends on integration with call center system
    # or internal queue management
    %{
      queue_length: 5,         # Number of callers in queue
      average_wait_time: 30,   # Average wait time in seconds
      available_agents: 3,     # Number of available agents
      longest_wait: 120        # Longest current wait in seconds
    }
  end
  
  # Gather additional information before transfer
  defp gather_information(call_id, fields) do
    # Implementation to gather additional information
    # using conversational prompts before completing transfer
  end
  
  # Notify caller of transfer
  defp notify_caller_of_transfer(call_id, destination, wait_time) do
    # Format wait time for voice
    wait_message = if wait_time > 60 do
      minutes = div(wait_time, 60)
      "approximately #{minutes} minutes"
    else
      "less than a minute"
    end
    
    # Prepare message
    message = "I'll transfer you to #{humanize_destination(destination)}. " <>
              "The estimated wait time is #{wait_message}. " <>
              "Thank you for your patience."
    
    # Send message to caller
    CallManager.send_message(call_id, message)
  end
  
  # Convert destination to human-friendly name
  defp humanize_destination(destination) do
    case destination do
      "sales" -> "our sales team"
      "support" -> "our support team"
      "billing" -> "our billing department"
      _ -> destination
    end
  end
  
  # Generate unique transfer ID
  defp generate_transfer_id do
    "transfer_#{System.unique_integer([:positive])}"
  end
  
  # Load transfer settings
  defp load_settings(opts) do
    # Default settings
    %{
      default_transfer_timeout: opts[:default_transfer_timeout] || 300,
      max_queue_size: opts[:max_queue_size] || 50,
      callback_option_threshold: opts[:callback_option_threshold] || 600,
      recording_retention: opts[:recording_retention] || true
    }
  end
end
```

### Twilio Integration for Transfers

```elixir
defmodule Aybiza.TransferHandler.TwilioTransfer do
  @moduledoc """
  Handles Twilio-specific transfer operations
  """
  
  alias Aybiza.VoiceGateway.TwiMLGenerator
  
  @doc """
  Generates TwiML for transferring a call
  """
  def generate_transfer_twiml(destination, options \\ %{}) do
    # Get destination phone number or SIP address
    dest_address = get_destination_address(destination)
    
    # Default options
    options = Map.merge(%{
      timeout: 30,
      record: true,
      action: "/api/twilio/transfer-status",
      play_music: true
    }, options)
    
    # Generate TwiML
    {:safe, twiml} = TwiML.response() do
      if options.play_music do
        TwiML.Play(url: music_url_for_transfer(destination))
      end
      
      TwiML.Dial(
        action: options.action,
        method: "POST",
        timeout: options.timeout,
        record: options.record,
        recordingStatusCallback: "/api/twilio/recording-status"
      ) do
        case get_destination_type(dest_address) do
          :phone ->
            TwiML.Number(
              statusCallbackEvent: "initiated ringing answered completed",
              statusCallback: "/api/twilio/number-status",
              statusCallbackMethod: "POST"
            ) do
              dest_address
            end
            
          :sip ->
            TwiML.Sip(
              statusCallbackEvent: "initiated ringing answered completed",
              statusCallback: "/api/twilio/sip-status",
              statusCallbackMethod: "POST"
            ) do
              dest_address
            end
            
          :queue ->
            TwiML.Queue(
              url: "/api/twilio/queue-status",
              method: "POST"
            ) do
              dest_address
            end
            
          :conference ->
            TwiML.Conference(
              statusCallback: "/api/twilio/conference-status",
              statusCallbackEvent: "start end join leave mute hold",
              statusCallbackMethod: "POST",
              record: options.record,
              beep: true
            ) do
              dest_address
            end
        end
      end
    end
    
    twiml
  end
  
  @doc """
  Process transfer status webhooks from Twilio
  """
  def process_transfer_status(params) do
    transfer_result = %{
      call_sid: params["CallSid"],
      parent_call_sid: params["DialCallSid"],
      status: params["DialCallStatus"],
      recording_url: params["RecordingUrl"],
      recording_sid: params["RecordingSid"],
      recording_duration: params["RecordingDuration"]
    }
    
    # Update transfer status
    Aybiza.TransferHandler.update_transfer_status(
      find_transfer_by_call(params["CallSid"]),
      transfer_result
    )
    
    # Generate appropriate response TwiML
    case transfer_result.status do
      "completed" ->
        handle_completed_transfer(transfer_result)
        
      "failed" ->
        handle_failed_transfer(transfer_result)
        
      "busy" ->
        handle_busy_transfer(transfer_result)
        
      "no-answer" ->
        handle_no_answer_transfer(transfer_result)
        
      _ ->
        handle_unknown_transfer_status(transfer_result)
    end
  end
  
  # Get the destination address based on destination name
  defp get_destination_address(destination) do
    # Implementation depends on configuration
    # Maps destination names to phone numbers, SIP addresses, etc.
    case destination do
      "sales" -> "+18005551234"
      "support" -> "+18005554321"
      "sales_queue" -> "sales_queue"
      "support_queue" -> "support_queue"
      "support_conference" -> "support_conference"
      _ -> destination
    end
  end
  
  # Determine the type of destination
  defp get_destination_type(address) do
    cond do
      String.starts_with?(address, "+") ->
        :phone
        
      String.starts_with?(address, "sip:") ->
        :sip
        
      String.ends_with?(address, "_queue") ->
        :queue
        
      String.ends_with?(address, "_conference") ->
        :conference
        
      true ->
        :phone
    end
  end
  
  # Get appropriate hold music for the transfer
  defp music_url_for_transfer(destination) do
    # Different hold music for different departments
    case destination do
      "sales" -> "https://assets.aybiza.com/music/sales-hold.mp3"
      "support" -> "https://assets.aybiza.com/music/support-hold.mp3"
      _ -> "https://assets.aybiza.com/music/default-hold.mp3"
    end
  end
  
  # Find transfer by call SID
  defp find_transfer_by_call(call_sid) do
    # Implementation to look up transfer by call SID
  end
  
  # Handle completed transfer
  defp handle_completed_transfer(transfer_result) do
    # Generate TwiML for after successful transfer
    {:safe, twiml} = TwiML.response() do
      TwiML.Hangup()
    end
    
    twiml
  end
  
  # Handle failed transfer
  defp handle_failed_transfer(transfer_result) do
    # Generate TwiML for failed transfer
    {:safe, twiml} = TwiML.response() do
      TwiML.Say(voice: "Polly.Amy-Neural") do
        "I'm sorry, but the transfer failed. Let me connect you with someone else who can help."
      end
      
      TwiML.Redirect() do
        "/api/twilio/fallback-transfer"
      end
    end
    
    twiml
  end
  
  # Handle busy transfer destination
  defp handle_busy_transfer(transfer_result) do
    # Generate TwiML for busy transfer
    {:safe, twiml} = TwiML.response() do
      TwiML.Say(voice: "Polly.Amy-Neural") do
        "I'm sorry, but that department is currently busy. Would you like to leave a callback number or try a different department?"
      end
      
      TwiML.Gather(
        numDigits: 1,
        action: "/api/twilio/busy-choice",
        method: "POST"
      ) do
        TwiML.Say(voice: "Polly.Amy-Neural") do
          "Press 1 to leave a callback number. Press 2 to try a different department."
        end
      end
    end
    
    twiml
  end
  
  # Handle no-answer from transfer destination
  defp handle_no_answer_transfer(transfer_result) do
    # Generate TwiML for no-answer
    {:safe, twiml} = TwiML.response() do
      TwiML.Say(voice: "Polly.Amy-Neural") do
        "I'm sorry, but there was no answer from that department. Let me connect you with someone else who can help."
      end
      
      TwiML.Redirect() do
        "/api/twilio/fallback-transfer"
      end
    end
    
    twiml
  end
  
  # Handle unknown transfer status
  defp handle_unknown_transfer_status(transfer_result) do
    # Generate TwiML for unknown status
    {:safe, twiml} = TwiML.response() do
      TwiML.Say(voice: "Polly.Amy-Neural") do
        "I'm sorry, but something unexpected happened with the transfer. Let me connect you with someone who can help."
      end
      
      TwiML.Redirect() do
        "/api/twilio/fallback-transfer"
      end
    end
    
    twiml
  end
end
```

### Transfer Analytics

```elixir
defmodule Aybiza.TransferHandler.Analytics do
  @moduledoc """
  Tracks and analyzes transfer patterns
  """
  
  alias Aybiza.CallAnalytics.Repo
  
  @doc """
  Record transfer event
  """
  def record_transfer(transfer) do
    # Create transfer record in database
    Repo.insert!(%Aybiza.CallAnalytics.TransferEvent{
      transfer_id: transfer.id,
      call_id: transfer.call_id,
      source_type: transfer.source_type,
      source_id: transfer.source_id,
      destination_type: transfer.destination_type,
      destination: transfer.destination,
      reason: transfer.reason,
      status: transfer.status,
      requested_at: transfer.requested_at,
      completed_at: Map.get(transfer, :completed_at),
      wait_time: calculate_wait_time(transfer),
      success: transfer.status == "completed"
    })
    
    # Emit telemetry event
    :telemetry.execute(
      [:aybiza, :transfer, :completed],
      %{wait_time: calculate_wait_time(transfer)},
      %{
        transfer_id: transfer.id,
        source_type: transfer.source_type,
        destination_type: transfer.destination_type,
        destination: transfer.destination,
        success: transfer.status == "completed"
      }
    )
  end
  
  @doc """
  Get transfer metrics for a tenant
  """
  def get_transfer_metrics(tenant_id, time_range \\ :last_7_days) do
    # Query transfer events
    transfers = query_transfers(tenant_id, time_range)
    
    # Calculate metrics
    %{
      total_transfers: length(transfers),
      successful_transfers: count_successful_transfers(transfers),
      average_wait_time: calculate_average_wait_time(transfers),
      transfers_by_reason: group_transfers_by_reason(transfers),
      transfers_by_destination: group_transfers_by_destination(transfers),
      transfers_by_time: group_transfers_by_time(transfers),
      common_transfer_paths: identify_common_transfer_paths(transfers)
    }
  end
  
  @doc """
  Identify potential transfer improvement opportunities
  """
  def identify_improvement_opportunities(tenant_id) do
    # Get recent transfers
    transfers = query_transfers(tenant_id, :last_30_days)
    
    # Identify patterns and opportunities
    [
      identify_high_transfer_rate_agents(transfers),
      identify_common_pre_transfer_phrases(transfers),
      identify_unsuccessful_transfers(transfers),
      identify_long_wait_transfers(transfers),
      identify_repeated_transfers(transfers)
    ]
    |> Enum.filter(&(&1 != nil))
  end
  
  # Calculate wait time for a transfer
  defp calculate_wait_time(transfer) do
    case {transfer.requested_at, transfer.completed_at} do
      {requested, completed} when not is_nil(completed) ->
        DateTime.diff(completed, requested, :second)
        
      _ ->
        nil
    end
  end
  
  # Query transfer events from database
  defp query_transfers(tenant_id, time_range) do
    # Implementation to query database
    []
  end
  
  # Count successful transfers
  defp count_successful_transfers(transfers) do
    Enum.count(transfers, & &1.success)
  end
  
  # Calculate average wait time
  defp calculate_average_wait_time(transfers) do
    wait_times = Enum.filter(transfers, & &1.wait_time != nil)
    
    if Enum.empty?(wait_times) do
      0
    else
      wait_times
      |> Enum.map(& &1.wait_time)
      |> Enum.sum()
      |> Kernel./(length(wait_times))
    end
  end
  
  # Group transfers by reason
  defp group_transfers_by_reason(transfers) do
    Enum.group_by(transfers, & &1.reason)
    |> Enum.map(fn {reason, transfers} ->
      {reason, length(transfers)}
    end)
    |> Enum.into(%{})
  end
  
  # Group transfers by destination
  defp group_transfers_by_destination(transfers) do
    Enum.group_by(transfers, & &1.destination)
    |> Enum.map(fn {destination, transfers} ->
      {destination, length(transfers)}
    end)
    |> Enum.into(%{})
  end
  
  # Group transfers by time
  defp group_transfers_by_time(transfers) do
    Enum.group_by(transfers, fn transfer ->
      hour = transfer.requested_at.hour
      
      cond do
        hour >= 9 and hour < 12 -> :morning
        hour >= 12 and hour < 15 -> :early_afternoon
        hour >= 15 and hour < 18 -> :late_afternoon
        hour >= 18 and hour < 21 -> :evening
        true -> :night
      end
    end)
    |> Enum.map(fn {time, transfers} ->
      {time, length(transfers)}
    end)
    |> Enum.into(%{})
  end
  
  # Identify common transfer paths
  defp identify_common_transfer_paths(transfers) do
    # Implementation to identify common paths
    # e.g., AI Sales Agent -> Human Sales -> Human Support
    []
  end
  
  # Identify agents with high transfer rates
  defp identify_high_transfer_rate_agents(transfers) do
    # Implementation to identify agents
    nil
  end
  
  # Identify common phrases before transfers
  defp identify_common_pre_transfer_phrases(transfers) do
    # Implementation to identify phrases
    nil
  end
  
  # Identify unsuccessful transfers
  defp identify_unsuccessful_transfers(transfers) do
    # Implementation to identify unsuccessful transfers
    nil
  end
  
  # Identify transfers with long wait times
  defp identify_long_wait_transfers(transfers) do
    # Implementation to identify long waits
    nil
  end
  
  # Identify repeated transfers for same caller
  defp identify_repeated_transfers(transfers) do
    # Implementation to identify repeated transfers
    nil
  end
end
```

## Configuration

### Organization-Level Configuration

```elixir
%{
  # Default settings for all agents in the organization
  organization_transfer_settings: %{
    # Enable/disable transfers completely
    transfers_enabled: true,
    
    # Enable/disable specific transfer types
    human_transfers_enabled: true,
    ai_agent_transfers_enabled: true,
    
    # Default transfer destinations
    default_human_transfer_destination: "support",
    
    # Default messages
    transfer_initiation_message: "I'll transfer you to a specialist who can help with this.",
    transfer_wait_message: "Please hold while I connect you. The estimated wait time is %{wait_time}.",
    
    # Analytics and compliance
    recording_retention: true,
    
    # Callback options
    offer_callback_threshold: 120,  # seconds
    callback_scheduling_enabled: true
  }
}
```

### Agent-Level Configuration

```elixir
%{
  # Agent-specific transfer settings (overrides organization defaults)
  agent_transfer_settings: %{
    # When to offer transfer (can be customized based on agent role)
    transfer_triggers: [
      "explicit_request",       # User explicitly asks for human
      "complex_issue",          # Issue too complex for AI
      "high_sentiment",         # User shows high negative sentiment
      "authentication_failure", # Failed authentication attempts
      "repeated_misunderstanding" # AI repeatedly misunderstands user
    ],
    
    # Which departments/teams this agent can transfer to
    allowed_transfer_destinations: [
      "sales",
      "support",
      "billing",
      "technical_specialists",
      "account_managers"
    ],
    
    # Which AI agents this agent can transfer to
    allowed_ai_agent_transfers: [
      "agent_sales_specialist",
      "agent_technical_support",
      "agent_account_manager"
    ],
    
    # Information to gather before transfer
    pre_transfer_information: [
      "account_number",
      "issue_summary",
      "attempted_solutions"
    ],
    
    # Custom transfer messages
    transfer_messages: %{
      sales: "I'll connect you with our sales team to discuss pricing options.",
      support: "Let me transfer you to our support team for further assistance.",
      technical: "Let me connect you with a technical specialist who can help with this issue."
    }
  }
}
```

## Monitoring

The Transfer Handler provides comprehensive monitoring capabilities:

1. **Real-time Queue Dashboard**: Live view of transfer queues and wait times
2. **Transfer Analytics**: Patterns, success rates, and wait time analysis
3. **Agent Performance**: Transfer rates by agent and reason
4. **Optimization Recommendations**: AI-driven recommendations to improve transfer processes

## Security and Compliance

1. **Call Recording**: Option to record transfers for quality assurance
2. **PCI Compliance**: Pause recording during sensitive information collection
3. **Access Controls**: Role-based access to transfer functions
4. **Audit Logging**: Comprehensive logging of all transfer events

## Integration

The Transfer Handler integrates with:

1. **Twilio**: For executing call transfers
2. **CRM Systems**: For tracking customer interactions
3. **Memory Service**: For maintaining context through transfers
4. **Analytics**: For tracking transfer metrics
5. **AI Models**: For determining when transfers are necessary