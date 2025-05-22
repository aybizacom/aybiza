# AYBIZA Tool Registry Implementation

## Overview

The Tool Registry is the heart of the AYBIZA AI Voice Business Agent platform, providing agents with direct access to powerful tools through Claude's function calling mechanism. This document outlines the technical implementation of the Tool Registry.

## Architecture

The Tool Registry follows these architectural principles:

1. **LLM-First Design**: All tools are designed to be called directly by the LLM
2. **Direct Integration**: Tools connect directly to external systems with no intermediary layers
3. **Standardized Interface**: Tools follow a consistent interface pattern for implementation
4. **Runtime Discovery**: Tools can be dynamically loaded and discovered at runtime
5. **Monitoring Capabilities**: All tool executions are tracked and monitored
6. **Security Controls**: Access to tools is governed by comprehensive permissions
7. **Complex Operations**: Tools can handle multi-step workflows and conditional logic internally
8. **Tool Composition**: Tools can call other tools to create complex business processes
9. **External Integrations**: Comprehensive framework for integrating with external systems

## Implementation

### Core Tool Registry

```elixir
defmodule Aybiza.ToolRegistry do
  @moduledoc """
  Registry for AI agent tools with function calling support
  """
  
  use GenServer
  
  require Logger
  
  # Start the registry
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  # Initialize the registry
  @impl true
  def init(opts) do
    # Load tools from modules
    {:ok, %{
      tools: load_tools(),
      stats: %{},
      cache: %{},
      settings: load_settings(opts)
    }}
  end
  
  @doc """
  Get tools for Claude function calling
  """
  def get_tools_for_function_calling(agent_id, user_info \\ nil) do
    GenServer.call(__MODULE__, {:get_tools_for_function_calling, agent_id, user_info})
  end
  
  @doc """
  Execute a tool function
  """
  def execute_tool(call_id, tool_name, params) do
    GenServer.call(__MODULE__, {:execute_tool, call_id, tool_name, params})
  end
  
  @doc """
  Get available tools for an agent
  """
  def get_agent_tools(agent_id) do
    GenServer.call(__MODULE__, {:get_agent_tools, agent_id})
  end
  
  @doc """
  Register a new custom tool
  """
  def register_tool(tool_definition, opts \\ %{}) do
    GenServer.call(__MODULE__, {:register_tool, tool_definition, opts})
  end
  
  @doc """
  Get tool usage statistics
  """
  def get_tool_stats(opts \\ %{}) do
    GenServer.call(__MODULE__, {:get_tool_stats, opts})
  end
  
  # Handle function calling tool request
  @impl true
  def handle_call({:get_tools_for_function_calling, agent_id, user_info}, _from, state) do
    # Get allowed tools for this agent
    allowed_tools = get_allowed_tools(agent_id, state.tools, user_info)
    
    # Convert to Claude function calling format
    tool_schemas = Enum.map(allowed_tools, fn {name, tool} ->
      %{
        name: name,
        description: tool.description,
        parameters: tool.parameters
      }
    end)
    
    {:reply, tool_schemas, state}
  end
  
  # Handle tool execution
  @impl true
  def handle_call({:execute_tool, call_id, tool_name, params}, _from, state) do
    # Verify tool exists
    case Map.fetch(state.tools, tool_name) do
      {:ok, tool} ->
        # Get call info
        {:ok, call} = Aybiza.CallAnalytics.get_call(call_id)
        
        # Verify tool is allowed for this agent
        if tool_allowed?(call.agent_id, tool_name, state.tools) do
          # Execute the tool
          {result, updated_state} = execute_tool_impl(tool, call_id, params, state)
          
          {:reply, result, updated_state}
        else
          # Tool not allowed for this agent
          {:reply, 
            %{
              error: "Tool not allowed",
              message: "The #{tool_name} tool is not authorized for this agent"
            }, 
            state
          }
        end
        
      :error ->
        # Tool not found
        {:reply, 
          %{
            error: "Tool not found",
            message: "The #{tool_name} tool is not available"
          }, 
          state
        }
    end
  end
  
  # Handle agent tools request
  @impl true
  def handle_call({:get_agent_tools, agent_id}, _from, state) do
    allowed_tools = get_allowed_tools(agent_id, state.tools, nil)
    {:reply, allowed_tools, state}
  end
  
  # Handle tool registration
  @impl true
  def handle_call({:register_tool, tool_definition, opts}, _from, state) do
    with :ok <- validate_tool_definition(tool_definition) do
      # Add tool to registry
      updated_tools = Map.put(state.tools, tool_definition.name, tool_definition)
      
      # Log registration
      Logger.info("Tool registered: #{tool_definition.name}", %{
        tool: tool_definition.name,
        custom: true
      })
      
      {:reply, :ok, %{state | tools: updated_tools}}
    else
      {:error, reason} ->
        {:reply, {:error, reason}, state}
    end
  end
  
  # Handle stats request
  @impl true
  def handle_call({:get_tool_stats, opts}, _from, state) do
    stats = filter_stats(state.stats, opts)
    {:reply, stats, state}
  end
  
  # Execute a tool and update stats
  defp execute_tool_impl(tool, call_id, params, state) do
    # Get call details for context
    {:ok, call} = Aybiza.CallAnalytics.get_call(call_id)
    
    # Resolve module
    module = resolve_tool_module(tool.name)
    
    # Execute tool with timing
    start_time = System.monotonic_time(:millisecond)
    
    result = try do
      apply(module, :execute, [call_id, params])
    rescue
      e ->
        Logger.error("Tool execution failed", %{
          tool: tool.name,
          call_id: call_id,
          error: inspect(e),
          stacktrace: __STACKTRACE__
        })
        
        %{
          error: "Execution failed",
          message: "The tool encountered an error: #{inspect(e)}"
        }
    end
    
    end_time = System.monotonic_time(:millisecond)
    duration = end_time - start_time
    
    # Update stats
    updated_stats = update_tool_stats(state.stats, tool.name, duration, result)
    
    # Log execution
    log_tool_execution(tool.name, call_id, params, result, duration)
    
    # Record in database for analytics
    record_tool_execution(tool.name, call_id, params, result, duration, call.tenant_id)
    
    {result, %{state | stats: updated_stats}}
  end
  
  # Load tool definitions from modules
  defp load_tools do
    # Core tools
    default_tools = [
      Aybiza.Tools.TransferCall,
      Aybiza.Tools.HoldCall,
      Aybiza.Tools.ResumeCall,
      Aybiza.Tools.RetrieveMemory,
      Aybiza.Tools.StoreMemory,
      Aybiza.Tools.SearchCustomer,
      Aybiza.Tools.CreateSupportTicket,
      Aybiza.Tools.GetAvailability,
      Aybiza.Tools.GetProductInformation,
      Aybiza.Tools.CreateQuote,
      Aybiza.Tools.GetKnowledgeBase,
      Aybiza.Tools.CheckServiceStatus
    ]
    
    # Dynamic tools from database
    dynamic_tools = load_dynamic_tools()
    
    # Combine and convert to map
    (default_tools ++ dynamic_tools)
    |> Enum.map(fn module -> 
      tool = apply(module, :definition, [])
      {tool.name, tool}
    end)
    |> Enum.into(%{})
  end
  
  # Load dynamically defined tools from database
  defp load_dynamic_tools do
    # Implementation to load custom tools from database
    []
  end
  
  # Get allowed tools for an agent
  defp get_allowed_tools(agent_id, tools, user_info) do
    # Get agent details
    {:ok, agent} = Aybiza.AgentManager.get_agent(agent_id)
    
    # Get tool permissions from agent config
    allowed_tool_names = get_allowed_tool_names(agent)
    
    # Filter tools by permissions
    tools
    |> Enum.filter(fn {name, _} -> name in allowed_tool_names end)
    |> Enum.into(%{})
  end
  
  # Check if a tool is allowed for an agent
  defp tool_allowed?(agent_id, tool_name, tools) do
    # Get agent details
    {:ok, agent} = Aybiza.AgentManager.get_agent(agent_id)
    
    # Get allowed tools
    allowed_tool_names = get_allowed_tool_names(agent)
    
    # Check if tool is allowed
    tool_name in allowed_tool_names
  end
  
  # Get allowed tool names from agent config
  defp get_allowed_tool_names(agent) do
    # Get tool permissions from agent config
    case agent.tool_permissions do
      %{"allowed_tools" => allowed_tools} when is_list(allowed_tools) ->
        allowed_tools
        
      _ ->
        # Default set of tools if not specified
        [
          "transfer_call", 
          "hold_call", 
          "resume_call", 
          "retrieve_memory", 
          "store_memory"
        ]
    end
  end
  
  # Resolve module for a tool
  defp resolve_tool_module(tool_name) do
    # Convert snake_case to CamelCase for module name
    module_suffix = tool_name
    |> String.split("_")
    |> Enum.map(&String.capitalize/1)
    |> Enum.join("")
    
    # Try to find module
    module_name = "Elixir.Aybiza.Tools.#{module_suffix}"
    String.to_existing_atom(module_name)
  end
  
  # Update tool stats
  defp update_tool_stats(stats, tool_name, duration, result) do
    # Create new stats entry if needed
    stats = Map.put_new(stats, tool_name, %{
      count: 0,
      success_count: 0,
      error_count: 0,
      total_duration: 0,
      average_duration: 0,
      last_used: nil
    })
    
    # Get current stats
    tool_stats = stats[tool_name]
    
    # Check if result was successful
    success = not Map.has_key?(result, :error)
    
    # Update stats
    updated_tool_stats = %{
      count: tool_stats.count + 1,
      success_count: tool_stats.success_count + (if success, do: 1, else: 0),
      error_count: tool_stats.error_count + (if success, do: 0, else: 1),
      total_duration: tool_stats.total_duration + duration,
      average_duration: (tool_stats.total_duration + duration) / (tool_stats.count + 1),
      last_used: DateTime.utc_now()
    }
    
    # Update stats map
    Map.put(stats, tool_name, updated_tool_stats)
  end
  
  # Log tool execution
  defp log_tool_execution(tool_name, call_id, params, result, duration) do
    # Sanitize params for logging (remove sensitive info)
    sanitized_params = sanitize_params(params)
    
    # Log execution
    Logger.info("Tool executed: #{tool_name}", %{
      tool: tool_name,
      call_id: call_id,
      params: sanitized_params,
      result_type: if Map.has_key?(result, :error), do: "error", else: "success",
      duration_ms: duration
    })
  end
  
  # Record tool execution in database
  defp record_tool_execution(tool_name, call_id, params, result, duration, tenant_id) do
    # Insert record into database
    Aybiza.CallAnalytics.Repo.insert!(%Aybiza.CallAnalytics.ToolExecution{
      call_id: call_id,
      tool_name: tool_name,
      params: sanitize_params(params),
      result: sanitize_result(result),
      duration_ms: duration,
      success: not Map.has_key?(result, :error),
      tenant_id: tenant_id,
      executed_at: DateTime.utc_now()
    })
    
    # Emit telemetry
    :telemetry.execute(
      [:aybiza, :tool, :execution],
      %{duration: duration},
      %{
        tool: tool_name,
        success: not Map.has_key?(result, :error),
        tenant_id: tenant_id
      }
    )
  end
  
  # Filter stats based on options
  defp filter_stats(stats, opts) do
    # Apply filters
    stats
  end
  
  # Sanitize parameters for logging (remove sensitive info)
  defp sanitize_params(params) do
    # Strip sensitive fields
    sensitive_fields = ["password", "token", "secret", "key", "auth", "credentials"]
    
    sanitize_map(params, sensitive_fields)
  end
  
  # Sanitize result for logging
  defp sanitize_result(result) do
    # Strip sensitive fields
    sensitive_fields = ["password", "token", "secret", "key", "auth", "credentials"]
    
    sanitize_map(result, sensitive_fields)
  end
  
  # Sanitize a map recursively
  defp sanitize_map(map, sensitive_fields) when is_map(map) do
    Enum.reduce(map, %{}, fn {k, v}, acc ->
      if is_binary(k) and Enum.any?(sensitive_fields, &String.contains?(String.downcase(k), &1)) do
        Map.put(acc, k, "[REDACTED]")
      else
        Map.put(acc, k, sanitize_value(v, sensitive_fields))
      end
    end)
  end
  
  # Sanitize a list recursively
  defp sanitize_value(list, sensitive_fields) when is_list(list) do
    Enum.map(list, &sanitize_value(&1, sensitive_fields))
  end
  
  # Sanitize a map recursively
  defp sanitize_value(map, sensitive_fields) when is_map(map) do
    sanitize_map(map, sensitive_fields)
  end
  
  # Pass through other values
  defp sanitize_value(value, _sensitive_fields) do
    value
  end
  
  # Validate tool definition
  defp validate_tool_definition(tool) do
    # Implementation to validate tool definition
    :ok
  end
  
  # Load registry settings
  defp load_settings(opts) do
    # Default settings
    %{
      default_timeout: opts[:default_timeout] || 5000,
      cache_enabled: opts[:cache_enabled] || true,
      cache_ttl: opts[:cache_ttl] || 300,
      log_execution: opts[:log_execution] || true
    }
  end
end
```

### Tool Base Module

```elixir
defmodule Aybiza.Tools.Base do
  @moduledoc """
  Base module for all tools
  """
  
  @callback definition() :: map()
  @callback execute(call_id :: String.t(), params :: map()) :: map()
  
  defmacro __using__(opts) do
    quote do
      @behaviour Aybiza.Tools.Base
      
      require Logger
      
      # Shared helper functions for all tools
      
      # Convert caller response to standard format
      def format_response(data) do
        # Implementation of standard response formatting
        data
      end
      
      # Log tool execution
      defp log_execution(call_id, params, result, duration) do
        Logger.info("Tool executed: #{__MODULE__}", %{
          tool: to_string(__MODULE__),
          call_id: call_id,
          duration_ms: duration
        })
      end
      
      # Validate tool parameters
      defp validate_params(params, required_fields) do
        # Check that all required fields are present
        missing_fields = Enum.filter(required_fields, fn field ->
          not Map.has_key?(params, field)
        end)
        
        if Enum.empty?(missing_fields) do
          :ok
        else
          {:error, "Missing required parameters: #{Enum.join(missing_fields, ", ")}"}
        end
      end
    end
  end
end
```

### Sample Tool Implementation

```elixir
defmodule Aybiza.Tools.TransferCall do
  @moduledoc """
  Tool for transferring calls to human agents or other AI agents
  """
  
  use Aybiza.Tools.Base
  
  alias Aybiza.TransferHandler
  alias Aybiza.MemoryService
  
  @impl true
  def definition do
    %{
      name: "transfer_call",
      description: "Transfers the current call to a human agent or another AI agent. Use this when a caller explicitly requests to speak with a human, or when the conversation requires human assistance.",
      parameters: %{
        type: "object",
        properties: %{
          transfer_type: %{
            type: "string",
            enum: ["human", "ai_agent"],
            description: "Whether to transfer to a human agent or another AI agent"
          },
          destination: %{
            type: "string",
            description: "Phone number, department name, agent name, or agent ID. For human transfers, use department names like 'sales' or 'support'. For AI transfers, use the agent ID."
          },
          reason: %{
            type: "string",
            description: "The reason for the transfer, used for routing and tracking"
          },
          context_summary: %{
            type: "string",
            description: "A brief summary of the conversation to provide context to the receiving agent"
          },
          gather_information: %{
            type: "boolean",
            description: "Whether to gather additional information before transferring"
          },
          information_to_gather: %{
            type: "array",
            items: %{
              type: "string"
            },
            description: "List of information fields to gather before transfer (e.g., name, account number)"
          }
        },
        required: ["transfer_type", "destination", "reason"]
      },
      returns: %{
        type: "object",
        properties: %{
          success: %{
            type: "boolean",
            description: "Whether the transfer was successful"
          },
          message: %{
            type: "string",
            description: "Status message about the transfer"
          },
          transfer_id: %{
            type: "string",
            description: "Unique ID for the transfer for tracking purposes"
          },
          estimated_wait_time: %{
            type: "integer",
            description: "Estimated wait time in seconds (for human transfers)"
          }
        }
      },
      memory_access: :read_write,
      auth_required: true,
      timeout: 5000
    }
  end
  
  @impl true
  def execute(call_id, params) do
    # Validate parameters
    case validate_params(params, ["transfer_type", "destination", "reason"]) do
      :ok ->
        # Initialize params with defaults
        params = Map.merge(%{
          gather_information: false,
          information_to_gather: []
        }, params)
        
        # Execute transfer
        start_time = System.monotonic_time(:millisecond)
        result = TransferHandler.transfer_call(call_id, params)
        end_time = System.monotonic_time(:millisecond)
        
        # Store in memory
        store_transfer_in_memory(call_id, result, params)
        
        # Log execution
        log_execution(call_id, params, result, end_time - start_time)
        
        # Return formatted result
        format_response(result)
        
      {:error, message} ->
        # Return error
        %{
          success: false,
          message: message
        }
    end
  end
  
  # Store transfer in memory
  defp store_transfer_in_memory(call_id, result, params) do
    MemoryService.store_memory(call_id, %{
      content: "Call transferred to #{params.destination} (#{params.transfer_type}). " <>
               "Reason: #{params.reason}. Transfer ID: #{result.transfer_id}",
      memory_type: "interaction",
      importance: 4,
      tags: ["transfer", params.transfer_type]
    })
  end
end
```

### Tool Permissions

Tools can be enabled or disabled at different levels:

#### Organization Level

```elixir
%{
  # Organization-wide tool settings
  organization_tool_settings: %{
    # Default allowed tools for all agents
    default_allowed_tools: [
      "transfer_call", 
      "hold_call", 
      "resume_call", 
      "retrieve_memory", 
      "store_memory",
      "search_customer",
      "get_knowledge_base"
    ],
    
    # Tools that require explicit permission
    restricted_tools: [
      "create_support_ticket",
      "create_quote"
    ],
    
    # Tools that are disabled for all agents
    disabled_tools: []
  }
}
```

#### Agent Level

```elixir
%{
  # Agent-specific tool settings
  agent_tool_permissions: %{
    # Tools this agent can access
    allowed_tools: [
      "transfer_call", 
      "hold_call", 
      "resume_call", 
      "retrieve_memory", 
      "store_memory",
      "search_customer",
      "get_knowledge_base",
      "create_support_ticket"
    ],
    
    # Tool-specific settings
    tool_settings: %{
      "transfer_call": %{
        allowed_destinations: ["sales", "support", "technical"],
        require_confirmation: true
      },
      "create_support_ticket": %{
        max_priority: "high",
        allowed_categories: ["billing", "technical", "account"]
      }
    }
  }
}
```

## Tool Categories

Tools are organized into the following categories:

### 1. Call Management

- `transfer_call`: Transfer to human or AI agent
- `hold_call`: Place call on hold
- `resume_call`: Resume call from hold
- `schedule_callback`: Schedule callback appointment
- `end_call`: End current call

### 2. Memory and Context

- `retrieve_memory`: Get relevant memories
- `store_memory`: Store important information
- `get_user_profile`: Retrieve user profile information
- `update_user_profile`: Update user profile

### 3. Integration Tools

- `search_customer`: Search for customer in CRM
- `create_support_ticket`: Create support ticket
- `create_quote`: Create price quote
- `schedule_appointment`: Schedule customer appointment
- `get_order_status`: Check order status
- `update_order`: Update order information

### 4. Knowledge Tools

- `get_knowledge_base`: Search knowledge base
- `get_product_information`: Get product details
- `check_service_status`: Check system status
- `get_availability`: Check resource availability
- `find_nearest_location`: Find closest brick-and-mortar location

### 5. Complex Workflow Tools

- `process_customer_onboarding`: Multi-step customer onboarding workflow
- `handle_payment_processing`: Complete payment processing with validation
- `execute_escalation_workflow`: Automated escalation with multiple paths
- `manage_appointment_lifecycle`: Full appointment management process

### 6. External System Integration Tools

- `salesforce_integration`: Complete Salesforce CRM operations
- `hubspot_integration`: HubSpot marketing and sales operations
- `stripe_payment_processing`: Stripe payment processing and management
- `slack_notifications`: Slack messaging and notifications
- `email_automation`: Email campaigns and automated messaging
- `sms_automation`: SMS campaigns and notifications

### 7. Conditional Logic Tools

- `evaluate_business_rules`: Execute complex business rule evaluation
- `decision_tree_processor`: Navigate decision trees with multiple outcomes
- `risk_assessment`: Automated risk assessment with scoring
- `compliance_checker`: Regulatory compliance validation

## Enhanced Tool Examples

### Complex Workflow Tool Implementation

```elixir
defmodule Aybiza.Tools.ProcessCustomerOnboarding do
  @moduledoc """
  Complex multi-step customer onboarding workflow tool
  """
  
  use Aybiza.Tools.Base
  
  alias Aybiza.ToolRegistry
  alias Aybiza.MemoryService
  
  @impl true
  def definition do
    %{
      name: "process_customer_onboarding",
      description: "Execute complete customer onboarding workflow including CRM creation, welcome email, and follow-up scheduling",
      parameters: %{
        type: "object",
        properties: %{
          customer_data: %{
            type: "object",
            properties: %{
              name: %{type: "string"},
              email: %{type: "string"},
              phone: %{type: "string"},
              company: %{type: "string"},
              interested_products: %{type: "array", items: %{type: "string"}}
            },
            required: ["name", "email", "phone"]
          },
          onboarding_type: %{
            type: "string",
            enum: ["basic", "premium", "enterprise"],
            description: "Type of onboarding workflow to execute"
          },
          send_welcome_kit: %{
            type: "boolean",
            default: true,
            description: "Whether to send physical welcome kit"
          }
        },
        required: ["customer_data", "onboarding_type"]
      },
      returns: %{
        type: "object",
        properties: %{
          success: %{type: "boolean"},
          customer_id: %{type: "string"},
          crm_record_id: %{type: "string"},
          welcome_email_sent: %{type: "boolean"},
          follow_up_scheduled: %{type: "boolean"},
          next_steps: %{type: "array", items: %{type: "string"}},
          estimated_completion: %{type: "string"}
        }
      },
      complexity: :high,
      estimated_duration: 30000  # 30 seconds
    }
  end
  
  @impl true
  def execute(call_id, params) do
    customer_data = params["customer_data"]
    onboarding_type = params["onboarding_type"]
    
    # Step 1: Create CRM record
    crm_result = ToolRegistry.execute_tool(call_id, "salesforce_integration", %{
      "action" => "create_lead",
      "data" => customer_data
    })
    
    if not crm_result.success do
      return %{success: false, error: "Failed to create CRM record"}
    end
    
    # Step 2: Send welcome email based on onboarding type
    email_template = get_email_template(onboarding_type)
    email_result = ToolRegistry.execute_tool(call_id, "email_automation", %{
      "action" => "send_template_email",
      "to" => customer_data["email"],
      "template" => email_template,
      "variables" => %{
        "customer_name" => customer_data["name"],
        "company" => customer_data["company"]
      }
    })
    
    # Step 3: Schedule follow-up based on type
    follow_up_days = case onboarding_type do
      "basic" -> 3
      "premium" -> 1
      "enterprise" -> 0.5  # Same day follow-up
    end
    
    follow_up_result = ToolRegistry.execute_tool(call_id, "schedule_appointment", %{
      "customer_id" => crm_result.customer_id,
      "type" => "onboarding_follow_up",
      "days_from_now" => follow_up_days,
      "duration_minutes" => get_follow_up_duration(onboarding_type)
    })
    
    # Step 4: Store onboarding progress in memory
    MemoryService.store_memory(call_id, %{
      content: "Customer onboarding initiated for #{customer_data["name"]} (#{onboarding_type})",
      memory_type: "workflow_progress",
      importance: 4,
      tags: ["onboarding", onboarding_type, "workflow"],
      metadata: %{
        customer_id: crm_result.customer_id,
        onboarding_type: onboarding_type,
        status: "in_progress"
      }
    })
    
    # Step 5: Send welcome kit if requested
    welcome_kit_sent = if params["send_welcome_kit"] do
      kit_result = ToolRegistry.execute_tool(call_id, "fulfillment_integration", %{
        "action" => "send_welcome_kit",
        "customer_id" => crm_result.customer_id,
        "kit_type" => onboarding_type
      })
      kit_result.success
    else
      false
    end
    
    # Determine next steps
    next_steps = build_next_steps(onboarding_type, customer_data)
    
    %{
      success: true,
      customer_id: crm_result.customer_id,
      crm_record_id: crm_result.record_id,
      welcome_email_sent: email_result.success,
      follow_up_scheduled: follow_up_result.success,
      welcome_kit_sent: welcome_kit_sent,
      next_steps: next_steps,
      estimated_completion: calculate_completion_date(onboarding_type)
    }
  end
  
  defp get_email_template("basic"), do: "basic_welcome"
  defp get_email_template("premium"), do: "premium_welcome"
  defp get_email_template("enterprise"), do: "enterprise_welcome"
  
  defp get_follow_up_duration("basic"), do: 30
  defp get_follow_up_duration("premium"), do: 45
  defp get_follow_up_duration("enterprise"), do: 60
  
  defp build_next_steps("basic", _customer_data) do
    [
      "Account setup email sent",
      "Follow-up call scheduled for 3 days",
      "Basic training materials provided"
    ]
  end
  
  defp build_next_steps("premium", customer_data) do
    [
      "Premium account setup initiated",
      "Dedicated success manager assigned",
      "Follow-up call scheduled for tomorrow",
      "Premium training session to be scheduled",
      "Product demo invitation sent"
    ]
  end
  
  defp build_next_steps("enterprise", customer_data) do
    [
      "Enterprise account creation in progress",
      "Technical architect assigned",
      "Same-day follow-up call scheduled",
      "Custom implementation planning session",
      "Security and compliance review initiated",
      "Executive sponsor introduction scheduled"
    ]
  end
  
  defp calculate_completion_date("basic"), do: "7-10 business days"
  defp calculate_completion_date("premium"), do: "3-5 business days"
  defp calculate_completion_date("enterprise"), do: "1-2 business days"
end
```

### External System Integration Tool

```elixir
defmodule Aybiza.Tools.SalesforceIntegration do
  @moduledoc """
  Comprehensive Salesforce CRM integration tool
  """
  
  use Aybiza.Tools.Base
  
  @impl true
  def definition do
    %{
      name: "salesforce_integration",
      description: "Perform comprehensive Salesforce CRM operations including lead creation, opportunity management, and data synchronization",
      parameters: %{
        type: "object",
        properties: %{
          action: %{
            type: "string",
            enum: ["create_lead", "create_contact", "create_opportunity", "update_lead", "search_accounts", "sync_data"],
            description: "The Salesforce operation to perform"
          },
          data: %{
            type: "object",
            description: "Data for the operation (structure varies by action)"
          },
          options: %{
            type: "object",
            properties: %{
              assign_to: %{type: "string"},
              follow_up_task: %{type: "boolean"},
              notification_emails: %{type: "array", items: %{type: "string"}}
            }
          }
        },
        required: ["action", "data"]
      },
      returns: %{
        type: "object",
        properties: %{
          success: %{type: "boolean"},
          salesforce_id: %{type: "string"},
          record_url: %{type: "string"},
          assigned_owner: %{type: "string"},
          next_actions: %{type: "array", items: %{type: "string"}}
        }
      },
      external_service: "salesforce",
      rate_limit: %{requests_per_minute: 100}
    }
  end
  
  @impl true
  def execute(call_id, params) do
    action = params["action"]
    data = params["data"]
    options = params["options"] || %{}
    
    case action do
      "create_lead" -> create_lead(call_id, data, options)
      "create_contact" -> create_contact(call_id, data, options)
      "create_opportunity" -> create_opportunity(call_id, data, options)
      "update_lead" -> update_lead(call_id, data, options)
      "search_accounts" -> search_accounts(call_id, data, options)
      "sync_data" -> sync_data(call_id, data, options)
      _ -> %{success: false, error: "Unknown action: #{action}"}
    end
  end
  
  defp create_lead(call_id, data, options) do
    # Get Salesforce credentials
    {:ok, credentials} = get_salesforce_credentials(call_id)
    
    # Prepare lead data
    lead_data = %{
      "FirstName" => data["first_name"] || extract_first_name(data["name"]),
      "LastName" => data["last_name"] || extract_last_name(data["name"]),
      "Company" => data["company"],
      "Email" => data["email"],
      "Phone" => data["phone"],
      "LeadSource" => "Voice Agent",
      "Status" => "New"
    }
    
    # Add custom fields based on call context
    lead_data = add_call_context(lead_data, call_id)
    
    # Create lead in Salesforce
    case SalesforceAPI.create_record("Lead", lead_data, credentials) do
      {:ok, response} ->
        sf_id = response["id"]
        
        # Assign to rep if specified
        if options["assign_to"] do
          assign_lead(sf_id, options["assign_to"], credentials)
        end
        
        # Create follow-up task if requested
        if options["follow_up_task"] do
          create_follow_up_task(sf_id, call_id, credentials)
        end
        
        # Send notifications
        if options["notification_emails"] do
          send_lead_notifications(sf_id, options["notification_emails"])
        end
        
        %{
          success: true,
          salesforce_id: sf_id,
          record_url: "https://your-org.salesforce.com/#{sf_id}",
          assigned_owner: get_lead_owner(sf_id, credentials),
          next_actions: ["Lead created and assigned", "Follow-up task scheduled"]
        }
        
      {:error, error} ->
        %{success: false, error: "Salesforce API error: #{inspect(error)}"}
    end
  end
  
  defp add_call_context(lead_data, call_id) do
    # Get call information
    {:ok, call_info} = Aybiza.CallAnalytics.get_call(call_id)
    
    lead_data
    |> Map.put("Description", "Lead generated from voice call on #{Date.utc_today()}")
    |> Map.put("LeadSource", "Voice Agent - #{call_info.agent_name}")
    |> Map.put("Call_Duration__c", call_info.duration_seconds)
    |> Map.put("Agent_ID__c", call_info.agent_id)
  end
end
```

### Decision Tree Tool

```elixir
defmodule Aybiza.Tools.DecisionTreeProcessor do
  @moduledoc """
  Navigate complex decision trees with conditional logic
  """
  
  use Aybiza.Tools.Base
  
  @impl true
  def definition do
    %{
      name: "decision_tree_processor",
      description: "Process complex decision trees with multiple conditional paths and outcomes",
      parameters: %{
        type: "object",
        properties: %{
          tree_name: %{
            type: "string",
            description: "Name of the decision tree to process"
          },
          input_data: %{
            type: "object",
            description: "Input data for decision evaluation"
          },
          context: %{
            type: "object",
            description: "Additional context for decision making"
          }
        },
        required: ["tree_name", "input_data"]
      },
      returns: %{
        type: "object",
        properties: %{
          success: %{type: "boolean"},
          decision_path: %{type: "array", items: %{type: "string"}},
          final_outcome: %{type: "string"},
          confidence_score: %{type: "number"},
          recommended_actions: %{type: "array", items: %{type: "string"}},
          escalation_required: %{type: "boolean"}
        }
      }
    }
  end
  
  @impl true
  def execute(call_id, params) do
    tree_name = params["tree_name"]
    input_data = params["input_data"]
    context = params["context"] || %{}
    
    # Load decision tree definition
    case load_decision_tree(tree_name) do
      {:ok, tree} ->
        # Process the decision tree
        result = process_tree(tree, input_data, context, [])
        
        # Store decision history in memory
        MemoryService.store_memory(call_id, %{
          content: "Decision tree '#{tree_name}' processed with outcome: #{result.final_outcome}",
          memory_type: "decision_history",
          importance: 3,
          tags: ["decision_tree", tree_name],
          metadata: result
        })
        
        result
        
      {:error, :not_found} ->
        %{success: false, error: "Decision tree '#{tree_name}' not found"}
    end
  end
  
  defp process_tree(tree, data, context, path) do
    current_node = tree.root
    process_node(tree, current_node, data, context, path)
  end
  
  defp process_node(tree, node, data, context, path) do
    case node.type do
      :condition ->
        # Evaluate condition
        result = evaluate_condition(node.condition, data, context)
        next_node_id = if result, do: node.true_branch, else: node.false_branch
        next_node = get_node(tree, next_node_id)
        
        new_path = path ++ [node.description]
        process_node(tree, next_node, data, context, new_path)
        
      :outcome ->
        # Final outcome reached
        %{
          success: true,
          decision_path: path ++ [node.description],
          final_outcome: node.outcome,
          confidence_score: calculate_confidence(path, data),
          recommended_actions: node.actions || [],
          escalation_required: node.escalate || false
        }
        
      :action ->
        # Execute action and continue
        execute_decision_action(node.action, data, context)
        next_node = get_node(tree, node.next)
        new_path = path ++ [node.description]
        process_node(tree, next_node, data, context, new_path)
    end
  end
end
```

## Custom Tool Development

Organizations can develop custom tools by implementing the `Aybiza.Tools.Base` behaviour and registering them with the Tool Registry:

```elixir
defmodule MyCompany.Tools.CheckInventory do
  @moduledoc """
  Custom tool for checking product inventory
  """
  
  use Aybiza.Tools.Base
  
  @impl true
  def definition do
    %{
      name: "check_inventory",
      description: "Checks inventory levels for products",
      parameters: %{
        type: "object",
        properties: %{
          product_id: %{
            type: "string",
            description: "Product ID to check"
          },
          location: %{
            type: "string",
            description: "Optional location to check"
          }
        },
        required: ["product_id"]
      },
      returns: %{
        type: "object",
        properties: %{
          available: %{
            type: "boolean",
            description: "Whether the product is available"
          },
          quantity: %{
            type: "integer",
            description: "Available quantity"
          },
          estimated_restock: %{
            type: "string",
            description: "When out-of-stock items will be restocked"
          }
        }
      }
    }
  end
  
  @impl true
  def execute(call_id, params) do
    # Implementation to check inventory in your system
    %{
      available: true,
      quantity: 42,
      estimated_restock: nil
    }
  end
end

# Register the tool
Aybiza.ToolRegistry.register_tool(MyCompany.Tools.CheckInventory.definition())
```

## Tool Composition and Architecture Benefits

### Advanced Tool Composition

The enhanced Tool Registry supports complex business processes through intelligent tool composition:

```elixir
defmodule Aybiza.Tools.CompositeWorkflow do
  @moduledoc """
  Example of tool composition for complex business processes
  """
  
  use Aybiza.Tools.Base
  
  def execute_customer_lifecycle(call_id, customer_data) do
    # Step 1: Validate customer data
    validation_result = ToolRegistry.execute_tool(call_id, "validate_customer_data", customer_data)
    
    if not validation_result.valid do
      return collect_missing_information(call_id, validation_result.missing_fields)
    end
    
    # Step 2: Check for existing customer
    search_result = ToolRegistry.execute_tool(call_id, "search_customer", %{
      "email" => customer_data["email"],
      "phone" => customer_data["phone"]
    })
    
    # Step 3: Branch logic based on existing customer
    if search_result.found do
      # Existing customer - update and upsell
      update_existing_customer(call_id, search_result.customer_id, customer_data)
    else
      # New customer - full onboarding
      onboard_new_customer(call_id, customer_data)
    end
  end
  
  defp onboard_new_customer(call_id, customer_data) do
    # Determine onboarding type based on customer profile
    onboarding_type = determine_onboarding_type(customer_data)
    
    # Execute complex onboarding workflow
    ToolRegistry.execute_tool(call_id, "process_customer_onboarding", %{
      "customer_data" => customer_data,
      "onboarding_type" => onboarding_type
    })
  end
  
  defp update_existing_customer(call_id, customer_id, new_data) do
    # Update customer record
    update_result = ToolRegistry.execute_tool(call_id, "salesforce_integration", %{
      "action" => "update_contact",
      "customer_id" => customer_id,
      "data" => new_data
    })
    
    # Check for upsell opportunities
    if should_present_upsell?(customer_id) do
      ToolRegistry.execute_tool(call_id, "present_upsell_options", %{
        "customer_id" => customer_id
      })
    end
    
    update_result
  end
end
```

### Architecture Advantages

**1. Voice-First Optimization**
- Direct Claude integration eliminates latency
- Streaming responses during tool execution
- Real-time decision making without workflow overhead

**2. Simplified Development**
- Single interface for all business logic
- No visual designer complexity
- Code-based tools are version controlled and testable

**3. Flexible Composition**
- Tools can call other tools for complex workflows
- Conditional logic handled within tools
- Dynamic decision making based on call context

**4. Enterprise Integration**
- Direct external system integration
- Comprehensive error handling and retry logic
- Security and audit logging built-in

**5. Scalability**
- Tools execute independently
- Horizontal scaling across multiple instances
- Efficient resource utilization

### Comparison: Enhanced Tool Registry vs. Separate Automation Engine

| Aspect | Enhanced Tool Registry | Separate Automation Engine |
|--------|----------------------|---------------------------|
| **Voice Latency** | ✅ Direct execution (< 100ms) | ❌ Workflow overhead (> 500ms) |
| **Development** | ✅ Code-based, version controlled | ❌ Visual designer dependency |
| **Complexity** | ✅ Single system to maintain | ❌ Multiple systems integration |
| **Flexibility** | ✅ Claude can make dynamic decisions | ❌ Pre-defined workflow paths |
| **Testing** | ✅ Standard Elixir testing | ❌ Workflow testing complexity |
| **Debugging** | ✅ Standard debugging tools | ❌ Visual workflow debugging |
| **Performance** | ✅ Optimized for voice calls | ❌ General purpose workflows |

## Analytics

The Tool Registry provides comprehensive analytics:

1. **Usage Metrics**: How often each tool is used
2. **Success Rates**: Success vs. failure rates for tools
3. **Performance**: Average execution time per tool
4. **Patterns**: Common tool sequences and combinations
5. **Agent Patterns**: Tool usage patterns by agent

## Versioning

Tools support versioning for backward compatibility:

```elixir
defmodule Aybiza.Tools.TransferCallV2 do
  @moduledoc """
  Updated version of the transfer_call tool
  """
  
  use Aybiza.Tools.Base
  
  @impl true
  def definition do
    %{
      name: "transfer_call",
      version: 2,
      description: "Transfers the current call with enhanced features",
      # Updated schema...
    }
  end
  
  @impl true
  def execute(call_id, params) do
    # Updated implementation...
  end
end
```

## Security Considerations

1. **Authentication**: Tools can require authentication
2. **Authorization**: Tool access based on agent permissions
3. **Audit Logging**: All tool executions are logged
4. **Parameter Validation**: Strict validation of input parameters
5. **Timeout Controls**: Tools have execution timeouts
6. **Error Handling**: Graceful error handling for all tools