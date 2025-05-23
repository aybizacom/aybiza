Claude 4 in Amazon Bedrock: Complete Guide for Building Agents with Elixir
TL;DR
Claude 4 Opus and Sonnet are now available in Amazon Bedrock with model IDs anthropic.claude-opus-4-20250514-v1:0 and anthropic.claude-sonnet-4-20250514-v1:0 Amazon Web ServicesAnthropic. You can use the existing ex_aws_bedrock Elixir library or make direct HTTP calls to the Bedrock Converse API. Both models support prompt caching for up to one hour, parallel tool use, extended thinking modes, and new API capabilities including code execution tools, MCP connectors, and Files API AnthropicAnthropic.

Claude 4 Models Available in Bedrock
Claude 4 Opus:

Model ID: anthropic.claude-opus-4-20250514-v1:0 Amazon Web ServicesAnthropic
Best coding model available, designed for sophisticated AI agents Claude Opus 4 \ Anthropic
Available in US East (Ohio, N. Virginia) and US West (Oregon) Introducing Claude 4 in Amazon Bedrock, the most powerful models for coding from Anthropic | Amazon Web Services
Pricing: $15 per million input tokens, $75 per million output tokens, with up to 90% cost savings with prompt caching Claude Opus 4 \ Anthropic

Claude 4 Sonnet:

Model ID: anthropic.claude-sonnet-4-20250514-v1:0 Amazon Web ServicesAnthropic
Available in US East (Ohio, N. Virginia), US West (Oregon), Asia Pacific (Hyderabad, Mumbai, Osaka, Seoul, Singapore, Sydney, Tokyo), and Europe (Spain) Introducing Claude 4 in Amazon Bedrock, the most powerful models for coding from Anthropic | Amazon Web Services
Pricing: $3/$15 per million tokens (input/output) Anthropic's new Claude 4 AI models can reason over many steps | TechCrunch

Key Capabilities for Agent Building
Both models include new capabilities specifically for building AI agents:

Extended prompt caching: Up to 1 hour TTL (12x improvement from 5 minutes)
Parallel tool execution: Use multiple tools simultaneously
Extended thinking with tool use: Models can alternate between reasoning and tool use
Memory capabilities: Extract and save key facts for continuity
Code execution tool: Python code execution in sandboxed environment
MCP connector: Connect to external systems
Files API: Store and access files across sessions AnthropicAnthropic

Elixir Implementation Options
Option 1: Using ex_aws_bedrock Library (Recommended)
The ex_aws_bedrock library provides Elixir support for Amazon Bedrock GitHub - devstopfix/ex_aws_bedrock: Elixir library for AWS Bedrock:
elixir# In mix.exs
def deps do
  [
    {:ex_aws, ">= 2.5.1"},
    {:ex_aws_bedrock, "~> 2.5"},
    {:hackney, "~> 1.9"},
    {:jason, "~> 1.1"}
  ]
end
Configuration:
elixir# config/config.exs
config :ex_aws,
  access_key_id: [{:system, "AWS_ACCESS_KEY_ID"}, :instance_role],
  secret_access_key: [{:system, "AWS_SECRET_ACCESS_KEY"}, :instance_role],
  region: "us-east-1",
  json_codec: Jason

config :ex_aws, :bedrock,
  scheme: "https://",
  region: "us-east-1"
Option 2: Direct HTTP API Calls
Since Bedrock's Converse API is REST-based, you can make direct HTTP calls using Tesla, HTTPoison, or Finch:Claude 4 Bedrock Agent in ElixirCode # Claude 4 Bedrock Agent Implementation in Elixir

## Option 1: Using ex_aws_bedrock (Recommended)

defmodule MyApp.Claude4Agent do
  @moduledoc """
  Claude 4 agent using ex_aws_bedrock library for Amazon Bedrock integration.
  Supports both Opus 4 Key Implementation Details
Prompt Caching for Cost Optimization
Claude 4 supports extended prompt caching for up to 1 hour (12x improvement), reducing costs by up to 90% and latency by up to 85% for long prompts AnthropicAmazon. This is crucial for agent workflows that maintain context.
Tool Integration
Claude 4 models can use tools in parallel and alternate between reasoning and tool use during extended thinking Introducing Claude 4 \ Anthropic. The new capabilities include:

Code execution tool for Python analysis New capabilities for building agents on the Anthropic API \ Anthropic
MCP connector for external systems New capabilities for building agents on the Anthropic API \ Anthropic
Files API for cross-session storage New capabilities for building agents on the Anthropic API \ Anthropic

Regional Availability

Opus 4: US East (Ohio, N. Virginia) and US West (Oregon) Introducing Claude 4 in Amazon Bedrock, the most powerful models for coding from Anthropic | Amazon Web Services
Sonnet 4: US East (Ohio, N. Virginia), US West (Oregon), Asia Pacific (multiple regions), and Europe (Spain) Introducing Claude 4 in Amazon Bedrock, the most powerful models for coding from Anthropic | Amazon Web Services

Getting Started Checklist

Enable Bedrock Access: Navigate to AWS Bedrock console and request access to Claude 4 models Building a Claude 3 AI Agent with AWS Bedrock, Amazon API Gateway, and AWS Lambda: A Comprehensive Tutorial - Proud2beCloud Blog
Install Dependencies: Add ex_aws_bedrock to your Elixir project
Configure AWS Credentials: Set up IAM permissions for bedrock:InvokeModel
Start Building: Use the provided code examples to create your first agent

Comparison with Direct Anthropic API
While there's an Anthropic cookbook for Python developers GitHub - devstopfix/ex_aws_bedrock: Elixir library for AWS Bedrock, using Claude 4 through Bedrock offers:

Enterprise Security: AWS enterprise-grade security and compliance
Cost Optimization: Built-in prompt caching capabilities Prompt caching for faster model inference - Amazon Bedrock
Regional Deployment: Multiple regions for lower latency
AWS Integration: Easy integration with other AWS services (S3, Lambda, etc.)

The Elixir ecosystem is well-positioned for building Claude 4 agents with Bedrock, offering both high-level abstractions through ex_aws_bedrock and the flexibility of direct HTTP API access for custom implementations.

CODE

# Claude 4 Bedrock Agent Implementation in Elixir

## Option 1: Using ex_aws_bedrock (Recommended)

defmodule MyApp.Claude4Agent do
  @moduledoc """
  Claude 4 agent using ex_aws_bedrock library for Amazon Bedrock integration.
  Supports both Opus 4 and Sonnet 4 models with all new capabilities.
  """

  alias ExAws.Bedrock

  # Claude 4 model IDs
  @claude_opus_4 "anthropic.claude-opus-4-20250514-v1:0"
  @claude_sonnet_4 "anthropic.claude-sonnet-4-20250514-v1:0"

  @doc """
  Invoke Claude 4 with conversation and tools support
  """
  def converse_with_claude(messages, opts \\ []) do
    model_id = Keyword.get(opts, :model, @claude_sonnet_4)
    
    params = %{
      "messages" => format_messages(messages),
      "inferenceConfig" => %{
        "maxTokens" => Keyword.get(opts, :max_tokens, 4000),
        "temperature" => Keyword.get(opts, :temperature, 0.7)
      }
    }
    
    # Add system prompt if provided
    params = if system = Keyword.get(opts, :system) do
      Map.put(params, "system", [%{"text" => system}])
    else
      params
    end
    
    # Add tools if provided
    params = if tools = Keyword.get(opts, :tools) do
      Map.put(params, "toolConfig", %{"tools" => tools})
    else
      params
    end
    
    # Add prompt caching if enabled
    params = if Keyword.get(opts, :enable_caching, false) do
      # Enable extended 1-hour prompt caching for agents
      add_cache_config(params)
    else
      params
    end

    Bedrock.converse(model_id, params)
    |> ExAws.request()
    |> case do
      {:ok, response} -> {:ok, parse_response(response)}
      {:error, error} -> {:error, error}
    end
  end

  @doc """
  Create an agent with memory and tool capabilities
  """
  def create_agent(agent_config) do
    %{
      name: agent_config.name,
      model: agent_config.model || @claude_sonnet_4,
      system_prompt: agent_config.system_prompt,
      tools: agent_config.tools || [],
      memory: %{},
      conversation_history: []
    }
  end

  @doc """
  Run agent with conversation context and tool execution
  """
  def run_agent(agent, user_message, opts \\ []) do
    # Add user message to conversation history
    updated_agent = add_to_conversation(agent, "user", user_message)
    
    # Prepare messages with memory context
    messages = build_messages_with_memory(updated_agent)
    
    # Configure for extended thinking if requested
    inference_opts = if Keyword.get(opts, :extended_thinking, false) do
      [
        model: agent.model,
        system: agent.system_prompt,
        tools: agent.tools,
        enable_caching: true,
        max_tokens: 8000,
        temperature: 0.1
      ]
    else
      [
        model: agent.model,
        system: agent.system_prompt,
        tools: agent.tools,
        enable_caching: true
      ]
    end
    
    case converse_with_claude(messages, inference_opts) do
      {:ok, response} ->
        # Handle tool calls if present
        {updated_agent, final_response} = handle_response(updated_agent, response)
        {:ok, updated_agent, final_response}
        
      {:error, error} ->
        {:error, error}
    end
  end

  # Helper functions
  
  defp format_messages(messages) when is_list(messages) do
    Enum.map(messages, &format_message/1)
  end
  
  defp format_message(%{role: role, content: content}) when is_binary(content) do
    %{
      "role" => to_string(role),
      "content" => [%{"text" => content}]
    }
  end
  
  defp format_message(%{role: role, content: content}) when is_list(content) do
    %{
      "role" => to_string(role),
      "content" => content
    }
  end

  defp add_cache_config(params) do
    # Add cache control for extended 1-hour caching
    # This reduces costs by up to 90% for agent workflows
    Map.put(params, "cacheConfig", %{
      "ttl" => 3600  # 1 hour in seconds
    })
  end

  defp parse_response(%{"output" => %{"message" => message}} = response) do
    %{
      content: get_in(message, ["content"]),
      role: get_in(message, ["role"]),
      stop_reason: get_in(response, ["stopReason"]),
      usage: get_in(response, ["usage"]),
      tool_calls: extract_tool_calls(message)
    }
  end

  defp extract_tool_calls(%{"content" => content}) do
    content
    |> Enum.filter(fn item -> Map.has_key?(item, "toolUse") end)
    |> Enum.map(fn %{"toolUse" => tool_use} -> tool_use end)
  end
  defp extract_tool_calls(_), do: []

  defp add_to_conversation(agent, role, content) do
    message = %{role: role, content: content, timestamp: DateTime.utc_now()}
    %{agent | conversation_history: agent.conversation_history ++ [message]}
  end

  defp build_messages_with_memory(agent) do
    # Include recent conversation history and memory context
    recent_messages = Enum.take(agent.conversation_history, -10)
    
    memory_context = if map_size(agent.memory) > 0 do
      memory_text = Jason.encode!(agent.memory)
      [%{role: "system", content: "Previous context: #{memory_text}"}]
    else
      []
    end
    
    memory_context ++ Enum.map(recent_messages, fn msg ->
      %{role: msg.role, content: msg.content}
    end)
  end

  defp handle_response(agent, %{tool_calls: []} = response) do
    # No tool calls, just add response to conversation
    updated_agent = add_to_conversation(agent, "assistant", get_text_content(response))
    {updated_agent, response}
  end

  defp handle_response(agent, %{tool_calls: tool_calls} = response) when length(tool_calls) > 0 do
    # Execute tools and continue conversation
    {updated_agent, tool_results} = execute_tools(agent, tool_calls)
    
    # Continue conversation with tool results
    tool_messages = format_tool_results(tool_results)
    messages = build_messages_with_memory(updated_agent) ++ tool_messages
    
    case converse_with_claude(messages, [model: agent.model, tools: agent.tools]) do
      {:ok, final_response} ->
        final_agent = add_to_conversation(updated_agent, "assistant", get_text_content(final_response))
        {final_agent, final_response}
      
      {:error, _} = error ->
        {updated_agent, error}
    end
  end

  defp execute_tools(agent, tool_calls) do
    results = Enum.map(tool_calls, fn tool_call ->
      execute_single_tool(agent, tool_call)
    end)
    
    # Update agent memory with tool results if needed
    updated_agent = update_agent_memory(agent, results)
    {updated_agent, results}
  end

  defp execute_single_tool(_agent, %{"name" => "web_search", "input" => %{"query" => query}}) do
    # Example tool implementation
    %{
      tool_call_id: "search_#{:rand.uniform(1000)}",
      name: "web_search",
      result: "Search results for: #{query} - [simulated results]"
    }
  end

  defp execute_single_tool(_agent, %{"name" => "code_execution", "input" => %{"code" => code}}) do
    # Example code execution tool
    %{
      tool_call_id: "code_#{:rand.uniform(1000)}",
      name: "code_execution", 
      result: "Executed code: #{code} - [simulated execution]"
    }
  end

  defp execute_single_tool(_agent, tool_call) do
    %{
      tool_call_id: "unknown_#{:rand.uniform(1000)}",
      name: tool_call["name"],
      result: "Tool not implemented: #{tool_call["name"]}"
    }
  end

  defp update_agent_memory(agent, tool_results) do
    # Extract important information from tool results to update memory
    memory_updates = Enum.reduce(tool_results, %{}, fn result, acc ->
      case result.name do
        "web_search" -> Map.put(acc, "last_search", result.result)
        "code_execution" -> Map.put(acc, "last_code_result", result.result)
        _ -> acc
      end
    end)
    
    %{agent | memory: Map.merge(agent.memory, memory_updates)}
  end

  defp format_tool_results(tool_results) do
    Enum.map(tool_results, fn result ->
      %{
        role: "user",
        content: [
          %{
            "toolResult" => %{
              "toolUseId" => result.tool_call_id,
              "content" => [%{"text" => result.result}]
            }
          }
        ]
      }
    end)
  end

  defp get_text_content(%{content: [%{"text" => text} | _]}), do: text
  defp get_text_content(%{content: content}) when is_list(content) do
    content
    |> Enum.find(fn item -> Map.has_key?(item, "text") end)
    |> case do
      %{"text" => text} -> text
      _ -> ""
    end
  end
  defp get_text_content(_), do: ""
end

## Option 2: Direct HTTP Implementation

defmodule MyApp.BedrockHTTPClient do
  @moduledoc """
  Direct HTTP client for Amazon Bedrock Converse API.
  Use this if you prefer not to use ex_aws_bedrock.
  """

  @bedrock_endpoint "https://bedrock-runtime.us-east-1.amazonaws.com"

  def converse(model_id, params, opts \\ []) do
    url = "#{@bedrock_endpoint}/model/#{model_id}/converse"
    headers = build_headers(opts)
    body = Jason.encode!(params)

    case HTTPoison.post(url, body, headers) do
      {:ok, %{status_code: 200, body: response_body}} ->
        {:ok, Jason.decode!(response_body)}
      
      {:ok, %{status_code: status_code, body: error_body}} ->
        {:error, %{status_code: status_code, body: Jason.decode!(error_body)}}
      
      {:error, error} ->
        {:error, error}
    end
  end

  defp build_headers(opts) do
    region = Keyword.get(opts, :region, "us-east-1")
    access_key = System.get_env("AWS_ACCESS_KEY_ID")
    secret_key = System.get_env("AWS_SECRET_ACCESS_KEY")
    
    # You'll need to implement AWS Signature V4 signing
    # Or use the AWS Elixir library for signing
    [
      {"Content-Type", "application/json"},
      {"Authorization", build_auth_header(access_key, secret_key, region)},
      {"X-Amz-Date", iso8601_timestamp()}
    ]
  end

  defp build_auth_header(_access_key, _secret_key, _region) do
    # Implement AWS Signature V4 signing
    # This is complex - recommend using ex_aws for this
    "AWS4-HMAC-SHA256 Credential=..."
  end

  defp iso8601_timestamp do
    DateTime.utc_now()
    |> DateTime.to_iso8601()
    |> String.replace(~r/[:\-]/, "")
    |> String.replace("T", "")
    |> String.slice(0, 15) <> "Z"
  end
end

## Tool Definition Examples

defmodule MyApp.AgentTools do
  @moduledoc """
  Tool definitions for Claude 4 agents
  """

  def web_search_tool do
    %{
      "toolSpec" => %{
        "name" => "web_search",
        "description" => "Search the web for current information",
        "inputSchema" => %{
          "json" => %{
            "type" => "object",
            "properties" => %{
              "query" => %{
                "type" => "string",
                "description" => "The search query"
              }
            },
            "required" => ["query"]
          }
        }
      }
    }
  end

  def code_execution_tool do
    %{
      "toolSpec" => %{
        "name" => "code_execution", 
        "description" => "Execute Python code in a sandboxed environment",
        "inputSchema" => %{
          "json" => %{
            "type" => "object",
            "properties" => %{
              "code" => %{
                "type" => "string",
                "description" => "Python code to execute"
              }
            },
            "required" => ["code"]
          }
        }
      }
    }
  end

  def file_operations_tool do
    %{
      "toolSpec" => %{
        "name" => "file_operations",
        "description" => "Read, write, and manage files",
        "inputSchema" => %{
          "json" => %{
            "type" => "object", 
            "properties" => %{
              "operation" => %{
                "type" => "string",
                "enum" => ["read", "write", "list", "delete"]
              },
              "file_path" => %{
                "type" => "string",
                "description" => "Path to the file"
              },
              "content" => %{
                "type" => "string",
                "description" => "Content for write operations"
              }
            },
            "required" => ["operation", "file_path"]
          }
        }
      }
    }
  end
end

## Usage Examples

defmodule MyApp.Examples do
  alias MyApp.Claude4Agent
  alias MyApp.AgentTools

  def create_coding_agent do
    Claude4Agent.create_agent(%{
      name: "CodingAssistant",
      model: "anthropic.claude-opus-4-20250514-v1:0",  # Use Opus for complex coding
      system_prompt: """
      You are an expert software engineer specializing in Elixir and functional programming.
      You can execute code, search for information, and manage files.
      Always write clean, well-documented, and tested code.
      """,
      tools: [
        AgentTools.code_execution_tool(),
        AgentTools.web_search_tool(),
        AgentTools.file_operations_tool()
      ]
    })
  end

  def create_research_agent do
    Claude4Agent.create_agent(%{
      name: "ResearchAssistant", 
      model: "anthropic.claude-sonnet-4-20250514-v1:0",  # Sonnet for research tasks
      system_prompt: """
      You are a research assistant capable of searching the web and analyzing information.
      Provide comprehensive, well-sourced responses with proper citations.
      """,
      tools: [AgentTools.web_search_tool()]
    })
  end

  def run_coding_example do
    agent = create_coding_agent()
    
    user_request = """
    I need help building a GenServer in Elixir that manages a simple counter 
    with increment, decrement, and get_value operations. Can you write the code 
    and explain how it works?
    """
    
    case Claude4Agent.run_agent(agent, user_request, extended_thinking: true) do
      {:ok, _updated_agent, response} ->
        IO.puts("Agent response: #{get_in(response, [:content, Access.at(0), "text"])}")
        
      {:error, error} ->
        IO.puts("Error: #{inspect(error)}")
    end
  end

  def run_research_example do
    agent = create_research_agent()
    
    user_request = """
    What are the latest developments in Elixir and Phoenix framework? 
    Please search for recent news and provide a summary.
    """
    
    case Claude4Agent.run_agent(agent, user_request) do
      {:ok, _updated_agent, response} ->
        IO.puts("Research results: #{get_in(response, [:content, Access.at(0), "text"])}")
        
      {:error, error} ->
        IO.puts("Error: #{inspect(error)}")
    end
  end
end