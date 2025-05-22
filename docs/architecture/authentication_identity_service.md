# AYBIZA Authentication & Identity Service

## Overview

The Authentication & Identity Service is a critical component of the AYBIZA AI Voice Business Agent platform that provides robust caller identification, authentication, and persistent identity management. This document outlines the architecture and implementation of this service, focusing on multi-factor authentication options for voice interactions.

## Architecture

### Key Components

1. **Identity Manager**: Core system for managing user identities across interactions
2. **Authentication Service**: Validates user identities through multiple factors
3. **Vector Identity Store**: Maintains embeddings of user identity information
4. **Identity Resolution Engine**: Matches callers to existing identities
5. **CRM Integration Layer**: Connects identities to customer records in external systems
6. **Compliance Service**: Ensures authentication meets security and regulatory requirements

### Data Flow

```
Incoming Call → Phone Number Identification → Identity Resolution →
  Authentication Request → Verification Methods → 
    Identity Confirmation → Session Context Enhancement
```

## Implementation

### Authentication Methods

The Authentication & Identity Service supports multiple authentication factors:

1. **Phone Number Recognition**: Automatic caller ID recognition
2. **Knowledge-Based Authentication (KBA)**:
   - Date of birth
   - Full name
   - Address information
   - Account numbers
   - Personal identification numbers
   - Security questions
3. **Email Verification**: One-time codes sent to registered email
4. **SMS Verification**: One-time codes sent to alternative phone numbers
5. **Voice Biometrics**: Voice print matching (optional integration)
6. **Multi-factor combinations**: Any combination of the above methods

### Identity Manager

```elixir
defmodule Aybiza.IdentityService.IdentityManager do
  @moduledoc """
  Core system for managing user identities across interactions
  """
  
  use GenServer
  
  alias Aybiza.IdentityService.IdentityStore
  alias Aybiza.IdentityService.CRMConnector
  alias Aybiza.AWS.BedrockEmbeddings
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  # Initialize the service
  @impl true
  def init(opts) do
    {:ok, %{
      settings: Map.merge(default_settings(), opts)
    }}
  end
  
  @doc """
  Resolve user identity from phone number
  """
  def resolve_identity(phone_number, opts \\ %{}) do
    GenServer.call(__MODULE__, {:resolve_identity, phone_number, opts})
  end
  
  @doc """
  Update user identity with additional information
  """
  def update_identity(identity_id, attributes, opts \\ %{}) do
    GenServer.call(__MODULE__, {:update_identity, identity_id, attributes, opts})
  end
  
  @doc """
  Authenticate a user with provided credentials
  """
  def authenticate(identity_id, credentials, opts \\ %{}) do
    GenServer.call(__MODULE__, {:authenticate, identity_id, credentials, opts})
  end
  
  @doc """
  Link identity to CRM record
  """
  def link_to_crm(identity_id, crm_system, crm_id, opts \\ %{}) do
    GenServer.call(__MODULE__, {:link_to_crm, identity_id, crm_system, crm_id, opts})
  end
  
  @doc """
  Find identity by attributes
  """
  def find_identity(attributes, opts \\ %{}) do
    GenServer.call(__MODULE__, {:find_identity, attributes, opts})
  end
  
  # Handle resolve identity request
  @impl true
  def handle_call({:resolve_identity, phone_number, opts}, _from, state) do
    # First try exact match on phone number
    case IdentityStore.find_by_phone(phone_number) do
      {:ok, identity} ->
        # Found exact match
        {:reply, {:ok, identity}, state}
      
      {:error, :not_found} ->
        # Try fuzzy matching across other identifiers
        case fuzzy_match_identity(phone_number, opts) do
          {:ok, identity} ->
            # Found fuzzy match
            {:reply, {:ok, identity}, state}
          
          {:error, :not_found} ->
            # Create new anonymous identity
            create_anonymous_identity(phone_number, state)
        end
    end
  end
  
  # Handle authenticate request
  @impl true
  def handle_call({:authenticate, identity_id, credentials, opts}, _from, state) do
    # Get identity
    case IdentityStore.get_identity(identity_id) do
      {:ok, identity} ->
        # Perform authentication
        result = authenticate_identity(identity, credentials, opts)
        
        # Log authentication attempt
        log_authentication_attempt(identity_id, credentials, result)
        
        # Return result
        {:reply, result, state}
      
      {:error, reason} ->
        # Identity not found
        {:reply, {:error, reason}, state}
    end
  end
  
  # Handle update identity request
  @impl true
  def handle_call({:update_identity, identity_id, attributes, opts}, _from, state) do
    # Get current identity
    with {:ok, identity} <- IdentityStore.get_identity(identity_id),
         :ok <- validate_attributes(attributes),
         {:ok, updated_identity} <- update_identity_attributes(identity, attributes, opts) do
      
      # Check if we should attempt CRM matching
      updated_identity = if opts[:match_crm] do
        match_with_crm(updated_identity)
      else
        updated_identity
      end
      
      # Return updated identity
      {:reply, {:ok, updated_identity}, state}
    else
      error -> {:reply, error, state}
    end
  end
  
  # Handle link to CRM request
  @impl true
  def handle_call({:link_to_crm, identity_id, crm_system, crm_id, opts}, _from, state) do
    # Get identity
    with {:ok, identity} <- IdentityStore.get_identity(identity_id),
         {:ok, crm_record} <- CRMConnector.get_record(crm_system, crm_id),
         {:ok, updated_identity} <- link_identity_to_crm(identity, crm_system, crm_id, crm_record, opts) do
      
      # Return updated identity
      {:reply, {:ok, updated_identity}, state}
    else
      error -> {:reply, error, state}
    end
  end
  
  # Handle find identity request
  @impl true
  def handle_call({:find_identity, attributes, opts}, _from, state) do
    # Search by exact attributes
    case IdentityStore.find_by_attributes(attributes) do
      {:ok, identity} ->
        # Found exact match
        {:reply, {:ok, identity}, state}
      
      {:error, :not_found} ->
        # Try vector search if enabled
        if opts[:vector_search] do
          find_by_vector_similarity(attributes, opts)
        else
          {:reply, {:error, :not_found}, state}
        end
    end
  end
  
  # Create a new anonymous identity
  defp create_anonymous_identity(phone_number, state) do
    # Generate unique ID
    identity_id = "identity_#{System.unique_integer([:positive])}"
    
    # Create basic identity record
    identity = %{
      id: identity_id,
      phone_numbers: [%{
        number: phone_number,
        type: "primary",
        verified: true,
        source: "caller_id",
        first_seen: DateTime.utc_now(),
        last_seen: DateTime.utc_now()
      }],
      attributes: %{},
      authentication_status: "unverified",
      confidence_score: 100,
      created_at: DateTime.utc_now(),
      updated_at: DateTime.utc_now(),
      interactions: []
    }
    
    # Store identity
    {:ok, _} = IdentityStore.create_identity(identity)
    
    # Return new identity
    {:reply, {:ok, identity}, state}
  end
  
  # Authenticate an identity
  defp authenticate_identity(identity, credentials, opts) do
    # Get required authentication factors
    required_factors = get_required_factors(identity, opts)
    
    # Check each required factor
    results = Enum.map(required_factors, fn factor ->
      verify_authentication_factor(identity, factor, credentials)
    end)
    
    # Check if all required factors passed
    if Enum.all?(results, fn {result, _} -> result == :ok end) do
      # Authentication succeeded
      {:ok, %{
        identity_id: identity.id,
        authentication_status: "verified",
        confidence_score: calculate_confidence_score(results),
        verified_factors: Enum.map(results, fn {_, factor} -> factor end)
      }}
    else
      # Authentication failed
      failed_factors = Enum.filter(results, fn {result, _} -> result == :error end)
      |> Enum.map(fn {_, factor} -> factor end)
      
      {:error, %{
        identity_id: identity.id,
        reason: "authentication_failed",
        failed_factors: failed_factors
      }}
    end
  end
  
  # Get required authentication factors
  defp get_required_factors(identity, opts) do
    cond do
      # Use specified factors if provided
      opts[:factors] ->
        opts[:factors]
      
      # Use identity risk level to determine factors
      identity[:risk_level] == "high" ->
        ["phone", "dob", "account_number"]
      
      identity[:risk_level] == "medium" ->
        ["phone", "dob"]
      
      # Default minimal authentication
      true ->
        ["phone"]
    end
  end
  
  # Verify a specific authentication factor
  defp verify_authentication_factor(identity, factor, credentials) do
    case factor do
      "phone" ->
        # Caller ID is already verified during identity resolution
        {:ok, "phone"}
      
      "dob" ->
        verify_date_of_birth(identity, credentials)
      
      "email" ->
        verify_email(identity, credentials)
      
      "full_name" ->
        verify_full_name(identity, credentials)
      
      "account_number" ->
        verify_account_number(identity, credentials)
      
      "security_question" ->
        verify_security_question(identity, credentials)
      
      "address" ->
        verify_address(identity, credentials)
      
      "pin" ->
        verify_pin(identity, credentials)
      
      "voice_biometrics" ->
        verify_voice_biometrics(identity, credentials)
      
      _ ->
        {:error, "unknown_factor"}
    end
  end
  
  # Verify date of birth
  defp verify_date_of_birth(identity, credentials) do
    stored_dob = get_in(identity, [:attributes, "date_of_birth"])
    provided_dob = credentials["date_of_birth"]
    
    if is_nil(stored_dob) do
      # No stored DOB, can't verify
      {:error, "dob"}
    else
      # Compare DOB
      if normalize_date(stored_dob) == normalize_date(provided_dob) do
        {:ok, "dob"}
      else
        {:error, "dob"}
      end
    end
  end
  
  # Verify full name
  defp verify_full_name(identity, credentials) do
    stored_name = get_full_name(identity)
    provided_name = credentials["full_name"]
    
    if is_nil(stored_name) do
      # No stored name, can't verify
      {:error, "full_name"}
    else
      # Compare names (normalized)
      if normalize_name(stored_name) == normalize_name(provided_name) do
        {:ok, "full_name"}
      else
        # Try fuzzy matching
        if name_similarity_score(stored_name, provided_name) > 0.8 do
          {:ok, "full_name"}
        else
          {:error, "full_name"}
        end
      end
    end
  end
  
  # Calculate authentication confidence score
  defp calculate_confidence_score(factor_results) do
    # Base confidence
    base_confidence = 60
    
    # Factor weights
    factor_weights = %{
      "phone" -> 20,
      "dob" -> 30,
      "full_name" -> 25,
      "account_number" -> 40,
      "email" -> 25,
      "security_question" -> 20,
      "address" -> 20,
      "pin" -> 35,
      "voice_biometrics" -> 45
    }
    
    # Calculate score
    verified_factors = Enum.filter(factor_results, fn {result, _} -> result == :ok end)
    |> Enum.map(fn {_, factor} -> factor end)
    
    factor_boost = Enum.reduce(verified_factors, 0, fn factor, acc ->
      acc + Map.get(factor_weights, factor, 10)
    end)
    
    # Cap at 100
    min(base_confidence + factor_boost, 100)
  end
  
  # Update identity attributes
  defp update_identity_attributes(identity, attributes, opts) do
    # Merge new attributes
    updated_attributes = Map.merge(identity.attributes || %{}, attributes)
    
    # Update identity
    updated_identity = %{
      identity |
      attributes: updated_attributes,
      updated_at: DateTime.utc_now()
    }
    
    # Determine if confidence score should be updated
    updated_identity = if opts[:update_confidence] do
      %{updated_identity | confidence_score: recalculate_confidence(updated_identity)}
    else
      updated_identity
    end
    
    # Save to store
    case IdentityStore.update_identity(updated_identity) do
      :ok -> {:ok, updated_identity}
      error -> error
    end
  end
  
  # Link identity to CRM record
  defp link_identity_to_crm(identity, crm_system, crm_id, crm_record, _opts) do
    # Update identity with CRM link
    updated_identity = %{
      identity |
      crm_links: [%{
        system: crm_system,
        id: crm_id,
        linked_at: DateTime.utc_now()
      } | (identity.crm_links || [])],
      updated_at: DateTime.utc_now()
    }
    
    # Update attributes from CRM if available
    updated_identity = update_attributes_from_crm(updated_identity, crm_record)
    
    # Save to store
    case IdentityStore.update_identity(updated_identity) do
      :ok -> {:ok, updated_identity}
      error -> error
    end
  end
  
  # Try to match with CRM systems
  defp match_with_crm(identity) do
    # Only attempt if we have enough identifying information
    if has_sufficient_matching_data?(identity) do
      # Try each configured CRM system
      results = Enum.map(get_crm_systems(), fn crm_system ->
        find_crm_match(identity, crm_system)
      end)
      
      # Find first successful match
      case Enum.find(results, fn {result, _} -> result == :ok end) do
        {:ok, match} ->
          # Link to matched CRM record
          {:ok, updated_identity} = link_identity_to_crm(
            identity,
            match.system,
            match.id,
            match.record,
            %{}
          )
          
          updated_identity
          
        nil ->
          # No matches found
          identity
      end
    else
      identity
    end
  end
  
  # Find matches by vector similarity
  defp find_by_vector_similarity(attributes, opts) do
    # Create embedding from attributes
    {:ok, embedding} = create_attribute_embedding(attributes)
    
    # Perform similarity search
    case IdentityStore.search_similar(embedding, opts[:threshold] || 0.8, opts[:limit] || 5) do
      {:ok, []} ->
        # No matches found
        {:reply, {:error, :not_found}, state}
        
      {:ok, matches} ->
        # Return best match
        [best_match | _] = matches
        {:reply, {:ok, best_match}, state}
        
      error ->
        # Error in search
        {:reply, error, state}
    end
  end
  
  # Create embedding from identity attributes
  defp create_attribute_embedding(attributes) do
    # Format attributes as text
    text = attributes
    |> Enum.map(fn {k, v} -> "#{k}: #{v}" end)
    |> Enum.join("\n")
    
    # Generate embedding using AWS Bedrock
    BedrockEmbeddings.generate_embedding(text)
  end
  
  # Get list of configured CRM systems
  defp get_crm_systems do
    Application.get_env(:aybiza, :crm_systems, ["salesforce", "hubspot"])
  end
  
  # Find matching record in CRM
  defp find_crm_match(identity, crm_system) do
    # Extract search criteria from identity
    search_criteria = extract_search_criteria(identity)
    
    # Search in CRM
    CRMConnector.search(crm_system, search_criteria)
  end
  
  # Extract search criteria from identity
  defp extract_search_criteria(identity) do
    %{
      phone: get_primary_phone(identity),
      email: get_in(identity, [:attributes, "email"]),
      name: get_full_name(identity),
      account_number: get_in(identity, [:attributes, "account_number"])
    }
    |> Enum.filter(fn {_, v} -> not is_nil(v) end)
    |> Enum.into(%{})
  end
  
  # Get full name from identity
  defp get_full_name(identity) do
    first_name = get_in(identity, [:attributes, "first_name"])
    last_name = get_in(identity, [:attributes, "last_name"])
    
    full_name = get_in(identity, [:attributes, "full_name"])
    
    cond do
      not is_nil(full_name) -> full_name
      not is_nil(first_name) and not is_nil(last_name) -> "#{first_name} #{last_name}"
      true -> nil
    end
  end
  
  # Get primary phone from identity
  defp get_primary_phone(identity) do
    case identity.phone_numbers do
      [first | _] -> first.number
      _ -> nil
    end
  end
  
  # Check if identity has sufficient data for matching
  defp has_sufficient_matching_data?(identity) do
    # Need at least phone plus one other identifier
    has_phone = not is_nil(get_primary_phone(identity))
    
    has_secondary = not is_nil(get_in(identity, [:attributes, "email"])) or
                    not is_nil(get_full_name(identity)) or
                    not is_nil(get_in(identity, [:attributes, "account_number"]))
    
    has_phone and has_secondary
  end
  
  # Update attributes from CRM record
  defp update_attributes_from_crm(identity, crm_record) do
    # Extract relevant attributes from CRM record
    crm_attributes = %{
      "first_name" => crm_record[:first_name],
      "last_name" => crm_record[:last_name],
      "email" => crm_record[:email],
      "account_number" => crm_record[:account_number],
      "account_type" => crm_record[:account_type],
      "customer_since" => crm_record[:customer_since]
    }
    |> Enum.filter(fn {_, v} -> not is_nil(v) end)
    |> Enum.into(%{})
    
    # Merge with existing attributes, preferring CRM data
    updated_attributes = Map.merge(identity.attributes || %{}, crm_attributes)
    
    %{identity | attributes: updated_attributes}
  end
  
  # Log authentication attempt
  defp log_authentication_attempt(identity_id, credentials, result) do
    # Log attempt details (sanitized)
    sanitized_credentials = sanitize_credentials(credentials)
    
    Aybiza.Security.AuditLogger.log_event(
      :authentication_attempt,
      %{
        identity_id: identity_id,
        factors: Map.keys(sanitized_credentials),
        success: match?({:ok, _}, result),
        result: if match?({:ok, data}, result), do: elem(result, 1), else: elem(result, 1)
      }
    )
  end
  
  # Sanitize credentials for logging
  defp sanitize_credentials(credentials) do
    # Remove sensitive values, keep keys
    credentials
    |> Enum.map(fn {k, _} -> {k, "[REDACTED]"} end)
    |> Enum.into(%{})
  end
  
  # Normalize date for comparison
  defp normalize_date(date) when is_binary(date) do
    # Handle common date formats
    case Regex.run(~r/(\d{1,2})[\/\-\.](\d{1,2})[\/\-\.](\d{2,4})/, date) do
      [_, day, month, year] ->
        # Ensure 4-digit year
        year = if String.length(year) == 2, do: "19#{year}", else: year
        "#{year}-#{String.pad_leading(month, 2, "0")}-#{String.pad_leading(day, 2, "0")}"
      
      nil ->
        date
    end
  end
  defp normalize_date(date), do: date
  
  # Normalize name for comparison
  defp normalize_name(name) when is_binary(name) do
    name
    |> String.downcase()
    |> String.replace(~r/[^a-z0-9]/, "")
  end
  defp normalize_name(name), do: name
  
  # Calculate similarity between names
  defp name_similarity_score(name1, name2) do
    # Simple Levenshtein-based similarity
    norm1 = normalize_name(name1)
    norm2 = normalize_name(name2)
    
    max_len = max(String.length(norm1), String.length(norm2))
    distance = String.jaro_distance(norm1, norm2)
    
    distance
  end
  
  # Default settings
  defp default_settings do
    %{
      min_confidence_threshold: 80,
      vector_search_enabled: true,
      vector_search_threshold: 0.8,
      auto_crm_matching: true,
      required_factors: %{
        "low_risk" => ["phone"],
        "medium_risk" => ["phone", "dob"],
        "high_risk" => ["phone", "dob", "account_number"]
      }
    }
  end
end
```

### Authentication Service Tool

```elixir
defmodule Aybiza.Tools.AuthenticateUser do
  @moduledoc """
  Tool for authenticating callers through various methods
  """
  
  use Aybiza.Tools.Base
  
  alias Aybiza.IdentityService.IdentityManager
  alias Aybiza.MemoryService
  
  @impl true
  def definition do
    %{
      name: "authenticate_user",
      description: "Authenticates a caller using available information. Use this when caller identity verification is needed to proceed with sensitive operations or access protected information.",
      parameters: %{
        type: "object",
        properties: %{
          method: %{
            type: "string",
            enum: ["date_of_birth", "full_name", "account_number", "email", "pin", "address", "security_question", "multi_factor"],
            description: "Authentication method to use"
          },
          date_of_birth: %{
            type: "string",
            description: "Caller's date of birth (format: YYYY-MM-DD or MM/DD/YYYY)"
          },
          full_name: %{
            type: "string",
            description: "Caller's full name"
          },
          account_number: %{
            type: "string",
            description: "Caller's account number"
          },
          email: %{
            type: "string",
            description: "Caller's email address"
          },
          pin: %{
            type: "string",
            description: "Caller's PIN or security code"
          },
          address: %{
            type: "object",
            properties: %{
              street: %{ type: "string" },
              city: %{ type: "string" },
              state: %{ type: "string" },
              zip: %{ type: "string" }
            },
            description: "Caller's address"
          },
          security_question: %{
            type: "object",
            properties: %{
              question_id: %{ type: "string" },
              answer: %{ type: "string" }
            },
            description: "Security question answer"
          },
          factors: %{
            type: "array",
            items: %{
              type: "string",
              enum: ["date_of_birth", "full_name", "account_number", "email", "pin", "address", "security_question"]
            },
            description: "List of factors to verify for multi-factor authentication"
          }
        },
        required: ["method"]
      },
      returns: %{
        type: "object",
        properties: %{
          authenticated: %{
            type: "boolean",
            description: "Whether authentication was successful"
          },
          confidence_score: %{
            type: "integer",
            description: "Confidence score of the authentication (0-100)"
          },
          verified_factors: %{
            type: "array",
            items: %{ type: "string" },
            description: "List of factors that were successfully verified"
          },
          customer_info: %{
            type: "object",
            description: "Basic customer information available after successful authentication"
          }
        }
      },
      memory_access: :read_write,
      auth_required: false,
      timeout: 5000
    }
  end
  
  @impl true
  def execute(call_id, params) do
    # Get identity from call context
    with {:ok, call} <- Aybiza.CallAnalytics.get_call(call_id),
         {:ok, identity_id} <- get_identity_from_call(call_id),
         {:ok, _} <- validate_params(params) do
      
      # Prepare credentials based on method
      credentials = prepare_authentication_credentials(params)
      
      # Perform authentication
      case IdentityManager.authenticate(identity_id, credentials, authentication_options(params)) do
        {:ok, auth_result} ->
          # Successful authentication
          
          # Get customer info if authenticated
          customer_info = if auth_result.authentication_status == "verified" do
            get_customer_info(identity_id)
          else
            %{}
          end
          
          # Store authentication result in memory
          store_authentication_in_memory(call_id, auth_result)
          
          # Update call context
          update_call_context(call_id, auth_result)
          
          # Return success result
          %{
            authenticated: auth_result.authentication_status == "verified",
            confidence_score: auth_result.confidence_score,
            verified_factors: auth_result.verified_factors,
            customer_info: customer_info
          }
          
        {:error, error} ->
          # Failed authentication
          
          # Store failed attempt in memory
          store_failed_authentication(call_id, error)
          
          # Return failure result
          %{
            authenticated: false,
            confidence_score: 0,
            verified_factors: [],
            message: get_user_friendly_error(error.reason)
          }
      end
    else
      error ->
        # Error getting identity or validating parameters
        %{
          authenticated: false,
          confidence_score: 0,
          verified_factors: [],
          message: "Unable to authenticate: #{inspect(error)}"
        }
    end
  end
  
  # Prepare authentication credentials based on method
  defp prepare_authentication_credentials(params) do
    case params.method do
      "date_of_birth" ->
        %{"date_of_birth" => params.date_of_birth}
        
      "full_name" ->
        %{"full_name" => params.full_name}
        
      "account_number" ->
        %{"account_number" => params.account_number}
        
      "email" ->
        %{"email" => params.email}
        
      "pin" ->
        %{"pin" => params.pin}
        
      "address" ->
        %{"address" => params.address}
        
      "security_question" ->
        %{
          "security_question" => %{
            "question_id" => params.security_question.question_id,
            "answer" => params.security_question.answer
          }
        }
        
      "multi_factor" ->
        # Combine all provided factors
        params
        |> Map.take(params.factors)
        |> Enum.into(%{})
    end
  end
  
  # Get authentication options based on parameters
  defp authentication_options(params) do
    case params.method do
      "multi_factor" ->
        %{factors: params.factors}
        
      method ->
        %{factors: [method]}
    end
  end
  
  # Store authentication result in memory
  defp store_authentication_in_memory(call_id, auth_result) do
    MemoryService.store_memory(call_id, %{
      content: "User authenticated with confidence score #{auth_result.confidence_score}. " <>
               "Verified factors: #{Enum.join(auth_result.verified_factors, ", ")}.",
      memory_type: "authentication",
      importance: 5,
      tags: ["authentication", "verified"]
    })
  end
  
  # Store failed authentication in memory
  defp store_failed_authentication(call_id, error) do
    MemoryService.store_memory(call_id, %{
      content: "Authentication failed: #{error.reason}",
      memory_type: "authentication",
      importance: 5,
      tags: ["authentication", "failed"]
    })
  end
  
  # Update call context with authentication result
  defp update_call_context(call_id, auth_result) do
    Aybiza.VoicePipeline.CallManager.update_call_context(call_id, %{
      authenticated: auth_result.authentication_status == "verified",
      confidence_score: auth_result.confidence_score,
      verified_factors: auth_result.verified_factors
    })
  end
  
  # Get customer info for authenticated user
  defp get_customer_info(identity_id) do
    case Aybiza.IdentityService.IdentityStore.get_identity(identity_id) do
      {:ok, identity} ->
        # Extract basic information
        %{
          name: get_in(identity, [:attributes, "full_name"]),
          email: get_in(identity, [:attributes, "email"]),
          account_type: get_in(identity, [:attributes, "account_type"]),
          customer_since: get_in(identity, [:attributes, "customer_since"])
        }
        |> Enum.filter(fn {_, v} -> not is_nil(v) end)
        |> Enum.into(%{})
        
      _ ->
        %{}
    end
  end
  
  # Get identity ID from call
  defp get_identity_from_call(call_id) do
    with {:ok, call} <- Aybiza.CallAnalytics.get_call(call_id),
         {:ok, call_context} <- Aybiza.VoicePipeline.CallManager.get_call_context(call_id) do
      
      case Map.get(call_context, :identity_id) do
        nil ->
          # Resolve identity from phone number
          IdentityManager.resolve_identity(call.from_number)
        
        identity_id ->
          {:ok, identity_id}
      end
    end
  end
  
  # Validate parameters
  defp validate_params(params) do
    case params.method do
      "date_of_birth" ->
        validate_presence(params, :date_of_birth)
        
      "full_name" ->
        validate_presence(params, :full_name)
        
      "account_number" ->
        validate_presence(params, :account_number)
        
      "email" ->
        validate_presence(params, :email)
        
      "pin" ->
        validate_presence(params, :pin)
        
      "address" ->
        validate_presence(params, :address)
        
      "security_question" ->
        validate_presence(params, :security_question)
        
      "multi_factor" ->
        validate_presence(params, :factors)
    end
  end
  
  # Validate presence of parameter
  defp validate_presence(params, field) do
    if Map.has_key?(params, field) and not is_nil(Map.get(params, field)) do
      :ok
    else
      {:error, "Missing required parameter: #{field}"}
    end
  end
  
  # Get user-friendly error message
  defp get_user_friendly_error(reason) do
    case reason do
      "authentication_failed" ->
        "The information provided doesn't match our records."
        
      _ ->
        "Authentication failed. Please try again."
    end
  end
end
```

### MCP Integration for Identity Services

Our platform can seamlessly integrate with Model Context Protocol (MCP) servers to extend authentication and identity capabilities with external systems:

```elixir
defmodule Aybiza.MCPIntegration.IdentityConnector do
  @moduledoc """
  Connects to external identity systems via MCP
  """
  
  @doc """
  Generate MCP configuration for identity services
  """
  def generate_mcp_config do
    [
      %{
        name: "identity_service",
        description: "External identity verification service that connects to enterprise authentication systems",
        url: "/mcp/identity",
        schema: %{
          properties: %{
            service_type: %{
              type: "string", 
              enum: ["authentication", "identity_resolution", "profile_lookup"]
            },
            authentication_type: %{
              type: "string",
              enum: ["knowledge_based", "document", "biometric", "multi_factor"]
            },
            query: %{
              type: "object"
            }
          },
          required: ["service_type"]
        }
      },
      %{
        name: "enterprise_directory",
        description: "Enterprise directory service for customer identification",
        url: "/mcp/directory",
        schema: %{
          properties: %{
            lookup_type: %{
              type: "string",
              enum: ["phone", "email", "name", "account", "advanced"]
            },
            query: %{
              type: "object"
            },
            include_details: %{
              type: "boolean"
            }
          },
          required: ["lookup_type", "query"]
        }
      }
    ]
  end
  
  @doc """
  Register an MCP identity provider
  """
  def register_mcp_identity_provider(provider_config) do
    # Validate config
    with :ok <- validate_provider_config(provider_config) do
      # Register with MCP registry
      Aybiza.MCPRegistry.register_provider(provider_config)
    end
  end
  
  @doc """
  Create tool definitions that use MCP identity services
  """
  def create_identity_mcp_tools do
    [
      %{
        name: "mcp_verify_identity",
        description: "Verifies caller identity using enterprise identity services via MCP",
        parameters: %{
          type: "object",
          properties: %{
            service_type: %{
              type: "string",
              enum: ["authentication", "identity_resolution", "profile_lookup"],
              description: "Type of identity service to use"
            },
            authentication_type: %{
              type: "string",
              enum: ["knowledge_based", "document", "biometric", "multi_factor"],
              description: "Type of authentication to perform"
            },
            query: %{
              type: "object",
              description: "Authentication query parameters"
            }
          },
          required: ["service_type", "query"]
        }
      },
      %{
        name: "mcp_customer_lookup",
        description: "Looks up customer information in enterprise systems via MCP",
        parameters: %{
          type: "object",
          properties: %{
            lookup_type: %{
              type: "string",
              enum: ["phone", "email", "name", "account", "advanced"],
              description: "Type of lookup to perform"
            },
            query: %{
              type: "object",
              description: "Lookup query parameters"
            },
            include_details: %{
              type: "boolean",
              description: "Whether to include detailed profile information"
            }
          },
          required: ["lookup_type", "query"]
        }
      }
    ]
  end
  
  @doc """
  Execute MCP identity verification
  """
  def execute_mcp_identity_verification(request) do
    # Send request to appropriate MCP endpoint
    Aybiza.MCPClient.send_request(
      request.service_type,
      Map.take(request, [:authentication_type, :query])
    )
  end
  
  # Validate provider configuration
  defp validate_provider_config(config) do
    # Implementation to validate provider config
    :ok
  end
end
```

## Schema Enhancements

The identity service requires the following database schema enhancements:

```sql
-- Identity table
CREATE TABLE identities (
  id VARCHAR(36) PRIMARY KEY,
  tenant_id VARCHAR(36) NOT NULL,
  confidence_score INTEGER NOT NULL DEFAULT 0,
  authentication_status VARCHAR(20) NOT NULL DEFAULT 'unverified',
  risk_level VARCHAR(10) NOT NULL DEFAULT 'medium',
  embedding VECTOR(1536),
  created_at TIMESTAMP WITH TIME ZONE NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- Phone numbers associated with identities
CREATE TABLE identity_phone_numbers (
  id VARCHAR(36) PRIMARY KEY,
  identity_id VARCHAR(36) NOT NULL REFERENCES identities(id),
  number VARCHAR(20) NOT NULL,
  type VARCHAR(20) NOT NULL,
  verified BOOLEAN NOT NULL DEFAULT false,
  source VARCHAR(50) NOT NULL,
  first_seen TIMESTAMP WITH TIME ZONE NOT NULL,
  last_seen TIMESTAMP WITH TIME ZONE NOT NULL
);

-- Identity attributes (flexible schema)
CREATE TABLE identity_attributes (
  id VARCHAR(36) PRIMARY KEY,
  identity_id VARCHAR(36) NOT NULL REFERENCES identities(id),
  key VARCHAR(100) NOT NULL,
  value TEXT,
  source VARCHAR(50) NOT NULL,
  verified BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMP WITH TIME ZONE NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- CRM links
CREATE TABLE identity_crm_links (
  id VARCHAR(36) PRIMARY KEY,
  identity_id VARCHAR(36) NOT NULL REFERENCES identities(id),
  crm_system VARCHAR(50) NOT NULL,
  crm_record_id VARCHAR(100) NOT NULL,
  linked_at TIMESTAMP WITH TIME ZONE NOT NULL,
  verified BOOLEAN NOT NULL DEFAULT false,
  confidence_score INTEGER NOT NULL DEFAULT 0
);

-- Authentication attempts
CREATE TABLE authentication_attempts (
  id VARCHAR(36) PRIMARY KEY,
  identity_id VARCHAR(36) NOT NULL REFERENCES identities(id),
  call_id VARCHAR(36) REFERENCES calls(id),
  method VARCHAR(50) NOT NULL,
  success BOOLEAN NOT NULL,
  factors JSONB NOT NULL,
  attempted_at TIMESTAMP WITH TIME ZONE NOT NULL,
  confidence_score INTEGER
);

-- Indexes
CREATE INDEX idx_identities_tenant_id ON identities(tenant_id);
CREATE INDEX idx_identity_phone_numbers_number ON identity_phone_numbers(number);
CREATE INDEX idx_identity_phone_numbers_identity_id ON identity_phone_numbers(identity_id);
CREATE INDEX idx_identity_attributes_identity_id ON identity_attributes(identity_id);
CREATE INDEX idx_identity_attributes_key ON identity_attributes(key);
CREATE INDEX idx_identity_crm_links_identity_id ON identity_crm_links(identity_id);
CREATE INDEX idx_identity_crm_links_crm ON identity_crm_links(crm_system, crm_record_id);
CREATE INDEX idx_authentication_attempts_identity_id ON authentication_attempts(identity_id);
CREATE INDEX idx_authentication_attempts_call_id ON authentication_attempts(call_id);
```

## AWS Integration

The Identity Service integrates with AWS services for enhanced functionality:

### Bedrock for Embedding-Based Identity Matching

```elixir
defmodule Aybiza.AWS.BedrockEmbeddings do
  @moduledoc """
  Uses AWS Bedrock for vector embeddings and similarity search
  """
  
  def generate_embedding(text) do
    # Configuration
    model_id = "amazon.titan-embed-text-v1"
    
    # Create payload
    payload = %{
      inputText: text
    }
    
    # Convert to JSON
    payload_json = Jason.encode!(payload)
    
    # Call AWS Bedrock API
    case ExAws.Bedrock.Runtime.invoke_model(model_id, payload_json) do
      {:ok, response} ->
        # Parse response and extract embedding
        parsed = Jason.decode!(response.body)
        {:ok, parsed["embedding"]}
        
      {:error, error} ->
        # Handle error
        {:error, error}
    end
  end
end

defmodule Aybiza.IdentityService.VectorStore do
  @moduledoc """
  Vector store for identity embeddings using AWS
  """
  
  def initialize_vector_store do
    # Configuration
    index_name = "identity_embeddings"
    vector_dimension = 1536
    
    # Create vector index
    create_vector_index(index_name, vector_dimension)
  end
  
  def store_embedding(identity_id, embedding, metadata) do
    # Store embedding in vector store
    case insert_vector(identity_id, embedding, metadata) do
      {:ok, _} -> :ok
      error -> error
    end
  end
  
  def search_similar(query_embedding, threshold, limit) do
    # Perform similarity search
    search_vectors(query_embedding, threshold, limit)
  end
  
  # Implementation would use either:
  # 1. Bedrock Knowledge Base
  # 2. OpenSearch with k-NN plugin
  # 3. RDS PostgreSQL with pgvector
  # Depending on deployment requirements
end
```

### API Deployment for MCP Integration

```yaml
# serverless.yml
service: aybiza-identity-mcp

provider:
  name: aws
  runtime: nodejs14.x
  region: us-east-1
  
functions:
  mcp-identity:
    handler: handlers.identityHandler
    events:
      - http:
          path: /mcp/identity
          method: post
          cors: true
  
  mcp-directory:
    handler: handlers.directoryHandler
    events:
      - http:
          path: /mcp/directory
          method: post
          cors: true
```

## Integration with Voice Agents

The Authentication & Identity Service is designed to be used by AI Voice Agents through tool calling:

```elixir
# Example agent system prompt with authentication capabilities
agent_system_prompt = """
You are an AI Voice Agent for ACME Financial Services. When handling sensitive information
or tasks, you MUST authenticate the caller using the authenticate_user tool.

Authentication guidelines:
1. For general account inquiries, authenticate with date_of_birth
2. For balance information, authenticate with date_of_birth AND full_name
3. For transactions or changes, authenticate with account_number AND date_of_birth
4. If authentication fails, politely explain and offer alternatives

Security reminders:
- Never reveal full account numbers or sensitive details without authentication
- Offer only general information to unauthenticated callers
- If caller mentions they're not the account holder, explain that you can only help the account owner

Once authenticated, you may use additional tools to access the caller's information.
"""
```

## Benefits of Enhanced Authentication

1. **Multi-Factor Flexibility**: Supports various authentication methods appropriate for voice
2. **Context-Aware Security**: Authentication requirements adapt based on request sensitivity
3. **CRM Integration**: Seamless connection between authenticated callers and CRM records
4. **Vector-Based Matching**: Advanced identity resolution using embeddings
5. **Voice-Optimized**: Authentication methods designed for conversation
6. **MCP Extensibility**: Supports integration with external identity systems via MCP

## Privacy and Compliance

1. **GDPR Compliance**: Supports data minimization and right to be forgotten
2. **PCI DSS**: Secure handling of payment information
3. **HIPAA Compatibility**: Appropriate handling of health information
4. **Data Retention Controls**: Configurable retention periods for identity data
5. **Consent Management**: Tracking of consent for identity data processing
6. **Audit Trail**: Comprehensive logging of all authentication actions