# Audit Logging Implementation Guide for AYBIZA

## Introduction

This document provides a comprehensive guide for implementing audit logging in the AYBIZA AI Voice Agent Platform. Proper audit logging is essential for security compliance, incident investigation, and maintaining a chain of evidence for security events.

The AYBIZA platform implements a multi-layered audit logging architecture that captures security-relevant events across all components of the system, with special attention to AI interactions, tenant isolation, and regulatory compliance requirements.

## Audit Logging Architecture

### 1. Core Components

The audit logging system consists of several key components:

1. **Core AuditLogger Service**: Central service for recording security events
2. **Specialized Event Loggers**: Domain-specific loggers for different components
3. **PaperTrail Integration**: Change tracking for data modifications
4. **Structured Event Schema**: Standardized schema for all audit events
5. **Log Storage and Indexing**: Secure, immutable storage with indexing for search
6. **Access Control**: Tenant-aware access to audit logs
7. **Alert Integration**: Real-time alerting for security events

```
┌────────────────┐     ┌───────────────────┐     ┌────────────────┐
│  API Endpoints │────▶│  Authentication/  │────▶│  Controllers   │
└────────────────┘     │  Authorization    │     └────────────────┘
                       └───────────────────┘              │
                                │                         ▼
                                │                ┌────────────────┐
                                ▼                │  Business      │
                       ┌───────────────────┐    │  Logic         │
                       │  Audit Logging    │◀───┤                │
                       │  Service          │    └────────────────┘
                       └───────────────────┘              │
                                │                         ▼
                                ▼                ┌────────────────┐
┌────────────────┐     ┌───────────────────┐    │  Database      │
│  Alert System  │◀────│  Security Event   │◀───│  Operations    │
└────────────────┘     │  Monitoring       │    └────────────────┘
                       └───────────────────┘
                                │
                                ▼
                       ┌───────────────────┐
                       │  Long-term        │
                       │  Archival Storage │
                       └───────────────────┘
```

### 2. Log Types and Categories

AYBIZA's audit logging categorizes events into the following types:

1. **Authentication Events**: Login, logout, password changes, etc.
2. **Authorization Events**: Access grants, denials, role changes
3. **Data Access Events**: Access to sensitive data (PHI, PII)
4. **Data Modification Events**: Create, update, delete operations
5. **AI Interaction Events**: Bedrock API calls, prompt injection attempts
6. **System Configuration Events**: Changes to system configuration
7. **Security Violation Events**: Potential security breaches
8. **User Management Events**: User creation, deletion, role changes
9. **Tenant Management Events**: Tenant creation, configuration changes
10. **Voice Pipeline Events**: Security events in voice processing

## Database Schema

### 1. Core Audit Log Schema

The primary audit log schema in the AYBIZA system:

```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  organization_id UUID NOT NULL,
  actor_id UUID,
  actor_type VARCHAR(50),
  event_type VARCHAR(100) NOT NULL,
  resource_type VARCHAR(100),
  resource_id VARCHAR(255),
  event_time TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
  ip_address VARCHAR(45),
  user_agent TEXT,
  details JSONB,
  
  -- Indexes for efficient querying
  CONSTRAINT fk_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id),
  CONSTRAINT fk_organization FOREIGN KEY (organization_id) REFERENCES organizations(id)
);

-- Indexes for common query patterns
CREATE INDEX idx_audit_logs_tenant_time ON audit_logs(tenant_id, event_time DESC);
CREATE INDEX idx_audit_logs_event_type ON audit_logs(event_type, event_time DESC);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_actor ON audit_logs(actor_id, event_time DESC);
CREATE INDEX idx_audit_logs_details ON audit_logs USING gin(details);
```

### 2. Bedrock API Audit Log Schema

Specialized schema for AWS Bedrock API calls:

```sql
CREATE TABLE bedrock_audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  user_id UUID,
  model_id VARCHAR(100) NOT NULL,
  request_type VARCHAR(20) NOT NULL, -- :invoke or :stream
  prompt_hash VARCHAR(64) NOT NULL,
  ip_address VARCHAR(45),
  session_id VARCHAR(128),
  call_id VARCHAR(128),
  timestamp TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
  metadata JSONB,
  
  CONSTRAINT fk_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);

CREATE INDEX idx_bedrock_audit_tenant_time ON bedrock_audit_logs(tenant_id, timestamp DESC);
CREATE INDEX idx_bedrock_audit_user ON bedrock_audit_logs(user_id, timestamp DESC);
CREATE INDEX idx_bedrock_audit_call ON bedrock_audit_logs(call_id);
```

### 3. Security Event Schema

Schema for security events that require immediate attention:

```sql
CREATE TABLE security_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  severity VARCHAR(20) NOT NULL, -- 'info', 'warning', 'critical'
  event_type VARCHAR(100) NOT NULL,
  source VARCHAR(100) NOT NULL,
  description TEXT NOT NULL,
  event_time TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
  related_audit_log_id UUID,
  metadata JSONB,
  resolved BOOLEAN DEFAULT false,
  resolution_time TIMESTAMP WITH TIME ZONE,
  resolution_notes TEXT,
  
  CONSTRAINT fk_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id),
  CONSTRAINT fk_audit_log FOREIGN KEY (related_audit_log_id) REFERENCES audit_logs(id)
);

CREATE INDEX idx_security_events_tenant ON security_events(tenant_id, event_time DESC);
CREATE INDEX idx_security_events_type ON security_events(event_type, event_time DESC);
CREATE INDEX idx_security_events_resolved ON security_events(resolved, event_time DESC);
```

## Implementation Examples

### 1. Core AuditLogger Module

The central AuditLogger service:

```elixir
defmodule Aybiza.Security.AuditLogger do
  @moduledoc """
  Central service for audit logging across the AYBIZA platform.
  """
  
  alias Aybiza.Repo
  alias Aybiza.Security.AuditLog
  require Logger
  
  @doc """
  Logs a security-relevant event to the audit log.
  
  ## Parameters
  
  - tenant_id: The tenant ID (required)
  - actor_id: The ID of the user or service performing the action (optional)
  - event_type: The type of event (required)
  - resource_type: The type of resource being accessed (optional)
  - resource_id: The ID of the resource being accessed (optional)
  - details: Additional details about the event (optional)
  
  ## Examples
  
      Aybiza.Security.AuditLogger.log_event(
        tenant_id,
        current_user.id,
        "login_success",
        %{
          ip_address: "192.168.1.1",
          user_agent: "Mozilla/5.0..."
        }
      )
  """
  def log_event(tenant_id, actor_id, event_type, details \\ %{}) do
    # Ensure tenant_id is present
    unless tenant_id do
      Logger.error("Attempted to log audit event without tenant_id: #{event_type}")
      return {:error, :missing_tenant_id}
    end
    
    # Get organization_id from tenant
    organization_id = get_organization_id(tenant_id)
    
    # Build the audit log entry
    audit_entry = %{
      tenant_id: tenant_id,
      organization_id: organization_id,
      actor_id: actor_id,
      actor_type: determine_actor_type(actor_id),
      event_type: event_type,
      resource_type: Map.get(details, :resource_type),
      resource_id: Map.get(details, :resource_id),
      event_time: DateTime.utc_now(),
      ip_address: Map.get(details, :ip_address) || get_client_ip(),
      user_agent: Map.get(details, :user_agent) || get_user_agent(),
      details: sanitize_details(details)
    }
    
    # Insert the audit log entry
    %AuditLog{}
    |> AuditLog.changeset(audit_entry)
    |> Repo.insert()
    |> case do
      {:ok, log} ->
        # Check if this is a security event that requires alerting
        if security_event?(event_type) do
          create_security_event(log)
        end
        
        {:ok, log}
        
      {:error, changeset} ->
        Logger.error("Failed to create audit log: #{inspect(changeset.errors)}")
        {:error, changeset}
    end
  end
  
  @doc """
  Logs a data access event for sensitive data.
  
  ## Examples
  
      Aybiza.Security.AuditLogger.log_data_access(
        tenant_id,
        current_user.id,
        "patient_record",
        patient_id,
        "view"
      )
  """
  def log_data_access(tenant_id, actor_id, resource_type, resource_id, access_type) do
    log_event(tenant_id, actor_id, "data_access", %{
      resource_type: resource_type,
      resource_id: resource_id,
      access_type: access_type
    })
  end
  
  @doc """
  Logs a data modification event.
  
  ## Examples
  
      Aybiza.Security.AuditLogger.log_data_modification(
        tenant_id,
        current_user.id,
        "patient_record",
        patient_id,
        "update",
        %{changed_fields: ["name", "address"]}
      )
  """
  def log_data_modification(tenant_id, actor_id, resource_type, resource_id, modification_type, details \\ %{}) do
    log_event(tenant_id, actor_id, "data_modification", Map.merge(details, %{
      resource_type: resource_type,
      resource_id: resource_id,
      modification_type: modification_type
    }))
  end
  
  # Private helper functions
  
  defp determine_actor_type(nil), do: "system"
  defp determine_actor_type(actor_id) do
    cond do
      is_user?(actor_id) -> "user"
      is_service?(actor_id) -> "service"
      is_api_key?(actor_id) -> "api_key"
      true -> "unknown"
    end
  end
  
  defp get_organization_id(tenant_id) do
    case Aybiza.Tenants.get_tenant(tenant_id) do
      %{organization_id: org_id} -> org_id
      _ -> nil
    end
  end
  
  defp get_client_ip do
    # Retrieve from process dictionary if available
    Process.get(:client_ip)
  end
  
  defp get_user_agent do
    # Retrieve from process dictionary if available
    Process.get(:user_agent)
  end
  
  defp sanitize_details(details) do
    # Remove any sensitive information
    details
    |> Map.drop([:password, :token, :secret_key, :api_key])
    |> redact_sensitive_fields()
  end
  
  defp redact_sensitive_fields(details) do
    # Redact known sensitive field patterns
    Enum.reduce(details, %{}, fn {key, value}, acc ->
      if sensitive_field?(key) do
        Map.put(acc, key, "[REDACTED]")
      else
        Map.put(acc, key, value)
      end
    end)
  end
  
  defp sensitive_field?(field) when is_atom(field) do
    sensitive_field?(Atom.to_string(field))
  end
  
  defp sensitive_field?(field) when is_binary(field) do
    patterns = ["password", "secret", "token", "key", "credential", "ssn", "credit_card"]
    Enum.any?(patterns, &String.contains?(String.downcase(field), &1))
  end
  
  defp security_event?(event_type) do
    security_events = [
      "login_failure",
      "password_reset",
      "role_change",
      "permission_change",
      "data_export",
      "configuration_change",
      "prompt_injection_detected",
      "rate_limit_exceeded",
      "unauthorized_access_attempt",
      "api_key_created",
      "api_key_deleted"
    ]
    
    event_type in security_events
  end
  
  defp create_security_event(audit_log) do
    Aybiza.Security.SecurityEvent.create_from_audit_log(audit_log)
  end
end
```

### 2. Specialized Bedrock API Audit Logger

Specialized logger for AWS Bedrock API calls:

```elixir
defmodule Aybiza.Security.BedrockAuditLogger do
  @moduledoc """
  Specialized audit logger for AWS Bedrock API calls.
  """
  
  alias Aybiza.Repo
  alias Aybiza.Security.BedrockAuditLog
  require Logger
  
  @doc """
  Logs a Bedrock API call for auditing purposes.
  """
  def log_bedrock_request(request_params) do
    %BedrockAuditLog{
      timestamp: DateTime.utc_now(),
      tenant_id: request_params.tenant_id,
      user_id: request_params.user_id,
      model_id: request_params.model_id,
      request_type: request_params.type, # :invoke or :stream
      prompt_hash: hash_prompt(request_params.prompt),
      ip_address: request_params.ip_address,
      session_id: request_params.session_id,
      call_id: request_params.call_id,
      metadata: %{
        max_tokens: request_params.max_tokens,
        temperature: request_params.temperature,
        has_tools: request_params.tools != nil
      }
    }
    |> Repo.insert()
  end
  
  @doc """
  Retrieves Bedrock API call logs for a specific tenant.
  """
  def get_bedrock_logs(tenant_id, filters \\ %{}) do
    BedrockAuditLog
    |> where([l], l.tenant_id == ^tenant_id)
    |> apply_filters(filters)
    |> order_by([l], desc: l.timestamp)
    |> Repo.all()
  end
  
  # Private helper functions
  
  defp hash_prompt(prompt) do
    # Create a one-way hash of the prompt for privacy
    :crypto.hash(:sha256, prompt)
    |> Base.encode16()
    |> String.downcase()
  end
  
  defp apply_filters(query, filters) do
    Enum.reduce(filters, query, fn {key, value}, query ->
      case key do
        :user_id -> where(query, [l], l.user_id == ^value)
        :model_id -> where(query, [l], l.model_id == ^value)
        :call_id -> where(query, [l], l.call_id == ^value)
        :start_date -> where(query, [l], l.timestamp >= ^value)
        :end_date -> where(query, [l], l.timestamp <= ^value)
        _ -> query
      end
    end)
  end
end
```

### 3. Authentication Audit Logger

Specialized logger for authentication events:

```elixir
defmodule Aybiza.Security.AuthAuditLogger do
  @moduledoc """
  Specialized audit logger for authentication events.
  """
  
  alias Aybiza.Security.AuditLogger
  
  def log_login_success(tenant_id, user_id, conn) do
    AuditLogger.log_event(tenant_id, user_id, "login_success", %{
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn),
      auth_method: "password"
    })
  end
  
  def log_login_failure(tenant_id, user_id, reason, conn) do
    AuditLogger.log_event(tenant_id, user_id, "login_failure", %{
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn),
      reason: reason
    })
    
    # Also create a security event for failed logins
    check_brute_force_attempts(tenant_id, user_id, get_client_ip(conn))
  end
  
  def log_mfa_success(tenant_id, user_id, conn) do
    AuditLogger.log_event(tenant_id, user_id, "mfa_success", %{
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn),
      mfa_method: "totp"
    })
  end
  
  def log_mfa_failure(tenant_id, user_id, conn) do
    AuditLogger.log_event(tenant_id, user_id, "mfa_failure", %{
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn),
      mfa_method: "totp"
    })
    
    # Also create a security event for failed MFA attempts
    check_mfa_failure_attempts(tenant_id, user_id, get_client_ip(conn))
  end
  
  def log_logout(tenant_id, user_id, conn) do
    AuditLogger.log_event(tenant_id, user_id, "logout", %{
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn)
    })
  end
  
  def log_password_reset_request(tenant_id, user_id, conn) do
    AuditLogger.log_event(tenant_id, user_id, "password_reset_request", %{
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn)
    })
  end
  
  def log_password_change(tenant_id, user_id, conn) do
    AuditLogger.log_event(tenant_id, user_id, "password_change", %{
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn)
    })
  end
  
  # Private helper functions
  
  defp get_client_ip(conn) do
    # Get client IP from the connection
    conn.remote_ip
    |> Tuple.to_list()
    |> Enum.join(".")
  end
  
  defp get_user_agent(conn) do
    # Get user agent from the connection
    case get_req_header(conn, "user-agent") do
      [user_agent] -> user_agent
      _ -> nil
    end
  end
  
  defp check_brute_force_attempts(tenant_id, user_id, ip_address) do
    # Check for potential brute force attempts
    recent_failures = count_recent_failures(tenant_id, user_id, ip_address)
    
    if recent_failures >= 5 do
      # Create a security event for potential brute force attack
      Aybiza.Security.SecurityEvent.create(%{
        tenant_id: tenant_id,
        severity: "warning",
        event_type: "potential_brute_force",
        source: "authentication",
        description: "Multiple failed login attempts detected",
        metadata: %{
          user_id: user_id,
          ip_address: ip_address,
          attempt_count: recent_failures
        }
      })
    end
  end
  
  defp count_recent_failures(tenant_id, user_id, ip_address) do
    # Count recent failed login attempts
    one_hour_ago = DateTime.utc_now() |> DateTime.add(-3600, :second)
    
    Aybiza.Security.AuditLog
    |> where([l], l.tenant_id == ^tenant_id)
    |> where([l], l.actor_id == ^user_id)
    |> where([l], l.event_type == "login_failure")
    |> where([l], l.ip_address == ^ip_address)
    |> where([l], l.event_time >= ^one_hour_ago)
    |> select([l], count(l.id))
    |> Repo.one()
  end
  
  defp check_mfa_failure_attempts(tenant_id, user_id, ip_address) do
    # Similar to check_brute_force_attempts but for MFA failures
    # Implementation details...
  end
end
```

### 4. PaperTrail Integration

Integration with PaperTrail for change tracking:

```elixir
defmodule Aybiza.PaperTrailHook do
  @moduledoc """
  Hook to integrate PaperTrail with AuditLogger.
  """
  
  use PaperTrail.Hook
  alias Aybiza.Security.AuditLogger
  
  @impl true
  def after_insert(changeset, _struct, version) do
    handle_version(:insert, changeset, version)
  end
  
  @impl true
  def after_update(changeset, _struct, version) do
    handle_version(:update, changeset, version)
  end
  
  @impl true
  def after_delete(changeset, _struct, version) do
    handle_version(:delete, changeset, version)
  end
  
  defp handle_version(action, changeset, version) do
    # Get schema name
    schema_name = changeset.data.__struct__
                  |> Module.split()
                  |> List.last()
    
    # Get tenant ID
    tenant_id = Map.get(changeset.data, :tenant_id) || 
                fetch_tenant_id_from_meta(changeset)
    
    # Get actor ID
    actor_id = version.whodunnit
    
    # Log the data modification
    if tenant_id do
      AuditLogger.log_data_modification(
        tenant_id,
        actor_id,
        schema_name,
        Map.get(changeset.data, :id),
        action,
        %{
          changed_fields: Map.keys(version.item_changes),
          version_id: version.id
        }
      )
    end
  end
  
  defp fetch_tenant_id_from_meta(changeset) do
    # Try to get tenant_id from meta
    Map.get(changeset.meta, :tenant_id)
  end
end
```

## API Integration Examples

### 1. Controller Integration

Example of integrating audit logging in controllers:

```elixir
defmodule AybizaWeb.CallController do
  use AybizaWeb, :controller
  
  alias Aybiza.Calls
  alias Aybiza.Security.AuditLogger
  
  def index(conn, params) do
    tenant_id = conn.assigns.tenant_id
    current_user = conn.assigns.current_user
    
    # Log the access to call list
    AuditLogger.log_event(tenant_id, current_user.id, "list_calls", %{
      resource_type: "calls",
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn),
      filter_params: sanitize_params(params)
    })
    
    calls = Calls.list_calls(tenant_id, params)
    render(conn, :index, calls: calls)
  end
  
  def show(conn, %{"id" => id}) do
    tenant_id = conn.assigns.tenant_id
    current_user = conn.assigns.current_user
    
    with {:ok, call} <- Calls.get_call(tenant_id, id) do
      # Log the access to specific call
      AuditLogger.log_data_access(
        tenant_id,
        current_user.id,
        "call",
        id,
        "view"
      )
      
      render(conn, :show, call: call)
    end
  end
  
  def create(conn, %{"call" => call_params}) do
    tenant_id = conn.assigns.tenant_id
    current_user = conn.assigns.current_user
    
    # Log attempt to create call
    AuditLogger.log_event(tenant_id, current_user.id, "create_call_attempt", %{
      resource_type: "call",
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn)
    })
    
    with {:ok, call} <- Calls.create_call(tenant_id, call_params) do
      # Log successful creation
      AuditLogger.log_data_modification(
        tenant_id,
        current_user.id,
        "call",
        call.id,
        "create"
      )
      
      conn
      |> put_status(:created)
      |> put_resp_header("location", ~p"/api/calls/#{call}")
      |> render(:show, call: call)
    end
  end
  
  # Helper functions
  
  defp get_client_ip(conn) do
    conn.remote_ip
    |> Tuple.to_list()
    |> Enum.join(".")
  end
  
  defp get_user_agent(conn) do
    case get_req_header(conn, "user-agent") do
      [user_agent] -> user_agent
      _ -> nil
    end
  end
  
  defp sanitize_params(params) do
    # Remove sensitive data from params before logging
    params
    |> Map.drop(["password", "token", "secret"])
  end
end
```

### 2. Channel Integration

Example of integrating audit logging in Phoenix Channels:

```elixir
defmodule AybizaWeb.CallChannel do
  use Phoenix.Channel
  
  alias Aybiza.Security.AuditLogger
  
  def join("call:" <> call_id, _params, socket) do
    tenant_id = socket.assigns.tenant_id
    current_user_id = socket.assigns.current_user_id
    
    # Log channel join
    AuditLogger.log_event(tenant_id, current_user_id, "join_call_channel", %{
      resource_type: "call",
      resource_id: call_id,
      ip_address: get_client_ip(socket),
      user_agent: get_user_agent(socket)
    })
    
    # Authorize join
    case Aybiza.Calls.authorize_call_access(tenant_id, call_id, current_user_id) do
      :ok ->
        {:ok, assign(socket, :call_id, call_id)}
      {:error, reason} ->
        # Log failed join attempt
        AuditLogger.log_event(tenant_id, current_user_id, "join_call_channel_denied", %{
          resource_type: "call",
          resource_id: call_id,
          reason: reason,
          ip_address: get_client_ip(socket),
          user_agent: get_user_agent(socket)
        })
        
        {:error, %{reason: reason}}
    end
  end
  
  def handle_in("send_message", %{"content" => content}, socket) do
    tenant_id = socket.assigns.tenant_id
    current_user_id = socket.assigns.current_user_id
    call_id = socket.assigns.call_id
    
    # Log message send
    AuditLogger.log_event(tenant_id, current_user_id, "send_call_message", %{
      resource_type: "call",
      resource_id: call_id,
      message_size: String.length(content),
      ip_address: get_client_ip(socket),
      user_agent: get_user_agent(socket)
    })
    
    # Process message...
    
    {:reply, :ok, socket}
  end
  
  def terminate(reason, socket) do
    tenant_id = socket.assigns.tenant_id
    current_user_id = socket.assigns.current_user_id
    call_id = socket.assigns.call_id
    
    # Log channel leave
    AuditLogger.log_event(tenant_id, current_user_id, "leave_call_channel", %{
      resource_type: "call",
      resource_id: call_id,
      reason: inspect(reason),
      ip_address: get_client_ip(socket),
      user_agent: get_user_agent(socket)
    })
    
    :ok
  end
  
  # Helper functions
  
  defp get_client_ip(socket) do
    socket.assigns[:client_ip]
  end
  
  defp get_user_agent(socket) do
    socket.assigns[:user_agent]
  end
end
```

### 3. GraphQL Integration

Example of integrating audit logging in GraphQL resolvers:

```elixir
defmodule AybizaWeb.Schema.Resolvers.Calls do
  alias Aybiza.Calls
  alias Aybiza.Security.AuditLogger
  
  def list_calls(_, args, %{context: context}) do
    tenant_id = context.tenant_id
    current_user_id = context.current_user_id
    
    # Log the graphql query
    AuditLogger.log_event(tenant_id, current_user_id, "graphql_query", %{
      resource_type: "calls",
      operation: "list_calls",
      filter_params: sanitize_params(args),
      ip_address: context.ip_address,
      user_agent: context.user_agent
    })
    
    # Execute the query
    {:ok, Calls.list_calls(tenant_id, args)}
  end
  
  def get_call(_, %{id: id}, %{context: context}) do
    tenant_id = context.tenant_id
    current_user_id = context.current_user_id
    
    # Log the graphql query
    AuditLogger.log_data_access(
      tenant_id,
      current_user_id,
      "call",
      id,
      "view"
    )
    
    # Execute the query
    case Calls.get_call(tenant_id, id) do
      {:ok, call} -> {:ok, call}
      {:error, reason} -> {:error, reason}
    end
  end
  
  # Helper functions
  
  defp sanitize_params(params) do
    # Remove sensitive data from params before logging
    params
    |> Map.drop([:password, :token, :secret])
  end
end
```

## Security Event Detection and Monitoring

### 1. Prompt Injection Detection

Example of detecting and logging prompt injection attempts:

```elixir
defmodule Aybiza.Security.PromptSecurityMonitor do
  alias Aybiza.Security.AuditLogger
  alias Aybiza.Security.SecurityEvent
  
  @prompt_injection_patterns [
    ~r/ignore previous instructions/i,
    ~r/ignore all instructions/i,
    ~r/ignore above instructions/i,
    ~r/disregard previous/i,
    ~r/forget your training/i,
    ~r/you are now a/i,
    ~r/system: you are/i
  ]
  
  def analyze_prompt(tenant_id, user_id, prompt, metadata) do
    # Check for prompt injection patterns
    matched_patterns = Enum.filter(@prompt_injection_patterns, fn pattern ->
      String.match?(prompt, pattern)
    end)
    
    if length(matched_patterns) > 0 do
      # Log the potential prompt injection attempt
      AuditLogger.log_event(tenant_id, user_id, "potential_prompt_injection", %{
        matched_patterns: Enum.map(matched_patterns, &Regex.source/1),
        prompt_hash: hash_prompt(prompt),
        resource_type: "bedrock_api",
        ip_address: metadata[:ip_address],
        user_agent: metadata[:user_agent]
      })
      
      # Create a security event
      SecurityEvent.create(%{
        tenant_id: tenant_id,
        severity: "warning",
        event_type: "potential_prompt_injection",
        source: "bedrock_api",
        description: "Potential prompt injection attempt detected",
        metadata: %{
          user_id: user_id,
          matched_patterns: Enum.map(matched_patterns, &Regex.source/1),
          prompt_hash: hash_prompt(prompt)
        }
      })
      
      # Return warning
      {:warning, :potential_prompt_injection}
    else
      # No injection detected
      :ok
    end
  end
  
  defp hash_prompt(prompt) do
    :crypto.hash(:sha256, prompt)
    |> Base.encode16()
    |> String.downcase()
  end
end
```

### 2. Abnormal API Usage Detection

Example of detecting and logging abnormal API usage:

```elixir
defmodule Aybiza.Security.APIUsageMonitor do
  alias Aybiza.Security.AuditLogger
  alias Aybiza.Security.SecurityEvent
  
  def monitor_api_usage(tenant_id) do
    # Calculate baseline usage patterns
    baseline = calculate_baseline(tenant_id)
    
    # Get current usage
    current = get_current_usage(tenant_id)
    
    # Check for abnormal usage
    abnormal_patterns = detect_abnormal_patterns(baseline, current)
    
    if length(abnormal_patterns) > 0 do
      # Log the abnormal usage
      AuditLogger.log_event(tenant_id, nil, "abnormal_api_usage", %{
        abnormal_patterns: abnormal_patterns,
        baseline: baseline,
        current: current
      })
      
      # Create a security event
      SecurityEvent.create(%{
        tenant_id: tenant_id,
        severity: "warning",
        event_type: "abnormal_api_usage",
        source: "api_monitor",
        description: "Abnormal API usage patterns detected",
        metadata: %{
          abnormal_patterns: abnormal_patterns,
          baseline: baseline,
          current: current
        }
      })
      
      # Return warning
      {:warning, :abnormal_api_usage}
    else
      # No abnormal usage detected
      :ok
    end
  end
  
  # Helper functions
  
  defp calculate_baseline(tenant_id) do
    # Calculate baseline usage patterns
    # Implementation details...
  end
  
  defp get_current_usage(tenant_id) do
    # Get current usage patterns
    # Implementation details...
  end
  
  defp detect_abnormal_patterns(baseline, current) do
    # Detect abnormal patterns
    # Implementation details...
  end
end
```

## Audit Log Access Control

### 1. Access Control Policy

Implementing access control for audit logs:

```elixir
defmodule Aybiza.Security.AuditLogPolicy do
  @moduledoc """
  Authorization policy for audit log access.
  """
  
  @doc """
  Determines if the user can view audit logs.
  """
  def authorize(:view_audit_logs, %{tenant_id: user_tenant_id, role: role}, %{tenant_id: tenant_id}) do
    # Only allow access to logs from the same tenant
    user_tenant_id == tenant_id and role in ["admin", "security_admin", "auditor"]
  end
  
  @doc """
  Determines if the user can export audit logs.
  """
  def authorize(:export_audit_logs, %{tenant_id: user_tenant_id, role: role}, %{tenant_id: tenant_id}) do
    # Only allow export of logs from the same tenant and only for admin or security_admin
    user_tenant_id == tenant_id and role in ["admin", "security_admin"]
  end
  
  @doc """
  Determines if the user can view security events.
  """
  def authorize(:view_security_events, %{tenant_id: user_tenant_id, role: role}, %{tenant_id: tenant_id}) do
    # Only allow access to security events from the same tenant
    user_tenant_id == tenant_id and role in ["admin", "security_admin"]
  end
  
  @doc """
  Determines if the user can resolve security events.
  """
  def authorize(:resolve_security_events, %{tenant_id: user_tenant_id, role: role}, %{tenant_id: tenant_id}) do
    # Only allow resolution of security events from the same tenant
    user_tenant_id == tenant_id and role in ["admin", "security_admin"]
  end
  
  # Default deny for all other actions
  def authorize(_, _, _), do: false
end
```

### 2. Audit Log Access API

Implementing API for accessing audit logs:

```elixir
defmodule AybizaWeb.AuditLogController do
  use AybizaWeb, :controller
  
  alias Aybiza.Security.AuditLogs
  alias Aybiza.Security.AuditLogger
  
  # Ensure only authorized users can access audit logs
  plug :authorize_audit_log_access
  
  def index(conn, params) do
    tenant_id = conn.assigns.tenant_id
    current_user = conn.assigns.current_user
    
    # Log the access to audit logs
    AuditLogger.log_event(tenant_id, current_user.id, "view_audit_logs", %{
      filter_params: sanitize_params(params),
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn)
    })
    
    # Paginate audit logs
    page = AuditLogs.list_audit_logs(tenant_id, params)
    render(conn, :index, page: page)
  end
  
  def export(conn, params) do
    tenant_id = conn.assigns.tenant_id
    current_user = conn.assigns.current_user
    
    # Log the export of audit logs
    AuditLogger.log_event(tenant_id, current_user.id, "export_audit_logs", %{
      filter_params: sanitize_params(params),
      format: params["format"] || "csv",
      ip_address: get_client_ip(conn),
      user_agent: get_user_agent(conn)
    })
    
    # Generate export file
    {:ok, file_path} = AuditLogs.export_audit_logs(tenant_id, params)
    
    # Stream the file to the client
    conn
    |> put_resp_content_type("text/csv")
    |> put_resp_header("content-disposition", "attachment; filename=\"audit_logs.csv\"")
    |> send_file(200, file_path)
  end
  
  # Helper functions
  
  defp authorize_audit_log_access(conn, _) do
    tenant_id = conn.assigns.tenant_id
    current_user = conn.assigns.current_user
    
    case Bodyguard.permit(Aybiza.Security.AuditLogPolicy, :view_audit_logs, current_user, %{tenant_id: tenant_id}) do
      :ok ->
        conn
      {:error, :unauthorized} ->
        conn
        |> put_status(:forbidden)
        |> put_view(json: AybizaWeb.ErrorJSON)
        |> render(:"403", %{message: "Unauthorized access to audit logs"})
        |> halt()
    end
  end
  
  defp get_client_ip(conn) do
    conn.remote_ip
    |> Tuple.to_list()
    |> Enum.join(".")
  end
  
  defp get_user_agent(conn) do
    case get_req_header(conn, "user-agent") do
      [user_agent] -> user_agent
      _ -> nil
    end
  end
  
  defp sanitize_params(params) do
    # Remove sensitive data from params before logging
    params
    |> Map.drop(["password", "token", "secret"])
  end
end
```

## Audit Log Retention and Archiving

### 1. Retention Policy

Implementing audit log retention policies:

```elixir
defmodule Aybiza.Security.AuditLogRetention do
  @moduledoc """
  Manages retention and archiving of audit logs.
  """
  
  alias Aybiza.Repo
  alias Aybiza.Security.AuditLog
  require Logger
  
  @doc """
  Archives audit logs older than the specified retention period.
  """
  def archive_old_logs(tenant_id, options \\ []) do
    # Default retention period is 90 days
    retention_days = options[:retention_days] || 90
    
    # Calculate cutoff date
    cutoff_date = DateTime.utc_now() |> DateTime.add(-retention_days * 86400, :second)
    
    # Query logs to archive
    logs_to_archive = AuditLog
                      |> where([l], l.tenant_id == ^tenant_id)
                      |> where([l], l.event_time < ^cutoff_date)
                      |> limit(1000) # Process in batches
                      |> Repo.all()
    
    # Archive logs
    if length(logs_to_archive) > 0 do
      # Export logs to archive storage
      {:ok, archive_path} = export_logs_to_archive(tenant_id, logs_to_archive)
      
      # Delete archived logs
      {count, _} = Repo.delete_all(
        from l in AuditLog,
        where: l.id in ^Enum.map(logs_to_archive, & &1.id)
      )
      
      # Log the archival
      Logger.info("Archived #{count} audit logs for tenant #{tenant_id} to #{archive_path}")
      
      # Return success with count
      {:ok, count, archive_path}
    else
      # No logs to archive
      {:ok, 0, nil}
    end
  end
  
  @doc """
  Gets the current retention policy for a tenant.
  """
  def get_retention_policy(tenant_id) do
    # Fetch tenant-specific retention policy
    case Aybiza.Tenants.get_tenant(tenant_id) do
      %{audit_log_retention_days: days} when is_integer(days) ->
        {:ok, days}
      _ ->
        # Default to 90 days
        {:ok, 90}
    end
  end
  
  @doc """
  Updates the retention policy for a tenant.
  """
  def update_retention_policy(tenant_id, retention_days) when is_integer(retention_days) and retention_days > 0 do
    # Update tenant-specific retention policy
    Aybiza.Tenants.update_tenant(tenant_id, %{audit_log_retention_days: retention_days})
  end
  
  # Private helper functions
  
  defp export_logs_to_archive(tenant_id, logs) do
    # Generate archive file
    archive_path = "archives/audit_logs/#{tenant_id}/#{DateTime.utc_now() |> DateTime.to_iso8601()}.json"
    
    # Ensure directory exists
    File.mkdir_p!(Path.dirname(archive_path))
    
    # Write logs to file
    logs_json = Jason.encode!(logs)
    File.write!(archive_path, logs_json)
    
    # Upload to S3 or other long-term storage
    upload_to_s3(tenant_id, archive_path, logs_json)
    
    {:ok, archive_path}
  end
  
  defp upload_to_s3(tenant_id, archive_path, logs_json) do
    # Upload to S3 bucket
    # Implementation details...
  end
end
```

### 2. Scheduled Archiving

Implementing scheduled archiving of audit logs:

```elixir
defmodule Aybiza.Scheduler.AuditLogArchiver do
  @moduledoc """
  Scheduled task for archiving audit logs.
  """
  
  use Oban.Worker, queue: :archiving
  
  alias Aybiza.Security.AuditLogRetention
  
  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"tenant_id" => tenant_id}}) do
    # Get retention policy for tenant
    {:ok, retention_days} = AuditLogRetention.get_retention_policy(tenant_id)
    
    # Archive old logs
    case AuditLogRetention.archive_old_logs(tenant_id, retention_days: retention_days) do
      {:ok, count, _archive_path} ->
        # Log success
        if count > 0 do
          {:ok, %{archived_count: count}}
        else
          {:ok, %{message: "No logs to archive"}}
        end
      
      {:error, reason} ->
        # Handle error
        {:error, reason}
    end
  end
  
  @doc """
  Schedules audit log archiving for all tenants.
  """
  def schedule_for_all_tenants do
    # Get all tenant IDs
    tenant_ids = Aybiza.Tenants.list_all_tenant_ids()
    
    # Schedule archiving job for each tenant
    Enum.each(tenant_ids, fn tenant_id ->
      %{tenant_id: tenant_id}
      |> __MODULE__.new()
      |> Oban.insert()
    end)
  end
end
```

## Monitoring and Alerting

### 1. Security Event Monitoring

Implementing security event monitoring:

```elixir
defmodule Aybiza.Security.SecurityEventMonitor do
  @moduledoc """
  Monitors security events and triggers alerts.
  """
  
  alias Aybiza.Security.SecurityEvent
  alias Aybiza.Notifications.Alert
  
  @doc """
  Monitors security events and triggers alerts as needed.
  """
  def monitor_security_events do
    # Get unresolved critical security events
    critical_events = SecurityEvent
                     |> where([e], e.severity == "critical")
                     |> where([e], e.resolved == false)
                     |> order_by([e], desc: e.event_time)
                     |> Repo.all()
    
    # Trigger alerts for critical events
    Enum.each(critical_events, &trigger_alert/1)
    
    # Get unresolved warning security events
    warning_events = SecurityEvent
                    |> where([e], e.severity == "warning")
                    |> where([e], e.resolved == false)
                    |> order_by([e], desc: e.event_time)
                    |> Repo.all()
    
    # Group warnings by type for summary alerts
    warning_groups = Enum.group_by(warning_events, & &1.event_type)
    
    # Trigger summary alerts for warning groups
    Enum.each(warning_groups, fn {event_type, events} ->
      trigger_summary_alert(event_type, events)
    end)
    
    # Return counts
    %{
      critical_count: length(critical_events),
      warning_count: length(warning_events)
    }
  end
  
  # Private helper functions
  
  defp trigger_alert(event) do
    # Check if alert already sent
    unless alert_already_sent?(event) do
      # Create alert
      Alert.create(%{
        tenant_id: event.tenant_id,
        severity: "high",
        title: "Critical Security Event: #{event.event_type}",
        message: event.description,
        source: "security_monitor",
        reference_id: event.id,
        reference_type: "security_event"
      })
      
      # Update event with alert_sent flag
      SecurityEvent.update(event, %{alert_sent: true})
    end
  end
  
  defp trigger_summary_alert(event_type, events) do
    # Group by tenant
    tenant_groups = Enum.group_by(events, & &1.tenant_id)
    
    # Create summary alert for each tenant
    Enum.each(tenant_groups, fn {tenant_id, tenant_events} ->
      # Check if summary alert already sent
      unless summary_alert_already_sent?(tenant_id, event_type) do
        # Create summary alert
        Alert.create(%{
          tenant_id: tenant_id,
          severity: "medium",
          title: "Security Warning: Multiple #{event_type} Events",
          message: "#{length(tenant_events)} #{event_type} events detected",
          source: "security_monitor",
          reference_type: "security_event_summary",
          details: %{
            event_type: event_type,
            count: length(tenant_events),
            event_ids: Enum.map(tenant_events, & &1.id)
          }
        })
        
        # Update events with alert_sent flag
        event_ids = Enum.map(tenant_events, & &1.id)
        Repo.update_all(
          from e in SecurityEvent,
          where: e.id in ^event_ids,
          set: [alert_sent: true]
        )
      end
    end)
  end
  
  defp alert_already_sent?(event) do
    event.alert_sent
  end
  
  defp summary_alert_already_sent?(tenant_id, event_type) do
    # Check if summary alert sent in the last 24 hours
    one_day_ago = DateTime.utc_now() |> DateTime.add(-86400, :second)
    
    Alert
    |> where([a], a.tenant_id == ^tenant_id)
    |> where([a], a.source == "security_monitor")
    |> where([a], a.reference_type == "security_event_summary")
    |> where([a], fragment("(?)::jsonb->>'event_type' = ?", a.details, ^event_type))
    |> where([a], a.inserted_at >= ^one_day_ago)
    |> Repo.exists?()
  end
end
```

### 2. Audit Log Metrics

Implementing audit log metrics for monitoring:

```elixir
defmodule Aybiza.Security.AuditLogMetrics do
  @moduledoc """
  Collects metrics from audit logs for monitoring.
  """
  
  alias Aybiza.Repo
  alias Aybiza.Security.AuditLog
  
  @doc """
  Collects metrics for a tenant over a specified time period.
  """
  def collect_metrics(tenant_id, period \\ :day) do
    # Calculate start time based on period
    start_time = case period do
      :hour -> DateTime.utc_now() |> DateTime.add(-3600, :second)
      :day -> DateTime.utc_now() |> DateTime.add(-86400, :second)
      :week -> DateTime.utc_now() |> DateTime.add(-604800, :second)
      :month -> DateTime.utc_now() |> DateTime.add(-2592000, :second)
    end
    
    # Basic counts
    total_count = count_logs(tenant_id, start_time)
    
    # Event type counts
    event_types = count_by_event_type(tenant_id, start_time)
    
    # User activity
    user_activity = count_by_actor(tenant_id, start_time)
    
    # Security events
    security_events = count_security_events(tenant_id, start_time)
    
    # Return metrics
    %{
      period: period,
      total_count: total_count,
      event_types: event_types,
      user_activity: user_activity,
      security_events: security_events
    }
  end
  
  # Private helper functions
  
  defp count_logs(tenant_id, start_time) do
    AuditLog
    |> where([l], l.tenant_id == ^tenant_id)
    |> where([l], l.event_time >= ^start_time)
    |> select([l], count(l.id))
    |> Repo.one()
  end
  
  defp count_by_event_type(tenant_id, start_time) do
    AuditLog
    |> where([l], l.tenant_id == ^tenant_id)
    |> where([l], l.event_time >= ^start_time)
    |> group_by([l], l.event_type)
    |> select([l], {l.event_type, count(l.id)})
    |> Repo.all()
    |> Map.new()
  end
  
  defp count_by_actor(tenant_id, start_time) do
    AuditLog
    |> where([l], l.tenant_id == ^tenant_id)
    |> where([l], l.event_time >= ^start_time)
    |> where([l], not is_nil(l.actor_id))
    |> group_by([l], l.actor_id)
    |> select([l], {l.actor_id, count(l.id)})
    |> Repo.all()
    |> Enum.sort_by(fn {_, count} -> count end, :desc)
    |> Enum.take(10)
    |> Map.new()
  end
  
  defp count_security_events(tenant_id, start_time) do
    Aybiza.Security.SecurityEvent
    |> where([e], e.tenant_id == ^tenant_id)
    |> where([e], e.event_time >= ^start_time)
    |> group_by([e], e.severity)
    |> select([e], {e.severity, count(e.id)})
    |> Repo.all()
    |> Map.new()
  end
end
```

## Testing

### 1. Unit Testing Audit Logging

Example of unit testing audit logging:

```elixir
defmodule Aybiza.Security.AuditLoggerTest do
  use Aybiza.DataCase
  
  alias Aybiza.Security.AuditLogger
  alias Aybiza.Security.AuditLog
  
  describe "log_event/4" do
    test "successfully logs an event" do
      tenant = insert(:tenant)
      user = insert(:user, tenant: tenant)
      
      event_type = "test_event"
      details = %{test: "data"}
      
      assert {:ok, log} = AuditLogger.log_event(tenant.id, user.id, event_type, details)
      
      # Verify log was created
      assert log.tenant_id == tenant.id
      assert log.actor_id == user.id
      assert log.event_type == event_type
      assert log.details["test"] == "data"
    end
    
    test "sanitizes sensitive data in details" do
      tenant = insert(:tenant)
      
      details = %{
        password: "secret123",
        token: "abcdef",
        api_key: "xyz123",
        safe_data: "public"
      }
      
      {:ok, log} = AuditLogger.log_event(tenant.id, nil, "test_event", details)
      
      # Verify sensitive data was sanitized
      assert log.details["password"] == "[REDACTED]"
      assert log.details["token"] == "[REDACTED]"
      assert log.details["api_key"] == "[REDACTED]"
      assert log.details["safe_data"] == "public"
    end
    
    test "creates security event for relevant event types" do
      tenant = insert(:tenant)
      
      # Login failure should trigger security event
      {:ok, log} = AuditLogger.log_event(tenant.id, nil, "login_failure", %{})
      
      # Verify security event was created
      security_event = Repo.get_by(Aybiza.Security.SecurityEvent, related_audit_log_id: log.id)
      assert security_event != nil
      assert security_event.event_type == "login_failure"
      assert security_event.severity == "warning"
    end
  end
  
  describe "log_data_access/5" do
    test "successfully logs data access" do
      tenant = insert(:tenant)
      user = insert(:user, tenant: tenant)
      
      resource_type = "patient"
      resource_id = "123"
      access_type = "view"
      
      assert {:ok, log} = AuditLogger.log_data_access(tenant.id, user.id, resource_type, resource_id, access_type)
      
      # Verify log was created
      assert log.tenant_id == tenant.id
      assert log.actor_id == user.id
      assert log.event_type == "data_access"
      assert log.resource_type == resource_type
      assert log.resource_id == resource_id
      assert log.details["access_type"] == access_type
    end
  end
end
```

### 2. Integration Testing Audit Logging

Example of integration testing audit logging:

```elixir
defmodule AybizaWeb.AuditLoggingIntegrationTest do
  use AybizaWeb.ConnCase
  
  alias Aybiza.Security.AuditLog
  
  setup %{conn: conn} do
    tenant = insert(:tenant)
    user = insert(:user, tenant: tenant, role: "admin")
    
    # Authenticate user
    conn = conn
           |> log_in_user(user)
           |> put_req_header("user-agent", "test-agent")
    
    %{conn: conn, tenant: tenant, user: user}
  end
  
  describe "API endpoints with audit logging" do
    test "logs access to call listing", %{conn: conn, tenant: tenant, user: user} do
      # Make request
      get(conn, Routes.call_path(conn, :index))
      
      # Verify audit log was created
      log = Repo.get_by(AuditLog, %{
        tenant_id: tenant.id,
        actor_id: user.id,
        event_type: "list_calls"
      })
      
      assert log != nil
      assert log.details["user_agent"] == "test-agent"
    end
    
    test "logs successful call creation", %{conn: conn, tenant: tenant, user: user} do
      # Create valid call params
      call_params = %{
        "call" => %{
          "phone_number" => "+15551234567",
          "status" => "in_progress"
        }
      }
      
      # Make request
      post(conn, Routes.call_path(conn, :create), call_params)
      
      # Verify audit logs were created - both attempt and success
      attempt_log = Repo.get_by(AuditLog, %{
        tenant_id: tenant.id,
        actor_id: user.id,
        event_type: "create_call_attempt"
      })
      
      success_log = Repo.get_by(AuditLog, %{
        tenant_id: tenant.id,
        actor_id: user.id,
        event_type: "data_modification"
      })
      
      assert attempt_log != nil
      assert success_log != nil
      assert success_log.details["modification_type"] == "create"
    end
    
    test "logs failed authentication attempt", %{conn: conn} do
      # Create invalid credentials
      credentials = %{
        "email" => "nonexistent@example.com",
        "password" => "wrongpassword"
      }
      
      # Make request
      post(conn, Routes.user_session_path(conn, :create), credentials)
      
      # Verify audit log was created
      log = Repo.get_by(AuditLog, %{
        event_type: "login_failure"
      })
      
      assert log != nil
      assert log.details["reason"] == "invalid_credentials"
    end
  end
end
```

## Security Compliance Considerations

### 1. SOC 2 Compliance

SOC 2 compliance requirements for audit logging:

- **Complete Audit Trail**: All user actions must be logged
- **User Attribution**: All logs must include user identification
- **Access Controls**: Audit logs must have proper access controls
- **Immutability**: Audit logs must be immutable
- **Retention Policy**: Audit logs must be retained according to policy
- **Monitoring**: Security events must be monitored with alerting

### 2. HIPAA Compliance

HIPAA compliance requirements for audit logging:

- **PHI Access Logging**: All access to PHI must be logged
- **Detailed Attribution**: Logs must include who, what, when, where, and how
- **Tamper Protection**: Logs must be protected from tampering
- **Extended Retention**: Logs must be retained for at least 6 years
- **Access Reports**: Capability to generate access reports for patients

### 3. GDPR Compliance

GDPR compliance requirements for audit logging:

- **Processing Logs**: All data processing activities must be logged
- **Data Access Logs**: Access to personal data must be logged
- **Consent Verification**: Verification of consent must be logged
- **Right to Access Reports**: Capability to provide data subject access reports
- **Data Deletion Logs**: Deletion of personal data must be logged

## Best Practices

### 1. General Best Practices

- **Log All Security-Relevant Events**: Ensure all security-relevant events are logged
- **Include Tenant Context**: Always include tenant_id in all audit logs
- **Sanitize Sensitive Data**: Never log sensitive data like passwords or tokens
- **Use Structured Logging**: Use consistent, structured log formats
- **Include Context Details**: Include IP addresses, user agents, and other context

### 2. Implementation Patterns

- **Use AuditLogger in Controllers**: Log all API access in controllers
- **Use PaperTrail for Data Changes**: Track all data changes with PaperTrail
- **Log Authentication Events**: Use specialized AuthAuditLogger for auth events
- **Monitor Security Events**: Implement real-time monitoring of security events
- **Implement Access Controls**: Ensure proper access controls for audit logs

### 3. Performance Considerations

- **Async Logging**: Use async logging for performance-critical paths
- **Batch Processing**: Use batch processing for log analysis
- **Indexing**: Ensure proper database indexing for log queries
- **Log Rotation**: Implement log rotation for database performance
- **Optimized Queries**: Optimize queries for accessing audit logs

## Conclusion

Proper audit logging is critical for security compliance and incident response in the AYBIZA platform. By following this implementation guide, you can ensure that your code meets the comprehensive audit logging requirements of the platform, supporting security compliance across multiple regulatory frameworks.

## References

- [Security Compliance Implementation](09_security_compliance_implementation.md)
- [Observability and Monitoring Guide](17_observability_monitoring_guide.md)
- [Authentication & Identity Service](30_authentication_identity_service.md)
- [Compliance Verification Checklist](46_compliance_verification_checklist.md)
- [Multi-Tenant Isolation Patterns](45_multi_tenant_isolation_patterns.md)