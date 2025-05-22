# Multi-Tenant Isolation Patterns for Claude Code

## Introduction

This document provides comprehensive guidelines for implementing multi-tenant isolation patterns in the AYBIZA AI Voice Agent Platform using Claude Code. AYBIZA's architecture requires strong tenant isolation to ensure security, data privacy, and resource fairness in a multi-tenant environment.

## Multi-Tenancy Overview

AYBIZA implements a comprehensive multi-tenant architecture with multiple layers of isolation:

1. **Database Schema Isolation**: Separate PostgreSQL schemas per tenant
2. **Row-Level Security (RLS)**: Additional database access control
3. **Tenant-Specific Encryption**: Dedicated encryption keys per tenant  
4. **Resource Quotas**: Fair resource allocation and throttling
5. **Process Isolation**: Tenant-specific execution environments
6. **Comprehensive Audit Logging**: Tenant-aware security events

## Implementation Patterns for Claude Code

When using Claude Code to implement AYBIZA components, follow these patterns to ensure proper tenant isolation:

### 1. Database Access Pattern

Always implement the tenant context pattern in database operations:

```elixir
defmodule Aybiza.TenantContext do
  @moduledoc """
  Manages tenant context for database operations.
  """
  
  def set_tenant_context(tenant_id) when is_binary(tenant_id) do
    # Set PostgreSQL session context for row-level security
    Ecto.Adapters.SQL.query(
      Aybiza.Repo, 
      "SELECT set_config('app.current_tenant_id', $1, true)",
      [tenant_id]
    )
  end
  
  def clear_tenant_context do
    Ecto.Adapters.SQL.query(
      Aybiza.Repo, 
      "SELECT set_config('app.current_tenant_id', '', true)",
      []
    )
  end
  
  def with_tenant(tenant_id, fun) when is_binary(tenant_id) and is_function(fun, 0) do
    # Save previous context if any
    previous_context = get_current_tenant_context()
    
    try do
      # Set new context and execute function
      set_tenant_context(tenant_id)
      fun.()
    after
      # Restore previous context
      if previous_context do
        set_tenant_context(previous_context)
      else
        clear_tenant_context()
      end
    end
  end
  
  def get_current_tenant_context do
    case Ecto.Adapters.SQL.query(
      Aybiza.Repo,
      "SELECT current_setting('app.current_tenant_id', true)",
      []
    ) do
      {:ok, %{rows: [[context]]}} when is_binary(context) and context != "" ->
        context
      _ ->
        nil
    end
  end
end
```

Use this pattern in your repository functions:

```elixir
defmodule Aybiza.Repo do
  use Ecto.Repo,
    otp_app: :aybiza,
    adapter: Ecto.Adapters.Postgres
  
  import Aybiza.TenantContext
  
  def tenant_aware_operation(tenant_id, fun) when is_binary(tenant_id) and is_function(fun, 0) do
    with_tenant(tenant_id, fun)
  end
  
  def get_calls(tenant_id, filters \\ %{}) do
    tenant_aware_operation(tenant_id, fn ->
      Call
      |> where(^filter_where(filters))
      |> order_by(desc: :inserted_at)
      |> all()
    end)
  end
  
  # Additional repository functions...
end
```

### 2. Schema Design Pattern

Always include tenant_id in all schema definitions:

```elixir
defmodule Aybiza.Calls.Call do
  use Ecto.Schema
  import Ecto.Changeset
  
  schema "calls" do
    # Tenant context - ALWAYS REQUIRED
    field :tenant_id, :binary_id
    field :organization_id, :binary_id
    
    # Entity fields
    field :call_id, :string
    field :status, :string
    field :duration, :integer
    field :caller_number, Aybiza.Encrypted.Binary
    field :agent_id, :binary_id
    
    # Timestamps
    timestamps()
  end
  
  def changeset(call, attrs) do
    call
    |> cast(attrs, [:tenant_id, :organization_id, :call_id, :status, :duration, :caller_number, :agent_id])
    |> validate_required([:tenant_id, :organization_id, :call_id, :status])
    |> validate_tenant_consistency()
  end
  
  # Validate that tenant_id and organization_id are consistent
  defp validate_tenant_consistency(changeset) do
    case {get_field(changeset, :tenant_id), get_field(changeset, :organization_id)} do
      {tenant_id, org_id} when is_binary(tenant_id) and is_binary(org_id) ->
        # Verify tenant belongs to organization
        # This would call a tenant validation service in a real implementation
        changeset
      _ ->
        add_error(changeset, :tenant_id, "must be valid and belong to the specified organization")
    end
  end
end
```

### 3. Context Module Pattern

Implement tenant-aware context modules for business logic:

```elixir
defmodule Aybiza.Calls do
  @moduledoc """
  The Calls context with tenant isolation.
  """
  
  import Ecto.Query
  alias Aybiza.Repo
  alias Aybiza.Calls.Call
  
  # ALWAYS require tenant_id for data access
  def get_call(tenant_id, id) when is_binary(tenant_id) and is_binary(id) do
    Repo.tenant_aware_operation(tenant_id, fn ->
      Repo.get(Call, id)
    end)
  end
  
  def list_calls(tenant_id, filters \\ %{}) when is_binary(tenant_id) do
    Repo.tenant_aware_operation(tenant_id, fn ->
      Call
      |> where(^build_filters(filters))
      |> order_by(desc: :inserted_at)
      |> Repo.all()
    end)
  end
  
  def create_call(tenant_id, attrs) when is_binary(tenant_id) do
    # Always merge tenant_id into attributes
    attrs_with_tenant = Map.put(attrs, :tenant_id, tenant_id)
    
    Repo.tenant_aware_operation(tenant_id, fn ->
      %Call{}
      |> Call.changeset(attrs_with_tenant)
      |> Repo.insert()
    end)
  end
  
  def update_call(tenant_id, %Call{} = call, attrs) when is_binary(tenant_id) do
    # Verify call belongs to tenant (belt and suspenders)
    if call.tenant_id != tenant_id do
      {:error, :not_found}
    else
      Repo.tenant_aware_operation(tenant_id, fn ->
        call
        |> Call.changeset(attrs)
        |> Repo.update()
      end)
    end
  end
  
  def delete_call(tenant_id, %Call{} = call) when is_binary(tenant_id) do
    # Verify call belongs to tenant (belt and suspenders)
    if call.tenant_id != tenant_id do
      {:error, :not_found}
    else
      Repo.tenant_aware_operation(tenant_id, fn ->
        Repo.delete(call)
      end)
    end
  end
  
  # Helper functions
  defp build_filters(filters) do
    Enum.reduce(filters, dynamic(true), fn
      {:status, status}, dynamic ->
        dynamic([c], ^dynamic and c.status == ^status)
      {:caller_number, number}, dynamic ->
        dynamic([c], ^dynamic and c.caller_number == ^number)
      # Additional filters...
      {_, _}, dynamic ->
        dynamic
    end)
  end
end
```

### 4. API Endpoint Pattern

Implement tenant context verification in API endpoints:

```elixir
defmodule AybizaWeb.CallController do
  use AybizaWeb, :controller
  
  alias Aybiza.Calls
  alias Aybiza.Calls.Call
  
  # Add plug to verify tenant context
  plug :verify_tenant_access
  
  def index(conn, params) do
    # Get tenant_id from authenticated context
    tenant_id = conn.assigns.tenant_id
    
    # Get pagination parameters
    page = Map.get(params, "page", 1)
    page_size = Map.get(params, "page_size", 20)
    
    # List calls with tenant isolation
    calls = Calls.list_calls(tenant_id, params)
    
    render(conn, :index, calls: calls)
  end
  
  def create(conn, %{"call" => call_params}) do
    # Get tenant_id from authenticated context
    tenant_id = conn.assigns.tenant_id
    
    # Create call with tenant isolation
    with {:ok, %Call{} = call} <- Calls.create_call(tenant_id, call_params) do
      conn
      |> put_status(:created)
      |> put_resp_header("location", ~p"/api/calls/#{call}")
      |> render(:show, call: call)
    end
  end
  
  # Additional controller actions...
  
  # Verify tenant access
  defp verify_tenant_access(conn, _opts) do
    # This function verifies the tenant context from authentication
    # The tenant_id and organization_id should be added by authentication plug
    case conn.assigns do
      %{tenant_id: tenant_id} when is_binary(tenant_id) ->
        # Valid tenant context
        conn
      _ ->
        # Invalid tenant context
        conn
        |> put_status(:unauthorized)
        |> put_view(json: AybizaWeb.ErrorJSON)
        |> render(:"401", %{message: "Unauthorized access"})
        |> halt()
    end
  end
end
```

### 5. Authentication Pattern

Implement tenant-aware authentication:

```elixir
defmodule AybizaWeb.AuthPlug do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    # Extract authentication from request
    conn
    |> get_authorization_header()
    |> verify_token()
    |> assign_tenant_context(conn)
  end
  
  # Extract authorization header
  defp get_authorization_header(conn) do
    case get_req_header(conn, "authorization") do
      ["Bearer " <> token] -> {:ok, token}
      _ -> {:error, :missing_authorization}
    end
  end
  
  # Verify token and extract claims
  defp verify_token({:ok, token}) do
    case Aybiza.Security.TokenVerifier.verify_token(token) do
      {:ok, claims} -> {:ok, claims}
      error -> error
    end
  end
  
  defp verify_token(error), do: error
  
  # Assign tenant context to connection
  defp assign_tenant_context({:ok, claims}, conn) do
    # Extract tenant_id and organization_id from claims
    # CRUCIAL: These must be verified by the token verification process
    tenant_id = Map.get(claims, "tenant_id")
    organization_id = Map.get(claims, "organization_id")
    user_id = Map.get(claims, "sub")
    
    # Add to connection assigns
    conn
    |> assign(:tenant_id, tenant_id)
    |> assign(:organization_id, organization_id)
    |> assign(:current_user_id, user_id)
    |> assign(:auth_claims, claims)
  end
  
  defp assign_tenant_context({:error, reason}, conn) do
    conn
    |> put_status(:unauthorized)
    |> put_view(json: AybizaWeb.ErrorJSON)
    |> render(:"401", %{message: "Authentication failed: #{reason}"})
    |> halt()
  end
end
```

### 6. Authorization Pattern

Implement tenant-aware authorization policies:

```elixir
defmodule Aybiza.Policies do
  @moduledoc """
  Authorization policies with tenant isolation.
  """
  
  def authorize(:view_call, %{tenant_id: user_tenant_id}, %{tenant_id: call_tenant_id}) do
    # Allow access only to calls in the user's tenant
    user_tenant_id == call_tenant_id
  end
  
  def authorize(:update_call, %{tenant_id: user_tenant_id, role: role}, %{tenant_id: call_tenant_id}) do
    # Allow update only to calls in the user's tenant
    # and only for users with appropriate role
    user_tenant_id == call_tenant_id and role in ["admin", "manager", "agent"]
  end
  
  def authorize(:delete_call, %{tenant_id: user_tenant_id, role: role}, %{tenant_id: call_tenant_id}) do
    # Allow deletion only to calls in the user's tenant
    # and only for users with appropriate role
    user_tenant_id == call_tenant_id and role in ["admin", "manager"]
  end
  
  # Use a default deny pattern
  def authorize(_, _, _), do: false
end
```

Configure Bodyguard for authorization:

```elixir
defmodule Aybiza.Authorization do
  @moduledoc """
  Authorization behaviour implementation with tenant isolation.
  """
  
  use Bodyguard.Policy
  
  # Delegate to Policies module
  def authorize(action, user, resource) do
    Aybiza.Policies.authorize(action, user, resource)
  end
end
```

Use authorization in context modules:

```elixir
defmodule Aybiza.Calls do
  # Existing implementations...
  
  # Add authorization checks to data access functions
  def get_call(tenant_id, id, current_user) do
    with {:ok, call} <- get_call(tenant_id, id) do
      case Bodyguard.permit(Aybiza.Authorization, :view_call, current_user, call) do
        :ok -> {:ok, call}
        {:error, reason} -> {:error, reason}
      end
    end
  end
  
  def update_call(tenant_id, %Call{} = call, attrs, current_user) do
    with :ok <- Bodyguard.permit(Aybiza.Authorization, :update_call, current_user, call) do
      # Proceed with update...
      update_call(tenant_id, call, attrs)
    end
  end
  
  def delete_call(tenant_id, %Call{} = call, current_user) do
    with :ok <- Bodyguard.permit(Aybiza.Authorization, :delete_call, current_user, call) do
      # Proceed with delete...
      delete_call(tenant_id, call)
    end
  end
end
```

### 7. GenServer and Process Isolation Pattern

Implement tenant-aware GenServer processes:

```elixir
defmodule Aybiza.TenantProcess do
  @moduledoc """
  Helper for tenant-isolated processes.
  """
  
  def via_tuple(tenant_id, key) do
    # Use Registry for process discovery with tenant isolation
    {:via, Registry, {Aybiza.Registry, {tenant_id, key}}}
  end
  
  def child_spec(tenant_id, module, key, args) do
    # Create child spec with tenant isolation
    %{
      id: {module, tenant_id, key},
      start: {module, :start_link, [tenant_id, key, args]},
      restart: :transient,
      shutdown: 5000,
      type: :worker
    }
  end
  
  def start_process(tenant_id, module, key, args) do
    # Start process under tenant-specific dynamic supervisor
    supervisor_name = get_tenant_supervisor(tenant_id)
    DynamicSupervisor.start_child(
      supervisor_name,
      child_spec(tenant_id, module, key, args)
    )
  end
  
  def get_tenant_supervisor(tenant_id) do
    # Get or create tenant-specific supervisor
    case Registry.lookup(Aybiza.Registry, {:tenant_supervisor, tenant_id}) do
      [{pid, _}] ->
        pid
      [] ->
        # Create new tenant supervisor
        {:ok, pid} = DynamicSupervisor.start_link(
          name: via_tuple(tenant_id, {:tenant_supervisor, tenant_id}),
          strategy: :one_for_one
        )
        pid
    end
  end
end
```

Implement tenant-aware GenServer template:

```elixir
defmodule Aybiza.TenantGenServer do
  @moduledoc """
  Template for tenant-isolated GenServer.
  """
  
  defmacro __using__(opts) do
    quote do
      use GenServer
      import Aybiza.TenantProcess
      require Logger
      
      # Client API
      def start_link(tenant_id, key, initial_args) do
        # Start with tenant isolation
        name = via_tuple(tenant_id, {__MODULE__, key})
        GenServer.start_link(__MODULE__, %{tenant_id: tenant_id, key: key, args: initial_args}, name: name)
      end
      
      # Server callbacks
      def init(%{tenant_id: tenant_id} = state) do
        Logger.metadata(tenant_id: tenant_id)
        
        # Initialize with tenant context
        {:ok, state}
      end
      
      # Default handle_call with tenant validation
      def handle_call({:call, _action, _args}, {from_pid, _}, %{tenant_id: tenant_id} = state) do
        # Validate caller's tenant context
        case validate_caller_tenant(from_pid, tenant_id) do
          :ok ->
            # Process normally
            {:reply, {:error, :not_implemented}, state}
          {:error, reason} ->
            # Log unauthorized access attempt
            Logger.warning("Unauthorized tenant access: #{reason}")
            {:reply, {:error, :unauthorized}, state}
        end
      end
      
      # Default handle_cast with tenant validation
      def handle_cast({:cast, _action, _args, caller_tenant_id}, %{tenant_id: tenant_id} = state) do
        # Validate caller's tenant context
        if caller_tenant_id == tenant_id do
          # Process normally
          {:noreply, state}
        else
          # Log unauthorized access attempt
          Logger.warning("Unauthorized tenant access attempt")
          {:noreply, state}
        end
      end
      
      # Terminate with cleanup
      def terminate(reason, %{tenant_id: tenant_id} = state) do
        Logger.info("Terminating tenant process, reason: #{inspect(reason)}")
        :ok
      end
      
      # Helper for validating caller tenant
      defp validate_caller_tenant(pid, tenant_id) do
        # In a real implementation, this would verify the caller's tenant context
        # This is a simplified example
        :ok
      end
      
      # Allow overriding
      defoverridable init: 1, handle_call: 3, handle_cast: 2, terminate: 2
    end
  end
end
```

Use the tenant-aware GenServer in your implementations:

```elixir
defmodule Aybiza.Calls.CallHandler do
  @moduledoc """
  Handles active call processing with tenant isolation.
  """
  
  use Aybiza.TenantGenServer
  
  # Client API
  def start_call(tenant_id, call_id, call_params) do
    Aybiza.TenantProcess.start_process(tenant_id, __MODULE__, call_id, call_params)
  end
  
  def get_call_state(tenant_id, call_id) do
    GenServer.call(via_tuple(tenant_id, {__MODULE__, call_id}), {:call, :get_state, nil})
  end
  
  def update_call_state(tenant_id, call_id, update) do
    GenServer.cast(via_tuple(tenant_id, {__MODULE__, call_id}), {:cast, :update_state, update, tenant_id})
  end
  
  def end_call(tenant_id, call_id) do
    GenServer.cast(via_tuple(tenant_id, {__MODULE__, call_id}), {:cast, :end_call, nil, tenant_id})
  end
  
  # Server callbacks
  def init(%{tenant_id: tenant_id, key: call_id, args: call_params} = state) do
    # Call super
    super(state)
    
    # Initialize call state
    {:ok, Map.merge(state, %{
      call_state: :initializing,
      call_data: call_params,
      started_at: DateTime.utc_now()
    })}
  end
  
  def handle_call({:call, :get_state, _}, _from, state) do
    {:reply, {:ok, state.call_state}, state}
  end
  
  def handle_cast({:cast, :update_state, update, _tenant_id}, state) do
    new_state = Map.merge(state.call_data, update)
    {:noreply, %{state | call_data: new_state}}
  end
  
  def handle_cast({:cast, :end_call, _, _tenant_id}, state) do
    # Log call end
    Logger.info("Ending call #{state.key}", tenant_id: state.tenant_id)
    
    # Perform cleanup
    # This would include saving call data, metrics, etc.
    
    # Terminate the process
    {:stop, :normal, state}
  end
end
```

### 8. Background Job Pattern

Implement tenant isolation in background jobs:

```elixir
defmodule Aybiza.Jobs.TenantJob do
  @moduledoc """
  Base module for tenant-isolated background jobs.
  """
  
  defmacro __using__(opts) do
    quote do
      use Oban.Worker
      import Aybiza.TenantContext
      require Logger
      
      # Add tenant context to job execution
      @impl Oban.Worker
      def perform(%Oban.Job{args: %{"tenant_id" => tenant_id} = args}) do
        # Set Logger metadata
        Logger.metadata(tenant_id: tenant_id)
        
        # Execute with tenant context
        Aybiza.TenantContext.with_tenant(tenant_id, fn ->
          # Call implementation
          execute(args)
        end)
      end
      
      # Handle missing tenant_id
      def perform(%Oban.Job{}) do
        Logger.error("Job executed without tenant_id")
        {:error, "Missing tenant_id"}
      end
      
      # To be implemented by concrete job modules
      def execute(args) do
        {:error, :not_implemented}
      end
      
      # Allow overriding
      defoverridable execute: 1
    end
  end
end
```

Create tenant-aware job:

```elixir
defmodule Aybiza.Jobs.CallTranscriptionJob do
  @moduledoc """
  Background job for call transcription with tenant isolation.
  """
  
  use Aybiza.Jobs.TenantJob
  
  # Implementation
  def execute(%{"call_id" => call_id} = args) do
    Logger.info("Processing transcription for call #{call_id}")
    
    # Process transcription
    # This code runs with tenant context already set
    
    # Success
    :ok
  end
end
```

Enqueue job with tenant context:

```elixir
defmodule Aybiza.CallTranscriptions do
  @moduledoc """
  Context for call transcriptions with tenant isolation.
  """
  
  def schedule_transcription(tenant_id, call_id, options \\ %{}) do
    # Create job with tenant context
    %{
      tenant_id: tenant_id,
      call_id: call_id,
      options: options
    }
    |> Aybiza.Jobs.CallTranscriptionJob.new()
    |> Oban.insert()
  end
end
```

### 9. PubSub Pattern

Implement tenant-isolated PubSub:

```elixir
defmodule Aybiza.TenantPubSub do
  @moduledoc """
  PubSub utilities with tenant isolation.
  """
  
  def subscribe(tenant_id, topic) when is_binary(tenant_id) and is_binary(topic) do
    # Subscribe to tenant-specific topic
    Phoenix.PubSub.subscribe(Aybiza.PubSub, tenant_topic(tenant_id, topic))
  end
  
  def unsubscribe(tenant_id, topic) when is_binary(tenant_id) and is_binary(topic) do
    # Unsubscribe from tenant-specific topic
    Phoenix.PubSub.unsubscribe(Aybiza.PubSub, tenant_topic(tenant_id, topic))
  end
  
  def broadcast(tenant_id, topic, message) when is_binary(tenant_id) and is_binary(topic) do
    # Broadcast to tenant-specific topic
    Phoenix.PubSub.broadcast(Aybiza.PubSub, tenant_topic(tenant_id, topic), message)
  end
  
  def broadcast_from(from_pid, tenant_id, topic, message) when is_binary(tenant_id) and is_binary(topic) do
    # Broadcast from PID to tenant-specific topic
    Phoenix.PubSub.broadcast_from(Aybiza.PubSub, from_pid, tenant_topic(tenant_id, topic), message)
  end
  
  # Create tenant-specific topic
  defp tenant_topic(tenant_id, topic) do
    "tenant:#{tenant_id}:#{topic}"
  end
end
```

Use tenant-isolated PubSub:

```elixir
defmodule Aybiza.Calls.CallChannel do
  @moduledoc """
  Socket channel for call events with tenant isolation.
  """
  
  use Phoenix.Channel
  alias Aybiza.TenantPubSub
  
  # Join with tenant validation
  def join("call:" <> call_id, _params, socket) do
    # Extract tenant_id from socket assigns
    tenant_id = socket.assigns.tenant_id
    
    # Validate authorization
    case Aybiza.Calls.authorize_call_access(tenant_id, call_id, socket.assigns.current_user) do
      :ok ->
        # Subscribe to tenant-specific topics
        TenantPubSub.subscribe(tenant_id, "call:#{call_id}:events")
        TenantPubSub.subscribe(tenant_id, "call:#{call_id}:transcripts")
        
        {:ok, assign(socket, :call_id, call_id)}
      {:error, reason} ->
        {:error, %{reason: reason}}
    end
  end
  
  # Handle call events from PubSub
  def handle_info({:call_event, event}, socket) do
    # Push event to client
    push(socket, "call_event", event)
    {:noreply, socket}
  end
  
  # Handle transcript events from PubSub
  def handle_info({:transcript, transcript}, socket) do
    # Push transcript to client
    push(socket, "transcript", transcript)
    {:noreply, socket}
  end
end
```

### 10. Encryption Pattern

Implement tenant-specific encryption:

```elixir
defmodule Aybiza.Encryption.TenantKeys do
  @moduledoc """
  Manages tenant-specific encryption keys.
  """
  
  def get_tenant_key(tenant_id) when is_binary(tenant_id) do
    # Get tenant key from vault
    # This would integrate with a key management service in production
    # Simplified example using environment variables
    tenant_key = System.get_env("TENANT_KEY_#{tenant_id}")
    
    if tenant_key do
      {:ok, tenant_key}
    else
      {:error, :key_not_found}
    end
  end
  
  def encrypt(tenant_id, plaintext) when is_binary(tenant_id) and is_binary(plaintext) do
    with {:ok, key} <- get_tenant_key(tenant_id) do
      # Initialize cipher
      iv = :crypto.strong_rand_bytes(16)
      
      # Encrypt with AES-GCM
      {ciphertext, tag} = :crypto.crypto_one_time_aead(
        :aes_gcm, key, iv, plaintext, tenant_id, 16, true
      )
      
      # Format encrypted data
      {:ok, %{
        iv: Base.encode64(iv),
        ciphertext: Base.encode64(ciphertext),
        tag: Base.encode64(tag),
        tenant_id: tenant_id
      }}
    end
  end
  
  def decrypt(tenant_id, encrypted_data) when is_binary(tenant_id) and is_map(encrypted_data) do
    with {:ok, key} <- get_tenant_key(tenant_id),
         {:ok, iv} <- decode_base64(encrypted_data.iv),
         {:ok, ciphertext} <- decode_base64(encrypted_data.ciphertext),
         {:ok, tag} <- decode_base64(encrypted_data.tag) do
      
      # Decrypt with AES-GCM
      plaintext = :crypto.crypto_one_time_aead(
        :aes_gcm, key, iv, ciphertext, tenant_id, tag, false
      )
      
      {:ok, plaintext}
    end
  end
  
  # Helper functions
  defp decode_base64(encoded) do
    case Base.decode64(encoded) do
      {:ok, decoded} -> {:ok, decoded}
      :error -> {:error, :invalid_encoding}
    end
  end
end
```

Use tenant-specific encryption with Cloak:

```elixir
defmodule Aybiza.Encrypted.TenantCipher do
  @moduledoc """
  Vault for tenant-specific encryption.
  """
  
  use Cloak.Vault, otp_app: :aybiza
  
  # This would be configured in your application configuration
  # config :aybiza, Aybiza.Encrypted.TenantCipher,
  #   default: Aybiza.Encrypted.TenantAES
  
  def encrypt(plaintext, tenant_id) do
    # Get the configured cipher
    cipher = __MODULE__.__cloak__(:default)
    
    # Encrypt with tenant context
    cipher.encrypt(plaintext, tenant_id: tenant_id)
  end
  
  def decrypt(ciphertext, tenant_id) do
    # Get tag from ciphertext
    {tag, _} = Cloak.Tags.split(ciphertext)
    
    # Get cipher for tag
    cipher = __MODULE__.__cloak__(tag)
    
    # Decrypt with tenant context
    cipher.decrypt(ciphertext, tenant_id: tenant_id)
  end
end
```

Define encrypted types with tenant context:

```elixir
defmodule Aybiza.Encrypted.Binary do
  @moduledoc """
  Encrypted binary type with tenant isolation.
  """
  
  use Cloak.Ecto.Binary, vault: Aybiza.Encrypted.TenantCipher
  
  # Override dump to include tenant context
  def dump(value, type, tenant_id) do
    # Get tenant ID from process dictionary if not provided
    tenant_id = tenant_id || Process.get(:current_tenant_id)
    
    if tenant_id do
      super(value, type, tenant_id: tenant_id)
    else
      {:error, "Missing tenant context for encryption"}
    end
  end
end
```

### 11. API Rate Limiting Pattern

Implement tenant-aware rate limiting:

```elixir
defmodule Aybiza.RateLimiter do
  @moduledoc """
  Rate limiter with tenant isolation.
  """
  
  def limit_requests(conn, opts) do
    # Extract tenant ID from connection
    tenant_id = conn.assigns.tenant_id
    
    # Get rate limit configuration for tenant
    limits = get_tenant_limits(tenant_id, opts)
    
    # Apply rate limiting with tenant key
    scale = opts[:scale] || 60_000 # Default 1 minute
    limit = limits.requests_per_minute
    
    # Check limit
    case Hammer.check_rate("api:#{tenant_id}:#{conn.request_path}", scale, limit) do
      {:allow, _count} ->
        # Request allowed
        conn
      {:deny, _limit} ->
        # Request denied - rate limit exceeded
        conn
        |> Plug.Conn.put_status(:too_many_requests)
        |> Phoenix.Controller.put_view(json: AybizaWeb.ErrorJSON)
        |> Phoenix.Controller.render(:"429", %{message: "Rate limit exceeded"})
        |> Plug.Conn.halt()
    end
  end
  
  # Get tenant-specific rate limits
  defp get_tenant_limits(tenant_id, opts) do
    # This would fetch tenant-specific limits from database or cache
    # Simplified example
    default_limits = %{
      requests_per_minute: opts[:default_rpm] || 60,
      requests_per_hour: opts[:default_rph] || 1000,
      requests_per_day: opts[:default_rpd] || 10000
    }
    
    # Get tenant-specific overrides
    case Aybiza.Tenants.get_rate_limits(tenant_id) do
      {:ok, limits} -> Map.merge(default_limits, limits)
      _ -> default_limits
    end
  end
end
```

### 12. Memory and Resource Isolation Pattern

Implement memory and CPU quotas for tenant processes:

```elixir
defmodule Aybiza.Resources.TenantLimiter do
  @moduledoc """
  Process resource limiting with tenant isolation.
  """
  
  def with_limits(tenant_id, opts \\ [], fun) when is_function(fun, 0) do
    # Get tenant resource limits
    limits = get_tenant_limits(tenant_id, opts)
    
    # Create a supervised task with resource limits
    Task.Supervisor.async_nolink(
      Aybiza.TaskSupervisor,
      fn ->
        # Set process flags for resource limiting
        set_process_limits(limits)
        
        # Execute function
        fun.()
      end
    )
    |> Task.await(limits.timeout)
  end
  
  # Apply process limits
  defp set_process_limits(limits) do
    # Set maximum heap size (memory limit)
    if limits.memory_limit_mb do
      :erlang.process_flag(:max_heap_size, limits.memory_limit_mb * 1024 * 1024)
    end
    
    # Set priority (CPU usage)
    if limits.priority do
      :erlang.process_flag(:priority, limits.priority)
    end
  end
  
  # Get tenant resource limits
  defp get_tenant_limits(tenant_id, opts) do
    # Default limits
    defaults = %{
      memory_limit_mb: 100,
      timeout: 5_000,
      priority: :normal
    }
    
    # Merge with options
    limits = Map.merge(defaults, Map.new(opts))
    
    # Get tenant-specific limits
    case Aybiza.Tenants.get_resource_limits(tenant_id) do
      {:ok, tenant_limits} -> Map.merge(limits, tenant_limits)
      _ -> limits
    end
  end
end
```

### 13. Audit Logging Pattern

Implement tenant-aware audit logging:

```elixir
defmodule Aybiza.Security.AuditLogger do
  @moduledoc """
  Audit logger with tenant isolation.
  """
  
  require Logger
  alias Aybiza.Repo
  alias Aybiza.Security.AuditLog
  
  def log_event(tenant_id, user_id, event_type, details \\ %{}) do
    # Create audit log entry
    log_entry = %{
      tenant_id: tenant_id,
      user_id: user_id,
      event_type: event_type,
      event_time: DateTime.utc_now(),
      ip_address: get_client_ip(),
      details: details
    }
    
    # Log to database with tenant context
    Aybiza.TenantContext.with_tenant(tenant_id, fn ->
      %AuditLog{}
      |> AuditLog.changeset(log_entry)
      |> Repo.insert()
    end)
    
    # Also log to logger for real-time monitoring
    Logger.info("AUDIT: #{event_type} for tenant #{tenant_id}", tenant_id: tenant_id)
    
    :ok
  end
  
  # Helper for getting client IP
  defp get_client_ip do
    # In a real implementation, this would extract client IP from current process metadata
    # Simplified example
    Process.get(:client_ip) || "127.0.0.1"
  end
end
```

### 14. External Service Integration Pattern

Implement tenant isolation for external services:

```elixir
defmodule Aybiza.ExternalServices.BedrockClient do
  @moduledoc """
  AWS Bedrock client with tenant isolation.
  """
  
  def new(tenant_id) do
    # Initialize client with tenant context
    %{
      tenant_id: tenant_id,
      credentials: get_tenant_credentials(tenant_id),
      rate_limiter: Aybiza.ExternalServices.RateLimiter.for_tenant(tenant_id, "bedrock")
    }
  end
  
  def invoke(client, params) do
    # Validate tenant context
    unless client.tenant_id do
      raise "Missing tenant context for Bedrock invocation"
    end
    
    # Check rate limits
    case client.rate_limiter.check_limit() do
      :ok ->
        # Make API call with tenant context
        perform_invoke(client, params)
      {:error, :rate_limited} ->
        {:error, :rate_limited}
    end
  end
  
  # Helper functions
  defp get_tenant_credentials(tenant_id) do
    # Get tenant-specific AWS credentials
    # This would integrate with a credential management system
    Aybiza.AWS.Credentials.for_tenant(tenant_id)
  end
  
  defp perform_invoke(client, params) do
    # Log request for auditing
    Aybiza.Security.AuditLogger.log_event(
      client.tenant_id,
      nil,
      "bedrock_invoke",
      %{
        model_id: params.model_id,
        max_tokens: params.max_tokens,
        has_tools: params.tools != nil
      }
    )
    
    # Make actual API call
    # This would call AWS Bedrock API with tenant credentials
    # Simplified example
    {:ok, %{
      completion: "This is a sample response.",
      model: params.model_id,
      usage: %{
        prompt_tokens: 100,
        completion_tokens: 50
      }
    }}
  end
end
```

## Validation and Testing

### 1. Tenant Isolation Testing

Create comprehensive tests for tenant isolation:

```elixir
defmodule Aybiza.TenantIsolationTest do
  use Aybiza.DataCase
  
  describe "tenant database isolation" do
    test "only returns data for the specified tenant" do
      # Create test tenants
      tenant_1 = insert_tenant()
      tenant_2 = insert_tenant()
      
      # Create data for tenant 1
      call_1 = insert(:call, tenant_id: tenant_1.id, status: "completed")
      
      # Create data for tenant 2
      call_2 = insert(:call, tenant_id: tenant_2.id, status: "completed")
      
      # Query with tenant 1 context
      result_1 = Aybiza.TenantContext.with_tenant(tenant_1.id, fn ->
        Aybiza.Repo.all(Aybiza.Calls.Call)
      end)
      
      # Query with tenant 2 context
      result_2 = Aybiza.TenantContext.with_tenant(tenant_2.id, fn ->
        Aybiza.Repo.all(Aybiza.Calls.Call)
      end)
      
      # Verify tenant isolation
      assert length(result_1) == 1
      assert hd(result_1).id == call_1.id
      
      assert length(result_2) == 1
      assert hd(result_2).id == call_2.id
    end
    
    test "prevents cross-tenant data access" do
      # Create test tenants
      tenant_1 = insert_tenant()
      tenant_2 = insert_tenant()
      
      # Create data for tenant 1
      call = insert(:call, tenant_id: tenant_1.id)
      
      # Attempt to access from tenant 2 context
      result = Aybiza.TenantContext.with_tenant(tenant_2.id, fn ->
        Aybiza.Repo.get(Aybiza.Calls.Call, call.id)
      end)
      
      # Verify data is not accessible
      assert result == nil
    end
  end
  
  describe "API tenant isolation" do
    test "prevents cross-tenant API access" do
      # Create test tenants
      tenant_1 = insert_tenant()
      tenant_2 = insert_tenant()
      
      # Create data for tenant 1
      call = insert(:call, tenant_id: tenant_1.id)
      
      # Create auth tokens
      token_1 = create_token(tenant_1.id)
      token_2 = create_token(tenant_2.id)
      
      # Attempt access with tenant 1 token
      conn = build_conn()
      |> put_req_header("authorization", "Bearer #{token_1}")
      |> get("/api/calls/#{call.id}")
      
      # Verify access allowed
      assert json_response(conn, 200)["id"] == call.id
      
      # Attempt access with tenant 2 token
      conn = build_conn()
      |> put_req_header("authorization", "Bearer #{token_2}")
      |> get("/api/calls/#{call.id}")
      
      # Verify access denied
      assert json_response(conn, 404)["error"] == "Resource not found"
    end
  end
  
  describe "process isolation" do
    test "maintains isolation between tenant processes" do
      # Create test tenants
      tenant_1 = insert_tenant()
      tenant_2 = insert_tenant()
      
      # Start process for tenant 1
      {:ok, pid_1} = Aybiza.Calls.CallHandler.start_call(
        tenant_1.id,
        "call_1",
        %{status: "in_progress"}
      )
      
      # Start process for tenant 2
      {:ok, pid_2} = Aybiza.Calls.CallHandler.start_call(
        tenant_2.id,
        "call_2",
        %{status: "in_progress"}
      )
      
      # Verify processes are running
      assert Process.alive?(pid_1)
      assert Process.alive?(pid_2)
      
      # Get state from tenant 1
      {:ok, state_1} = Aybiza.Calls.CallHandler.get_call_state(tenant_1.id, "call_1")
      
      # Verify tenant 1 can access its state
      assert state_1 == :initializing
      
      # Attempt to access tenant 1's call from tenant 2 context
      result = try do
        Aybiza.Calls.CallHandler.get_call_state(tenant_2.id, "call_1")
      rescue
        e -> {:error, e}
      end
      
      # Verify access is denied
      assert match?({:error, _}, result)
    end
  end
  
  # Helper functions
  defp insert_tenant do
    %Aybiza.Tenants.Tenant{}
    |> Aybiza.Tenants.Tenant.changeset(%{
      name: "Tenant #{System.unique_integer([:positive])}",
      status: "active"
    })
    |> Aybiza.Repo.insert!()
  end
  
  defp create_token(tenant_id) do
    # Create auth token for testing
    # This would use your actual token generation logic
    "test_token_#{tenant_id}"
  end
end
```

## Security Best Practices

### 1. Defense in Depth

Implement multiple layers of tenant isolation:

1. **API Authentication**: Tenant context in all auth tokens
2. **API Authorization**: Explicit tenant validation in all endpoints
3. **Database Isolation**: Schema separation and row-level security
4. **Process Isolation**: Tenant-aware process management
5. **Encryption**: Tenant-specific encryption keys
6. **Resource Limits**: Tenant-specific quotas and rate limits
7. **Audit Logging**: Comprehensive tenant-aware audit trail

### 2. Default Deny

Always use a default deny pattern for tenant access:

- Explicitly check tenant_id for all data access
- Deny access by default, only allow with explicit verification
- Always validate tenant IDs across trust boundaries
- Implement comprehensive error handling for tenant validation

### 3. Tenant Context Propagation

Ensure tenant context is propagated throughout the system:

- Include tenant_id in auth tokens
- Pass tenant_id to all external services
- Store tenant_id in process metadata
- Include tenant_id in background jobs
- Add tenant_id to all log entries

## Conclusion

Proper tenant isolation is critical for the security and reliability of the AYBIZA platform. By following these patterns when using Claude Code to implement AYBIZA components, you can ensure strong tenant isolation across all layers of the system, from database access to API endpoints to background processing.

## References

- [Elixir Best Practices Guide](29_elixir_best_practices_guide.md)
- [Security Compliance Implementation](09_security_compliance_implementation.md)
- [Authentication & Identity Service](30_authentication_identity_service.md)
- [AWS IAM Configuration](19_aws_iam_configuration.md)
- [System Architecture](03_system_architecture.md)