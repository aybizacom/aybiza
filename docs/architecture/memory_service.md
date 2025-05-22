# AYBIZA Memory Service

## Overview

The Memory Service is a core component of the AYBIZA AI Voice Business Agent platform that provides persistent memory capabilities, allowing agents to remember and recall information across sessions. This document outlines the architecture and implementation of the Memory Service.

## Architecture

### Key Components

1. **Vector Database**: Stores conversation and entity embeddings for similarity-based retrieval
2. **Context Manager**: Manages the retrieval and integration of relevant context
3. **Entity Recognizer**: Identifies and stores key entities from conversations
4. **Identity Manager**: Associates conversations with users based on identifiers (phone number, etc.)
5. **Memory Summarizer**: Creates condensed summaries of past interactions
6. **Persistence Service**: Ensures memory is retained across sessions

### Data Flow

```
Current Conversation → Entity Recognition → Vector Embedding → 
  Storage in Vector DB → Retrieval → Summarization → 
    Context Integration → Enhanced LLM Context
```

## Implementation

### Vector Database

```elixir
defmodule Aybiza.MemoryService.VectorDB do
  @moduledoc """
  Handles storage and retrieval of vector embeddings for memory
  """
  
  use GenServer
  
  alias Aybiza.AWS.BedrockEmbeddings
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def store_memory(user_id, content, metadata \\ %{}) do
    GenServer.call(__MODULE__, {:store_memory, user_id, content, metadata})
  end
  
  def retrieve_relevant(user_id, query, opts \\ %{}) do
    GenServer.call(__MODULE__, {:retrieve_relevant, user_id, query, opts})
  end
  
  @impl true
  def init(opts) do
    {:ok, %{index_options: opts[:index_options] || %{}}}
  end
  
  @impl true
  def handle_call({:store_memory, user_id, content, metadata}, _from, state) do
    # Generate embedding from content
    {:ok, embedding} = BedrockEmbeddings.generate_embedding(content)
    
    # Store in database with metadata
    memory_id = "mem_#{System.unique_integer([:positive])}"
    item = %{
      id: memory_id,
      user_id: user_id,
      content: content,
      embedding: embedding,
      metadata: Map.merge(metadata, %{
        created_at: DateTime.utc_now(),
        source: metadata[:source] || "conversation"
      })
    }
    
    # Store in database
    {:ok, _} = store_vector(item)
    
    {:reply, {:ok, memory_id}, state}
  end
  
  @impl true
  def handle_call({:retrieve_relevant, user_id, query, opts}, _from, state) do
    # Generate embedding from query
    {:ok, query_embedding} = BedrockEmbeddings.generate_embedding(query)
    
    # Set defaults
    opts = Map.merge(%{
      limit: 5,
      threshold: 0.7,
      time_frame: nil
    }, opts)
    
    # Search for similar vectors
    results = search_vectors(%{
      user_id: user_id,
      embedding: query_embedding,
      limit: opts.limit,
      threshold: opts.threshold,
      time_frame: opts.time_frame
    })
    
    {:reply, {:ok, results}, state}
  end
  
  defp store_vector(item) do
    # Store in vector database (PostgreSQL with pgvector or dedicated vector DB)
    # Implementation depends on chosen vector store
    {:ok, item.id}
  end
  
  defp search_vectors(params) do
    # Search for similar vectors in database
    # Returns list of matches with similarity scores
    []
  end
end
```

### Context Manager

```elixir
defmodule Aybiza.MemoryService.ContextManager do
  @moduledoc """
  Manages conversation context with memory integration
  """
  
  alias Aybiza.MemoryService.VectorDB
  alias Aybiza.MemoryService.Summarizer
  
  @doc """
  Enhances a prompt with relevant memory
  """
  def enhance_with_memory(call_id, prompt, user_id) do
    # Retrieve relevant memories
    {:ok, memories} = VectorDB.retrieve_relevant(user_id, prompt)
    
    # If no memories found, return original prompt
    if Enum.empty?(memories) do
      prompt
    else
      # Summarize memories if they're too long
      summary = if total_tokens(memories) > 1000 do
        Summarizer.summarize_memories(memories)
      else
        format_memories(memories)
      end
      
      # Create enhanced prompt with memory
      """
      The caller has previously interacted with our system. Here is relevant context:
      
      #{summary}
      
      Current conversation:
      #{prompt}
      """
    end
  end
  
  @doc """
  Stores important information from current conversation
  """
  def store_conversation_memory(call_id, transcript, user_id) do
    # Extract key information from transcript
    key_points = extract_key_points(transcript)
    
    # Store each key point as separate memory
    Enum.each(key_points, fn point ->
      VectorDB.store_memory(user_id, point, %{
        call_id: call_id,
        source: "conversation"
      })
    end)
    
    :ok
  end
  
  defp extract_key_points(transcript) do
    # Extract key information from transcript
    # Implementation uses LLM to identify important points
    []
  end
  
  defp format_memories(memories) do
    memories
    |> Enum.map(fn memory ->
      date = format_date(memory.metadata.created_at)
      "- #{date}: #{memory.content}"
    end)
    |> Enum.join("\n")
  end
  
  defp total_tokens(memories) do
    # Estimate token count for all memories
    Enum.reduce(memories, 0, fn memory, acc ->
      acc + String.length(memory.content) ÷ 4
    end)
  end
  
  defp format_date(datetime) do
    # Format datetime to human-readable string
    Calendar.strftime(datetime, "%B %d, %Y")
  end
end
```

### Entity Recognizer

```elixir
defmodule Aybiza.MemoryService.EntityRecognizer do
  @moduledoc """
  Identifies and extracts entities from conversations
  """
  
  alias Aybiza.AWS.BedrockClient
  alias Aybiza.MemoryService.VectorDB
  
  @doc """
  Extract entities from transcript
  """
  def extract_entities(transcript, call_id, user_id) do
    # Prepare LLM prompt for entity extraction
    prompt = """
    Extract important entities from this conversation transcript. 
    Focus on:
    - Personal information (name, preferences)
    - Product interests
    - Issues mentioned
    - Requests made
    - Status of interactions
    
    Transcript:
    #{transcript}
    
    Return a JSON array of entities with type and value.
    """
    
    # Call Bedrock Claude to extract entities
    {:ok, response} = BedrockClient.invoke_model("anthropic.claude-3-haiku-20240307-v1:0", %{
      "messages" => [
        %{
          "role" => "user",
          "content" => prompt
        }
      ],
      "max_tokens" => 1000,
      "temperature" => 0
    })
    
    # Parse entities from response
    case parse_entities_from_response(response.content) do
      {:ok, entities} ->
        # Store each entity in vector DB
        store_entities(entities, call_id, user_id)
        {:ok, entities}
        
      error ->
        error
    end
  end
  
  defp parse_entities_from_response(content) do
    # Extract JSON from response
    case Regex.run(~r/\[.*\]/s, content) do
      [json_str] ->
        case Jason.decode(json_str) do
          {:ok, entities} -> {:ok, entities}
          error -> error
        end
      
      nil ->
        {:error, :no_entities_found}
    end
  end
  
  defp store_entities(entities, call_id, user_id) do
    Enum.each(entities, fn entity ->
      VectorDB.store_memory(user_id, entity["value"], %{
        call_id: call_id,
        entity_type: entity["type"],
        source: "entity_recognition"
      })
    end)
  end
end
```

### Identity Manager

```elixir
defmodule Aybiza.MemoryService.IdentityManager do
  @moduledoc """
  Manages user identity for memory persistence
  """
  
  alias Aybiza.AgentManager.Repo
  
  @doc """
  Resolve user ID from phone number
  """
  def resolve_user_id(phone_number) do
    # Check if we have an existing user ID for this phone
    case get_user_id_by_phone(phone_number) do
      {:ok, user_id} ->
        {:ok, user_id}
        
      {:error, :not_found} ->
        # Create new anonymous user
        create_anonymous_user(phone_number)
    end
  end
  
  @doc """
  Link anonymous user to customer record
  """
  def link_user_to_customer(user_id, customer_id) do
    # Link user ID to customer record
    # Implementation depends on database schema
    :ok
  end
  
  @doc """
  Merge two user identities
  """
  def merge_user_identities(primary_user_id, secondary_user_id) do
    # Transfer all memories from secondary to primary
    # Implementation depends on database schema
    :ok
  end
  
  defp get_user_id_by_phone(phone_number) do
    # Query database for existing user ID
    # Implementation depends on database schema
    {:error, :not_found}
  end
  
  defp create_anonymous_user(phone_number) do
    # Create new anonymous user record
    user_id = "user_#{System.unique_integer([:positive])}"
    
    # Store user ID with phone number
    # Implementation depends on database schema
    
    {:ok, user_id}
  end
end
```

### Memory Summarizer

```elixir
defmodule Aybiza.MemoryService.Summarizer do
  @moduledoc """
  Summarizes memories for more efficient context
  """
  
  alias Aybiza.AWS.BedrockClient
  
  @doc """
  Summarize a list of memories
  """
  def summarize_memories(memories) do
    # Format memories for summarization
    memories_text = memories
    |> Enum.map(fn memory ->
      date = format_date(memory.metadata.created_at)
      "#{date}: #{memory.content}"
    end)
    |> Enum.join("\n\n")
    
    # Prepare LLM prompt for summarization
    prompt = """
    Summarize these past interactions with a customer in a concise way that
    captures the most important information. Focus on preferences, issues,
    requests, and status of interactions.
    
    Previous interactions:
    #{memories_text}
    
    Create a brief summary that could be used to quickly understand this customer's
    history with our service. Focus on what would be most relevant for continuing
    a conversation with them.
    """
    
    # Call Bedrock Claude for summarization
    {:ok, response} = BedrockClient.invoke_model("anthropic.claude-3-haiku-20240307-v1:0", %{
      "messages" => [
        %{
          "role" => "user",
          "content" => prompt
        }
      ],
      "max_tokens" => 1000,
      "temperature" => 0
    })
    
    response.content
  end
  
  defp format_date(datetime) do
    # Format datetime to human-readable string
    Calendar.strftime(datetime, "%B %d, %Y")
  end
end
```

## API

### Internal API

```elixir
defmodule Aybiza.MemoryService do
  @moduledoc """
  Memory service for AYBIZA AI Voice Business Agents
  """
  
  alias Aybiza.MemoryService.VectorDB
  alias Aybiza.MemoryService.ContextManager
  alias Aybiza.MemoryService.EntityRecognizer
  alias Aybiza.MemoryService.IdentityManager
  alias Aybiza.MemoryService.Summarizer
  
  @doc """
  Store memory for a user
  """
  def store_memory(call_id, params) do
    # Extract parameters
    content = params.content
    memory_type = params.memory_type
    importance = params.importance || 3
    expiration = params.expiration || "never"
    tags = params.tags || []
    
    # Get user ID from call ID
    {:ok, user_id} = get_user_id_from_call(call_id)
    
    # Store memory with metadata
    VectorDB.store_memory(user_id, content, %{
      call_id: call_id,
      memory_type: memory_type,
      importance: importance,
      expiration: expiration,
      tags: tags
    })
  end
  
  @doc """
  Retrieve memories based on query
  """
  def retrieve_memory(call_id, params) do
    # Extract parameters
    query = params.query
    memory_type = params.memory_type
    max_results = params.max_results || 5
    time_frame = params.time_frame || "all"
    caller_phone = params.caller_phone
    
    # Get user ID from call or phone
    {:ok, user_id} = if caller_phone do
      IdentityManager.resolve_user_id(caller_phone)
    else
      get_user_id_from_call(call_id)
    end
    
    # Retrieve memories
    {:ok, results} = VectorDB.retrieve_relevant(user_id, query, %{
      limit: max_results,
      memory_type: memory_type,
      time_frame: time_frame
    })
    
    # Format results
    %{
      memories: results,
      summary: Summarizer.summarize_memories(results)
    }
  end
  
  @doc """
  Process call transcript for entity extraction and memory storage
  """
  def process_transcript(call_id, transcript) do
    # Get user ID from call
    {:ok, user_id} = get_user_id_from_call(call_id)
    
    # Extract entities
    {:ok, entities} = EntityRecognizer.extract_entities(transcript, call_id, user_id)
    
    # Store conversation context
    ContextManager.store_conversation_memory(call_id, transcript, user_id)
    
    {:ok, entities}
  end
  
  @doc """
  Enhance prompt with memory
  """
  def enhance_prompt(call_id, prompt) do
    # Get user ID from call
    {:ok, user_id} = get_user_id_from_call(call_id)
    
    # Enhance prompt with memory
    ContextManager.enhance_with_memory(call_id, prompt, user_id)
  end
  
  defp get_user_id_from_call(call_id) do
    # Get call details
    {:ok, call} = Aybiza.CallAnalytics.get_call(call_id)
    
    # Resolve user ID from phone number
    IdentityManager.resolve_user_id(call.from_number)
  end
end
```

### Function Schema for External API

```elixir
%{
  name: "retrieve_memory",
  description: "Retrieves relevant information from the caller's past interactions. Use this to provide personalized service based on previous conversations.",
  parameters: %{
    type: "object",
    properties: %{
      caller_phone: %{
        type: "string",
        description: "The caller's phone number to search for memories (defaults to current caller)"
      },
      query: %{
        type: "string",
        description: "Query to search for in past conversations"
      },
      memory_type: %{
        type: "string",
        enum: ["conversations", "entities", "preferences"],
        description: "Type of memory to retrieve"
      },
      max_results: %{
        type: "integer",
        default: 5,
        description: "Maximum number of results to return"
      },
      time_frame: %{
        type: "string",
        enum: ["all", "recent", "last_week", "last_month"],
        default: "all",
        description: "Time frame for memory retrieval"
      }
    },
    required: ["query", "memory_type"]
  },
  returns: %{
    type: "object",
    properties: %{
      memories: %{
        type: "array",
        items: %{
          type: "object",
          properties: %{
            content: %{
              type: "string",
              description: "Content of the memory"
            },
            timestamp: %{
              type: "string",
              format: "date-time",
              description: "When the memory was created"
            },
            relevance_score: %{
              type: "number",
              description: "Relevance score (0-1) to the query"
            },
            source: %{
              type: "string",
              description: "Source of the memory (e.g., 'call', 'email')"
            }
          }
        },
        description: "Retrieved memories matching the query"
      },
      summary: %{
        type: "string",
        description: "A concise summary of the retrieved memories"
      }
    }
  }
}
```

## Benefits

1. **Personalized Interactions**: Agents remember caller preferences and history
2. **Reduced Repetition**: Callers don't need to repeat information from previous calls
3. **Improved Efficiency**: Agents can reference past issues and resolutions
4. **Enhanced Customer Experience**: Creates a sense of continuity across interactions
5. **Better Problem Solving**: Access to historical context helps resolve complex issues

## Security and Privacy

1. **Data Isolation**: Strict multi-tenant isolation prevents cross-customer data access
2. **Configurable Retention**: Memory retention periods configurable by organization
3. **Data Redaction**: PII detection and redaction in stored memories
4. **Access Controls**: Permission-based access to memory functions
5. **Audit Logging**: Comprehensive logging of all memory operations
6. **Compliance features**: GDPR, CCPA, and other regulatory compliance support with right-to-delete functionality

## Integration

The Memory Service integrates with:

1. **Agent Engine**: Enhances prompts with relevant memory before LLM calls
2. **Call Analytics**: Processes call transcripts for memory extraction
3. **Security**: Enforces security and privacy policies
4. **CRM Systems**: Links anonymous users to customer records when identified