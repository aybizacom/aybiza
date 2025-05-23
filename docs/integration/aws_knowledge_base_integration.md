# AYBIZA AWS Knowledge Base Integration

## Overview

The AWS Knowledge Base Integration is a core component of the AYBIZA AI Voice Business Agent platform's hybrid Cloudflare+AWS architecture that leverages AWS Bedrock's Knowledge Base capabilities for vector-based retrieval of relevant information. This implementation works seamlessly with Claude 3.7 Sonnet's extended thinking capabilities to provide enhanced memory persistence, context retrieval, and knowledge base functionality with optimized edge caching.

## Architecture

### Key Components

1. **Vector Database Client**: Interfaces with AWS Bedrock Knowledge Base through hybrid architecture
2. **Memory Vectorization**: Converts memories to vector embeddings using Claude 3.7 Sonnet
3. **Similarity Search**: Finds relevant context based on semantic similarity with edge caching
4. **Document Processing**: Preprocesses documents with Claude 3.7 Sonnet for knowledge base ingestion
5. **Incremental Updates**: Handles efficient updates via hybrid edge-cloud synchronization
6. **Multi-Tenant Isolation**: Ensures data separation with enhanced security architecture
7. **Edge Knowledge Cache**: Cloudflare Workers cache for frequently accessed knowledge

### Data Flow

```
Memory/Document → Preprocessing → Embedding Generation →
  Knowledge Base Storage → Query Processing → Similarity Search →
    Result Ranking → Context Integration
```

## Implementation

### Core Knowledge Base Client

```elixir
defmodule Aybiza.AWS.KnowledgeBaseClient do
  @moduledoc """
  Client for AWS Bedrock Knowledge Base
  Note: This uses the aws-elixir library (not ex_aws) for AWS SDK functionality
  """
  
  require Logger
  alias AWS.Client
  
  @doc """
  Create a new knowledge base
  """
  def create_knowledge_base(name, description, opts \\ %{}) do
    # Create unique ID for the knowledge base
    kb_id = "kb_#{sanitize_name(name)}_#{System.unique_integer([:positive])}"
    
    # Set embedding model
    embedding_model = opts[:embedding_model] || "amazon.titan-embed-text-v1"
    
    # Set knowledge base config
    kb_config = %{
      "name" => name,
      "description" => description,
      "roleArn" => get_role_arn(),
      "knowledgeBaseConfiguration" => %{
        "type" => "VECTOR",
        "vectorKnowledgeBaseConfiguration" => %{
          "embeddingModelArn" => get_model_arn(embedding_model)
        }
      }
    }
    
    # Create the knowledge base
    case AWS.BedrockAgent.create_knowledge_base(client(), kb_config) do
      {:ok, response} ->
        # Log creation
        Logger.info("Created knowledge base: #{name}", %{
          knowledge_base_id: response["knowledgeBase"]["knowledgeBaseId"],
          type: "vector"
        })
        
        {:ok, response["knowledgeBase"]["knowledgeBaseId"]}
        
      {:error, error} ->
        # Log error
        Logger.error("Error creating knowledge base: #{inspect(error)}", %{
          name: name,
          error: inspect(error)
        })
        
        {:error, error}
    end
  end
  
  @doc """
  Add document to knowledge base
  """
  def add_document(knowledge_base_id, content, metadata \\ %{}) do
    # Generate unique document ID
    doc_id = "doc_#{System.unique_integer([:positive])}"
    
    # Prepare document data
    document = %{
      "dataSourceId" => doc_id,
      "document" => %{
        "contentType" => "text/plain",
        "text" => content
      },
      "metadata" => metadata
    }
    
    # Add document to knowledge base
    case AWS.BedrockAgent.ingest_to_knowledge_base(client(), knowledge_base_id, document) do
      {:ok, response} ->
        # Log addition
        Logger.info("Added document to knowledge base", %{
          knowledge_base_id: knowledge_base_id,
          document_id: doc_id,
          content_length: String.length(content)
        })
        
        {:ok, doc_id}
        
      {:error, error} ->
        # Log error
        Logger.error("Error adding document to knowledge base: #{inspect(error)}", %{
          knowledge_base_id: knowledge_base_id,
          error: inspect(error)
        })
        
        {:error, error}
    end
  end
  
  @doc """
  Delete document from knowledge base
  """
  def delete_document(knowledge_base_id, document_id) do
    # Delete document
    case AWS.BedrockAgent.delete_from_knowledge_base(client(), knowledge_base_id, document_id) do
      {:ok, _} ->
        # Log deletion
        Logger.info("Deleted document from knowledge base", %{
          knowledge_base_id: knowledge_base_id,
          document_id: document_id
        })
        
        :ok
        
      {:error, error} ->
        # Log error
        Logger.error("Error deleting document from knowledge base: #{inspect(error)}", %{
          knowledge_base_id: knowledge_base_id,
          document_id: document_id,
          error: inspect(error)
        })
        
        {:error, error}
    end
  end
  
  @doc """
  Perform similarity search in knowledge base
  """
  def search(knowledge_base_id, query, opts \\ %{}) do
    # Set default options
    max_results = opts[:max_results] || 5
    filter = opts[:filter] || %{}
    
    # Create embedding from query using same model as KB
    {:ok, embedding} = generate_embedding(query)
    
    # Prepare search request
    search_request = %{
      "knowledgeBaseId" => knowledge_base_id,
      "vectorSearchConfiguration" => %{
        "vectorSearchInput" => %{
          "vectorInput" => embedding,
          "maxResults" => max_results
        }
      }
    }
    
    # Add filter if provided
    search_request = if map_size(filter) > 0 do
      Map.put(search_request, "filter", filter)
    else
      search_request
    end
    
    # Execute search
    case AWS.BedrockAgentRuntime.retrieve(client(), knowledge_base_id, search_request) do
      {:ok, response} ->
        # Process results
        results = process_search_results(response)
        
        # Log search
        Logger.info("Performed knowledge base search", %{
          knowledge_base_id: knowledge_base_id,
          query_length: String.length(query),
          results_count: length(results)
        })
        
        {:ok, results}
        
      {:error, error} ->
        # Log error
        Logger.error("Error searching knowledge base: #{inspect(error)}", %{
          knowledge_base_id: knowledge_base_id,
          query: query,
          error: inspect(error)
        })
        
        {:error, error}
    end
  end
  
  @doc """
  Generate embeddings for text
  """
  def generate_embedding(text, model \\ "amazon.titan-embed-text-v1") do
    # Prepare payload
    payload = %{
      "inputText" => text
    }
    
    # Convert to JSON
    payload_json = Jason.encode!(payload)
    
    # Get model ARN
    model_arn = get_model_arn(model)
    
    # Generate embedding
    case AWS.BedrockRuntime.invoke_model(client(), model_arn, payload_json) do
      {:ok, response} ->
        # Parse response
        parsed = Jason.decode!(response.body)
        
        # Extract embedding
        {:ok, parsed["embedding"]}
        
      {:error, error} ->
        # Log error
        Logger.error("Error generating embedding: #{inspect(error)}", %{
          model: model,
          text_length: String.length(text),
          error: inspect(error)
        })
        
        {:error, error}
    end
  end
  
  # Process search results
  defp process_search_results(response) do
    # Extract retrieval results
    results = response["retrievalResults"] || []
    
    # Process each result
    Enum.map(results, fn result ->
      %{
        document_id: result["retrievedDocument"]["dataSourceId"],
        content: result["retrievedDocument"]["content"],
        metadata: result["retrievedDocument"]["metadata"] || %{},
        score: result["score"]
      }
    end)
  end
  
  # Get model ARN from name
  defp get_model_arn(model_name) do
    # Get AWS region
    region = Application.get_env(:aybiza, :aws_region, "us-east-1")
    
    # Get AWS account ID
    account_id = Application.get_env(:aybiza, :aws_account_id)
    
    # Construct ARN
    "arn:aws:bedrock:#{region}:#{account_id}:model/#{model_name}"
  end
  
  # Get IAM role ARN
  defp get_role_arn do
    # Get AWS region
    region = Application.get_env(:aybiza, :aws_region, "us-east-1")
    
    # Get AWS account ID
    account_id = Application.get_env(:aybiza, :aws_account_id)
    
    # Get role name
    role_name = Application.get_env(:aybiza, :aws_bedrock_role, "AybizaBedrockKnowledgeBaseRole")
    
    # Construct ARN
    "arn:aws:iam::#{account_id}:role/#{role_name}"
  end
  
  # Get AWS client
  defp client do
    AWS.Client.create(
      Application.get_env(:aybiza, :aws_access_key_id),
      Application.get_env(:aybiza, :aws_secret_access_key),
      Application.get_env(:aybiza, :aws_region, "us-east-1")
    )
  end
  
  # Sanitize name for use in IDs
  defp sanitize_name(name) do
    name
    |> String.downcase()
    |> String.replace(~r/[^a-z0-9]/, "_")
    |> String.slice(0, 20)
  end
end
```

### Memory Service Integration

```elixir
defmodule Aybiza.MemoryService.BedrockStore do
  @moduledoc """
  Stores memory data in AWS Bedrock Knowledge Base
  """
  
  alias Aybiza.AWS.KnowledgeBaseClient
  
  @doc """
  Initialize tenant knowledge base
  """
  def initialize_tenant_kb(tenant_id, name) do
    # Create knowledge base for tenant
    KnowledgeBaseClient.create_knowledge_base(
      "tenant_#{tenant_id}_memory",
      "Memory storage for tenant #{name}",
      %{
        tenant_id: tenant_id
      }
    )
  end
  
  @doc """
  Store memory in knowledge base
  """
  def store_memory(tenant_id, user_id, content, metadata \\ %{}) do
    # Get knowledge base ID for tenant
    kb_id = get_tenant_kb_id(tenant_id)
    
    # Add user ID to metadata
    metadata = Map.put(metadata, "user_id", user_id)
    
    # Store in knowledge base
    KnowledgeBaseClient.add_document(kb_id, content, metadata)
  end
  
  @doc """
  Retrieve relevant memories
  """
  def retrieve_memories(tenant_id, user_id, query, opts \\ %{}) do
    # Get knowledge base ID for tenant
    kb_id = get_tenant_kb_id(tenant_id)
    
    # Create filter for user
    filter = %{
      "metadata" => %{
        "user_id" => %{
          "equals" => user_id
        }
      }
    }
    
    # Add additional filters from options
    filter = add_filters_from_opts(filter, opts)
    
    # Search knowledge base
    KnowledgeBaseClient.search(kb_id, query, Map.merge(opts, %{filter: filter}))
  end
  
  @doc """
  Delete memories for a user
  """
  def delete_user_memories(tenant_id, user_id) do
    # Get knowledge base ID for tenant
    kb_id = get_tenant_kb_id(tenant_id)
    
    # Get all documents for user
    {:ok, memories} = retrieve_all_user_memories(tenant_id, user_id)
    
    # Delete each document
    Enum.each(memories, fn memory ->
      KnowledgeBaseClient.delete_document(kb_id, memory.document_id)
    end)
    
    {:ok, length(memories)}
  end
  
  # Get all memories for a user (for deletion)
  defp retrieve_all_user_memories(tenant_id, user_id) do
    # This is a simplified approach - in practice we'd need pagination
    # for users with many memories
    retrieve_memories(tenant_id, user_id, "", %{max_results: 100})
  end
  
  # Get knowledge base ID for tenant
  defp get_tenant_kb_id(tenant_id) do
    # In practice, this would be stored in the database and retrieved
    "kb_tenant_#{tenant_id}_memory"
  end
  
  # Add additional filters from options
  defp add_filters_from_opts(filter, opts) do
    filter = add_time_filter(filter, opts)
    filter = add_type_filter(filter, opts)
    filter = add_tag_filter(filter, opts)
    filter
  end
  
  # Add time-based filter
  defp add_time_filter(filter, %{time_frame: time_frame}) when not is_nil(time_frame) do
    # Calculate time threshold based on time_frame
    threshold = case time_frame do
      :recent -> DateTime.add(DateTime.utc_now(), -7 * 24 * 60 * 60, :second)
      :last_week -> DateTime.add(DateTime.utc_now(), -14 * 24 * 60 * 60, :second)
      :last_month -> DateTime.add(DateTime.utc_now(), -30 * 24 * 60 * 60, :second)
      _ -> nil
    end
    
    if threshold do
      put_in(filter, ["metadata", "created_at", "gte"], DateTime.to_iso8601(threshold))
    else
      filter
    end
  end
  defp add_time_filter(filter, _), do: filter
  
  # Add type filter
  defp add_type_filter(filter, %{memory_type: type}) when not is_nil(type) do
    put_in(filter, ["metadata", "memory_type", "equals"], type)
  end
  defp add_type_filter(filter, _), do: filter
  
  # Add tag filter
  defp add_tag_filter(filter, %{tag: tag}) when not is_nil(tag) do
    put_in(filter, ["metadata", "tags", "contains"], tag)
  end
  defp add_tag_filter(filter, _), do: filter
end
```

### Knowledge Base Tool

```elixir
defmodule Aybiza.Tools.GetKnowledgeBase do
  @moduledoc """
  Tool for retrieving information from knowledge bases
  """
  
  use Aybiza.Tools.Base
  
  alias Aybiza.AWS.KnowledgeBaseClient
  
  @impl true
  def definition do
    %{
      name: "get_knowledge_base",
      description: "Searches the knowledge base for articles, FAQs, and troubleshooting guides.",
      parameters: %{
        type: "object",
        properties: %{
          query: %{
            type: "string",
            description: "Search query for the knowledge base"
          },
          product_id: %{
            type: "string",
            description: "Specific product to filter results"
          },
          category: %{
            type: "string",
            description: "Category to filter results (e.g., 'troubleshooting', 'setup', 'billing')"
          },
          max_results: %{
            type: "integer",
            default: 3,
            description: "Maximum number of results to return"
          },
          summarize: %{
            type: "boolean",
            default: true,
            description: "Whether to summarize the articles for voice communication"
          }
        },
        required: ["query"]
      },
      returns: %{
        type: "object",
        properties: %{
          articles: %{
            type: "array",
            items: %{
              type: "object",
              properties: %{
                id: %{
                  type: "string",
                  description: "Article ID"
                },
                title: %{
                  type: "string",
                  description: "Article title"
                },
                summary: %{
                  type: "string",
                  description: "Summary of the article"
                },
                content: %{
                  type: "string",
                  description: "Full content of the article"
                },
                relevance_score: %{
                  type: "number",
                  description: "Relevance score (0-1) to the query"
                },
                url: %{
                  type: "string",
                  description: "URL to the full article"
                },
                last_updated: %{
                  type: "string",
                  format: "date-time",
                  description: "When the article was last updated"
                },
                product_applicable: %{
                  type: "array",
                  items: %{
                    type: "string"
                  },
                  description: "Products this article applies to"
                }
              }
            },
            description: "Knowledge base articles matching the query"
          },
          best_answer: %{
            type: "string",
            description: "Concise answer extracted from the knowledge base"
          },
          suggested_followup_queries: %{
            type: "array",
            items: %{
              type: "string"
            },
            description: "Suggested follow-up queries based on the search"
          }
        }
      },
      memory_access: :read,
      auth_required: false,
      timeout: 4000
    }
  end
  
  @impl true
  def execute(call_id, params) do
    # Get call context
    with {:ok, call} <- Aybiza.CallAnalytics.get_call(call_id) do
      # Get knowledge base ID for tenant
      kb_id = get_knowledge_base_id(call.tenant_id)
      
      # Prepare filter
      filter = build_filter_from_params(params)
      
      # Execute search
      case KnowledgeBaseClient.search(kb_id, params.query, %{
        max_results: params.max_results || 3,
        filter: filter
      }) do
        {:ok, results} ->
          # Process results
          processed_results = process_kb_results(results)
          
          # Generate best answer if requested
          answer = if params.summarize do
            generate_best_answer(params.query, processed_results)
          else
            nil
          end
          
          # Generate follow-up queries
          followups = generate_followup_queries(params.query, processed_results)
          
          # Return formatted results
          %{
            articles: processed_results,
            best_answer: answer,
            suggested_followup_queries: followups
          }
          
        {:error, error} ->
          # Return error
          %{
            error: "Search failed",
            message: "Unable to search knowledge base: #{inspect(error)}",
            articles: []
          }
      end
    end
  end
  
  # Get knowledge base ID for tenant
  defp get_knowledge_base_id(tenant_id) do
    # In practice, this would be retrieved from the database
    "kb_tenant_#{tenant_id}_knowledge"
  end
  
  # Build filter from parameters
  defp build_filter_from_params(params) do
    filter = %{}
    
    # Add product filter
    filter = if params[:product_id] do
      put_in(filter, ["metadata", "product_id", "equals"], params.product_id)
    else
      filter
    end
    
    # Add category filter
    filter = if params[:category] do
      put_in(filter, ["metadata", "category", "equals"], params.category)
    else
      filter
    end
    
    filter
  end
  
  # Process knowledge base results
  defp process_kb_results(results) do
    Enum.map(results, fn result ->
      # Extract title from content or metadata
      title = get_title(result)
      
      # Format result
      %{
        id: result.document_id,
        title: title,
        summary: generate_summary(result.content),
        content: result.content,
        relevance_score: result.score,
        url: Map.get(result.metadata, "url"),
        last_updated: Map.get(result.metadata, "last_updated"),
        product_applicable: Map.get(result.metadata, "products", [])
      }
    end)
  end
  
  # Extract title from result
  defp get_title(result) do
    cond do
      # Try to get from metadata
      Map.has_key?(result.metadata, "title") ->
        result.metadata["title"]
        
      # Try to extract from content (first line)
      true ->
        case String.split(result.content, "\n", parts: 2) do
          [first_line | _] -> first_line
          _ -> "Article #{result.document_id}"
        end
    end
  end
  
  # Generate summary of content
  defp generate_summary(content) do
    # In production, this would use an LLM to generate a concise summary
    # Here we'll use a simple approach
    max_length = 150
    
    if String.length(content) <= max_length do
      content
    else
      String.slice(content, 0, max_length) <> "..."
    end
  end
  
  # Generate best answer from results
  defp generate_best_answer(query, results) do
    # In production, this would use an LLM to generate a best answer
    # Here we'll use a simple approach
    case results do
      [first | _] ->
        "Based on our knowledge base: " <> first.summary
        
      [] ->
        "I couldn't find specific information about that in our knowledge base."
    end
  end
  
  # Generate follow-up queries
  defp generate_followup_queries(query, results) do
    # In production, this would use an LLM to generate follow-up queries
    # Here we'll use a simple approach
    [
      "Can you explain more about #{query}?",
      "What are the common issues with #{query}?",
      "How do I troubleshoot #{query} problems?"
    ]
  end
end
```

### Document Management

```elixir
defmodule Aybiza.KnowledgeBase.DocumentManager do
  @moduledoc """
  Manages document ingestion and updates for knowledge bases
  """
  
  alias Aybiza.AWS.KnowledgeBaseClient
  
  @doc """
  Ingest document into knowledge base
  """
  def ingest_document(tenant_id, document, opts \\ %{}) do
    # Get knowledge base ID
    kb_id = get_knowledge_base_id(tenant_id)
    
    # Process document
    {content, metadata} = process_document(document, opts)
    
    # Store in knowledge base
    KnowledgeBaseClient.add_document(kb_id, content, metadata)
  end
  
  @doc """
  Bulk ingest documents
  """
  def bulk_ingest(tenant_id, documents, opts \\ %{}) do
    # Get knowledge base ID
    kb_id = get_knowledge_base_id(tenant_id)
    
    # Process and ingest each document
    results = Enum.map(documents, fn document ->
      {content, metadata} = process_document(document, opts)
      
      Task.async(fn ->
        KnowledgeBaseClient.add_document(kb_id, content, metadata)
      end)
    end)
    |> Task.await_many(30_000)
    
    # Count successes and failures
    successes = Enum.count(results, fn
      {:ok, _} -> true
      _ -> false
    end)
    
    failures = Enum.count(results, fn
      {:error, _} -> true
      _ -> false
    end)
    
    %{
      total: length(documents),
      succeeded: successes,
      failed: failures
    }
  end
  
  @doc """
  Update document in knowledge base
  """
  def update_document(tenant_id, document_id, document, opts \\ %{}) do
    # Get knowledge base ID
    kb_id = get_knowledge_base_id(tenant_id)
    
    # Process document
    {content, metadata} = process_document(document, opts)
    
    # Delete existing document
    :ok = KnowledgeBaseClient.delete_document(kb_id, document_id)
    
    # Add updated document with same ID
    KnowledgeBaseClient.add_document(kb_id, content, Map.put(metadata, "dataSourceId", document_id))
  end
  
  @doc """
  Delete document from knowledge base
  """
  def delete_document(tenant_id, document_id) do
    # Get knowledge base ID
    kb_id = get_knowledge_base_id(tenant_id)
    
    # Delete document
    KnowledgeBaseClient.delete_document(kb_id, document_id)
  end
  
  # Process document for ingestion
  defp process_document(document, opts) do
    content = case document do
      %{content: content} when is_binary(content) ->
        content
        
      %{text: text} when is_binary(text) ->
        text
        
      %{html: html} when is_binary(html) ->
        html_to_text(html)
        
      %{url: url} when is_binary(url) ->
        fetch_url_content(url)
        
      _ ->
        raise "Invalid document format"
    end
    
    # Extract metadata
    metadata = Map.take(document, [:title, :category, :tags, :product_id, :url, :source])
    
    # Add timestamp
    metadata = Map.put(metadata, "last_updated", DateTime.utc_now() |> DateTime.to_iso8601())
    
    # Apply content transformations
    content = apply_content_transformations(content, opts)
    
    {content, metadata}
  end
  
  # Convert HTML to plain text
  defp html_to_text(html) do
    # In production, this would use a proper HTML-to-text converter
    # Here's a simplified version
    html
    |> String.replace(~r/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/i, "")
    |> String.replace(~r/<style\b[^<]*(?:(?!<\/style>)<[^<]*)*<\/style>/i, "")
    |> String.replace(~r/<[^>]*>/, " ")
    |> String.replace(~r/\s+/, " ")
    |> String.trim()
  end
  
  # Fetch content from URL
  defp fetch_url_content(url) do
    # In production, this would use a proper web scraper
    # Here's a simplified version
    case HTTPoison.get(url) do
      {:ok, %{body: body, status_code: 200}} ->
        html_to_text(body)
        
      _ ->
        "Error fetching content from #{url}"
    end
  end
  
  # Apply transformations to content
  defp apply_content_transformations(content, opts) do
    # Apply configured transformations
    content = if opts[:remove_pii] do
      remove_pii(content)
    else
      content
    end
    
    content = if opts[:normalize_whitespace] do
      normalize_whitespace(content)
    else
      content
    end
    
    content = if opts[:chunking] do
      chunk_content(content, opts[:chunk_size] || 1000)
    else
      content
    end
    
    content
  end
  
  # Remove PII from content
  defp remove_pii(content) do
    # In production, this would use a proper PII detector
    # Here's a simplified version that removes email addresses and phone numbers
    content
    |> String.replace(~r/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/, "[EMAIL]")
    |> String.replace(~r/\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/, "[PHONE]")
  end
  
  # Normalize whitespace
  defp normalize_whitespace(content) do
    content
    |> String.replace(~r/\s+/, " ")
    |> String.trim()
  end
  
  # Chunk content into smaller pieces
  defp chunk_content(content, chunk_size) do
    # In production, this would use semantic chunking
    # Here's a simplified version that splits by paragraphs
    paragraphs = String.split(content, "\n\n")
    
    # Combine paragraphs into chunks
    combine_paragraphs(paragraphs, chunk_size, [])
    |> Enum.join("\n\n")
  end
  
  # Combine paragraphs into chunks
  defp combine_paragraphs([], _, acc), do: Enum.reverse(acc)
  defp combine_paragraphs(paragraphs, chunk_size, acc) do
    # Take paragraphs until we exceed chunk size
    {chunk, rest} = take_until_size(paragraphs, chunk_size, "", [])
    combine_paragraphs(rest, chunk_size, [chunk | acc])
  end
  
  # Take paragraphs until we exceed chunk size
  defp take_until_size([], _, current, taken), do: {current, []}
  defp take_until_size([p | rest], size, current, taken) do
    if String.length(current) + String.length(p) <= size do
      new_current = if current == "", do: p, else: current <> "\n\n" <> p
      take_until_size(rest, size, new_current, [p | taken])
    else
      {current, rest}
    end
  end
  
  # Get knowledge base ID for tenant
  defp get_knowledge_base_id(tenant_id) do
    # In practice, this would be retrieved from the database
    "kb_tenant_#{tenant_id}_knowledge"
  end
end
```

### MCP Integration for Knowledge Bases

```elixir
defmodule Aybiza.MCPIntegration.KnowledgeConnector do
  @moduledoc """
  Connects to external knowledge systems via MCP
  """
  
  @doc """
  Generate MCP configuration for knowledge services
  """
  def generate_mcp_config do
    [
      %{
        name: "external_knowledge_base",
        description: "External knowledge base service that connects to enterprise documentation systems",
        url: "/mcp/knowledge",
        schema: %{
          properties: %{
            query: %{
              type: "string",
              description: "Search query for the knowledge base"
            },
            source: %{
              type: "string",
              enum: ["docs", "faqs", "manuals", "tickets", "all"],
              description: "Source to search within"
            },
            filters: %{
              type: "object",
              description: "Additional search filters"
            },
            max_results: %{
              type: "integer",
              default: 5,
              description: "Maximum number of results to return"
            }
          },
          required: ["query"]
        }
      },
      %{
        name: "document_retrieval",
        description: "Document retrieval service for enterprise content",
        url: "/mcp/documents",
        schema: %{
          properties: %{
            document_id: %{
              type: "string",
              description: "ID of the document to retrieve"
            },
            format: %{
              type: "string",
              enum: ["text", "html", "markdown", "summary"],
              default: "text",
              description: "Format to return the document in"
            }
          },
          required: ["document_id"]
        }
      }
    ]
  end
  
  @doc """
  Create tool definitions that use MCP knowledge services
  """
  def create_knowledge_mcp_tools do
    [
      %{
        name: "mcp_search_knowledge",
        description: "Searches external knowledge base for information",
        parameters: %{
          type: "object",
          properties: %{
            query: %{
              type: "string",
              description: "Search query for the knowledge base"
            },
            source: %{
              type: "string",
              enum: ["docs", "faqs", "manuals", "tickets", "all"],
              description: "Source to search within"
            },
            filters: %{
              type: "object",
              description: "Additional search filters"
            },
            max_results: %{
              type: "integer",
              default: 5,
              description: "Maximum number of results to return"
            }
          },
          required: ["query"]
        }
      },
      %{
        name: "mcp_get_document",
        description: "Retrieves a specific document from external systems",
        parameters: %{
          type: "object",
          properties: %{
            document_id: %{
              type: "string",
              description: "ID of the document to retrieve"
            },
            format: %{
              type: "string",
              enum: ["text", "html", "markdown", "summary"],
              default: "text",
              description: "Format to return the document in"
            }
          },
          required: ["document_id"]
        }
      }
    ]
  end
end
```

## Configuration

### Environment Variables

```bash
# AWS Configuration
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_ACCOUNT_ID=your_account_id

# Bedrock Configuration
BEDROCK_EMBEDDING_MODEL=amazon.titan-embed-text-v1
BEDROCK_ROLE_NAME=AybizaBedrockKnowledgeBaseRole

# Knowledge Base Configuration
KB_MAX_RESULTS_DEFAULT=5
KB_FILTER_THRESHOLD=0.7
```

### IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:CreateKnowledgeBase",
        "bedrock:GetKnowledgeBase",
        "bedrock:ListKnowledgeBases",
        "bedrock:DeleteKnowledgeBase",
        "bedrock:IngestDocument",
        "bedrock:DeleteDocument",
        "bedrock:Query"
      ],
      "Resource": "*"
    }
  ]
}
```

## Tool Integration

### Memory Tool Integration

The AWS Knowledge Base is integrated with memory tools:

```elixir
# Memory retrieval tool using Bedrock Knowledge Base
def retrieve_memory(call_id, params) do
  # Get call details
  {:ok, call} <- Aybiza.CallAnalytics.get_call(call_id)
  
  # Get phone number
  phone_number = params.caller_phone || call.from_number
  
  # Resolve identity
  {:ok, identity_id} = Aybiza.IdentityService.IdentityManager.resolve_identity(phone_number)
  
  # Create search options
  search_opts = %{
    max_results: params.max_results || 5,
    time_frame: params.time_frame,
    memory_type: params.memory_type
  }
  
  # Search for memories
  {:ok, results} = Aybiza.MemoryService.BedrockStore.retrieve_memories(
    call.tenant_id,
    identity_id,
    params.query,
    search_opts
  )
  
  # Format results
  memories = Enum.map(results, fn result ->
    %{
      content: result.content,
      timestamp: result.metadata["created_at"],
      relevance_score: result.score,
      source: result.metadata["source"] || "conversation"
    }
  end)
  
  # Generate summary
  summary = generate_memory_summary(memories, params.query)
  
  %{
    memories: memories,
    summary: summary
  }
end
```

### Knowledge Base Tool Integration

```elixir
# Knowledge base tool using Bedrock Knowledge Base
def get_knowledge_base(call_id, params) do
  # Get call details
  {:ok, call} <- Aybiza.CallAnalytics.get_call(call_id)
  
  # Create search options
  search_opts = %{
    max_results: params.max_results || 3,
    filter: %{
      "metadata" => %{}
    }
  }
  
  # Add product filter
  search_opts = if params.product_id do
    put_in(search_opts, [:filter, "metadata", "product_id", "equals"], params.product_id)
  else
    search_opts
  end
  
  # Add category filter
  search_opts = if params.category do
    put_in(search_opts, [:filter, "metadata", "category", "equals"], params.category)
  else
    search_opts
  end
  
  # Search knowledge base
  {:ok, results} = Aybiza.AWS.KnowledgeBaseClient.search(
    get_knowledge_base_id(call.tenant_id),
    params.query,
    search_opts
  )
  
  # Format results
  articles = Enum.map(results, fn result ->
    %{
      id: result.document_id,
      title: result.metadata["title"] || "Article #{result.document_id}",
      summary: generate_summary(result.content),
      content: result.content,
      relevance_score: result.score,
      url: result.metadata["url"],
      last_updated: result.metadata["last_updated"],
      product_applicable: result.metadata["products"] || []
    }
  end)
  
  # Generate best answer
  best_answer = if params.summarize do
    generate_best_answer(params.query, articles)
  else
    nil
  end
  
  # Generate follow-up queries
  followups = generate_followup_queries(params.query, articles)
  
  %{
    articles: articles,
    best_answer: best_answer,
    suggested_followup_queries: followups
  }
end
```

## Benefits of AWS Knowledge Base Integration

1. **Semantic Search**: Vector-based similarity search for more accurate results
2. **Persistent Memory**: Long-term storage of conversation context
3. **Scalable Architecture**: AWS-managed service that scales with usage
4. **Multi-Tenant Isolation**: Separate knowledge bases for each tenant
5. **Fast Retrieval**: Sub-second query performance for real-time voice interactions
6. **Consistent Knowledge**: Single source of truth for both memory and knowledge content
7. **MCP Extensibility**: Support for external knowledge sources via MCP

## Security Considerations

1. **Data Isolation**: Each tenant has separate knowledge base
2. **Access Controls**: IAM roles control access to knowledge bases
3. **PII Detection**: Content preprocessing removes sensitive information
4. **Audit Logs**: Comprehensive logging of all knowledge base operations
5. **Data Encryption**: All data encrypted at rest and in transit

## Data Management

1. **Data Retention**: Configurable retention policies for memory data
2. **Content Refresh**: Automated processes keep knowledge content current
3. **Versioning**: Document versioning for knowledge base content
4. **Usage Analytics**: Tracking of search queries and results
5. **Content Quality**: Monitoring and improvement of knowledge content