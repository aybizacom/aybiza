# Future-Ready MCP Placeholder Architecture
> Architecture Document #008 - v1.0

## Overview

This document outlines a placeholder architecture for Model Context Protocol (MCP) integration that allows AYBIZA to prepare for MCP support without implementing custom solutions that would need to be replaced. The design ensures minimal refactoring when AWS Bedrock adds native MCP support.

## Current State vs Future State

### Current State (2025)
- AWS Bedrock does not yet support MCP servers
- HTTP/Webhook tools provide similar functionality
- Custom tool registry handles integrations

### Future State (When Available)
- Native MCP support in AWS Bedrock
- Direct MCP server connections
- Standardized tool interfaces
- Enhanced tool discovery and capabilities

## Placeholder Architecture Design

### 1. MCP-Compatible Interface Layer

```elixir
defmodule AYBIZA.MCP.Interface do
  @moduledoc """
  MCP-compatible interface that wraps current HTTP/Webhook tools.
  When MCP is available, only this module needs updating.
  """
  
  @behaviour AYBIZA.ToolInterface
  
  def list_tools do
    # Currently returns HTTP/Webhook tools
    # Future: Returns MCP server tools
    ToolRegistry.list_tools()
    |> Enum.map(&transform_to_mcp_format/1)
  end
  
  def execute_tool(tool_name, params) do
    # Currently delegates to HTTP/Webhook
    # Future: Delegates to MCP server
    ToolRegistry.execute(tool_name, params)
  end
  
  defp transform_to_mcp_format(tool) do
    %{
      name: tool.id,
      description: tool.description,
      inputSchema: tool.parameters,
      # MCP-compatible structure
    }
  end
end
```

### 2. Tool Definition Standards

```yaml
# Use MCP-compatible tool definitions now
tool_definition:
  name: "search_customer"
  description: "Search for customer information"
  inputSchema:
    type: "object"
    properties:
      query:
        type: "string"
        description: "Search query"
    required: ["query"]
  
  # Current implementation details
  _implementation:
    type: "http"
    endpoint: "https://api.aybiza.com/customers/search"
    method: "POST"
```

### 3. Migration Path Planning

```elixir
defmodule AYBIZA.MCP.MigrationPlan do
  @doc """
  Detects MCP availability and manages migration
  """
  def check_mcp_availability do
    case AWS.Bedrock.check_feature("mcp_support") do
      {:ok, true} -> 
        Logger.info("MCP support detected, enabling migration")
        :mcp_available
      _ -> 
        :use_http_tools
    end
  end
  
  def migrate_if_available do
    if mcp_available?() do
      start_gradual_migration()
    end
  end
end
```

## Implementation Guidelines

### 1. Tool Registry Abstraction

```elixir
defmodule AYBIZA.Tools.AbstractRegistry do
  @callback register_tool(tool_definition :: map()) :: {:ok, tool_id} | {:error, reason}
  @callback execute_tool(tool_id :: String.t(), params :: map()) :: {:ok, result} | {:error, reason}
  @callback list_tools() :: [map()]
end

# Current implementation
defmodule AYBIZA.Tools.HTTPRegistry do
  @behaviour AYBIZA.Tools.AbstractRegistry
  # HTTP/Webhook implementation
end

# Future implementation (placeholder)
defmodule AYBIZA.Tools.MCPRegistry do
  @behaviour AYBIZA.Tools.AbstractRegistry
  # Will implement MCP when available
  def register_tool(_tool), do: {:error, :not_implemented}
  def execute_tool(_id, _params), do: {:error, :not_implemented}
  def list_tools(), do: []
end
```

### 2. Configuration for Easy Switching

```elixir
# config/config.exs
config :aybiza, :tool_system,
  adapter: AYBIZA.Tools.HTTPRegistry, # Easy to switch to MCPRegistry
  mcp_check_interval: :timer.hours(24),
  auto_migrate: false
```

### 3. Tool Compatibility Layer

```elixir
defmodule AYBIZA.Tools.Compatibility do
  @doc """
  Ensures tools work with both HTTP and future MCP
  """
  def wrap_tool(tool_definition) do
    tool_definition
    |> add_mcp_metadata()
    |> validate_schema()
    |> prepare_for_future()
  end
  
  defp add_mcp_metadata(tool) do
    Map.merge(tool, %{
      "_mcp_compatible": true,
      "_version": "1.0",
      "_migration_ready": true
    })
  end
end
```

## Monitoring and Readiness

### 1. MCP Readiness Dashboard

```elixir
defmodule AYBIZA.MCPReadiness do
  def generate_report do
    %{
      total_tools: count_tools(),
      mcp_compatible: count_mcp_compatible_tools(),
      migration_effort: estimate_migration_effort(),
      blockers: identify_migration_blockers()
    }
  end
  
  def identify_migration_blockers do
    # Check for tools that need refactoring
    ToolRegistry.list_tools()
    |> Enum.filter(&requires_refactoring?/1)
    |> Enum.map(&describe_required_changes/1)
  end
end
```

### 2. Feature Detection

```elixir
defmodule AYBIZA.FeatureDetection do
  use GenServer
  
  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end
  
  def init(state) do
    schedule_check()
    {:ok, state}
  end
  
  def handle_info(:check_features, state) do
    features = detect_bedrock_features()
    
    if features[:mcp_enabled] && !state[:mcp_notified] do
      notify_mcp_availability()
    end
    
    schedule_check()
    {:noreply, Map.put(state, :features, features)}
  end
  
  defp detect_bedrock_features do
    # Check AWS Bedrock for new features
    %{
      mcp_enabled: check_mcp_support(),
      native_memory: check_memory_support(),
      web_search: check_search_support(),
      code_execution: check_code_execution_support()
    }
  end
end
```

## Best Practices for Future-Ready Development

### 1. Use Abstraction Layers
- Never directly call HTTP endpoints from business logic
- Always go through the tool registry abstraction
- Keep tool definitions in standardized format

### 2. Document MCP Intentions
```elixir
@moduledoc """
This module implements customer search functionality.

MCP_READY: Yes
MCP_BLOCKERS: None
MCP_NOTES: Will map directly to customer_search MCP tool
"""
```

### 3. Avoid MCP-Incompatible Patterns
- Don't use stateful HTTP sessions
- Avoid complex authentication flows
- Keep tool interfaces simple and functional

### 4. Test with MCP in Mind
```elixir
defmodule ToolTest do
  test "tool is MCP compatible" do
    tool = ToolRegistry.get_tool("search_customer")
    assert MCPCompatibility.validate(tool) == :ok
  end
end
```

## Migration Checklist

When MCP becomes available on AWS Bedrock:

- [ ] Enable MCP feature flag in configuration
- [ ] Run MCP compatibility report
- [ ] Update tool adapter to use MCPRegistry
- [ ] Deploy MCP servers (if using external)
- [ ] Test tool execution through MCP
- [ ] Monitor performance differences
- [ ] Gradually migrate tools
- [ ] Update documentation
- [ ] Remove HTTP fallbacks (after validation)

## Conclusion

This architecture ensures AYBIZA is ready for MCP without premature optimization. By using abstraction layers and MCP-compatible patterns now, the future migration will be straightforward and low-risk. The system continues to use proven HTTP/Webhook tools while maintaining readiness for the enhanced capabilities MCP will bring.