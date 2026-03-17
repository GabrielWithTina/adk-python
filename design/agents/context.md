# Context & ReadonlyContext — Runtime Context for Callbacks and Tools

**Source:** `src/google/adk/agents/context.py`, `src/google/adk/agents/readonly_context.py`

## Purpose

`ReadonlyContext` provides a read-only view of invocation state for tools and instruction providers. `Context` extends it with mutation capabilities — state changes, artifact management, credential handling, memory operations, and UI widget rendering. Together they form the primary interface callbacks and tools use to interact with the runtime.

## Class Hierarchy

```mermaid
classDiagram
    class ReadonlyContext {
        #_invocation_context: InvocationContext

        +user_content → Optional[Content]
        +invocation_id → str
        +agent_name → str
        +state → MappingProxyType (read-only)
        +session → Session
        +user_id → str
        +run_config → Optional[RunConfig]
    }

    class Context {
        -_event_actions: EventActions
        -_state: State (delta-aware)
        -_function_call_id: Optional[str]
        -_tool_confirmation: Optional[ToolConfirmation]

        +state → State (read-write)
        +actions → EventActions
        +function_call_id → str

        +load_artifact(filename, version) → Part
        +save_artifact(filename, artifact) → int
        +get_artifact_version(filename) → ArtifactVersion
        +list_artifacts() → list[str]

        +save_credential(auth_config)
        +load_credential(auth_config) → AuthCredential
        +get_auth_response(auth_config) → AuthCredential
        +request_credential(auth_config)

        +request_confirmation(hint, payload)

        +add_session_to_memory()
        +add_events_to_memory(events)
        +add_memory(memories)
        +search_memory(query) → SearchMemoryResponse

        +render_ui_widget(ui_widget)
    }

    ReadonlyContext <|-- Context
    Context .. CallbackContext : "alias"
```

`CallbackContext` is a backward-compatible alias: `CallbackContext = Context`.

## ReadonlyContext — Read-Only View

```mermaid
flowchart LR
    subgraph "ReadonlyContext Properties"
        direction TB
        P1["user_content → invocation_context.user_content"]
        P2["invocation_id → invocation_context.invocation_id"]
        P3["agent_name → invocation_context.agent.name"]
        P4["state → MappingProxyType(session.state)"]
        P5["session → invocation_context.session"]
        P6["user_id → invocation_context.user_id"]
        P7["run_config → invocation_context.run_config"]
    end
```

The `state` property returns a `MappingProxyType` — a read-only dict wrapper that prevents mutations.

**Used by:** `InstructionProvider` callables, tool `get_tools_with_prefix()`.

## Context — Read-Write Operations

### State Management

```mermaid
flowchart TD
    A["Context.__init__()"] --> B["Create EventActions"]
    B --> C["Create State(value=session.state, delta=event_actions.state_delta)"]

    D["ctx.state['key'] = 'value'"] --> E["State.__setitem__"]
    E --> F["Updates session.state"]
    E --> G["Records delta in event_actions"]
```

The `State` object is a dict-like wrapper that tracks all mutations as a delta. These deltas are carried in `EventActions` and persisted with events.

### Capability Map

```mermaid
flowchart TB
    subgraph "Artifact Methods"
        ART_LOAD["load_artifact(filename, version)"]
        ART_SAVE["save_artifact(filename, artifact)"]
        ART_VER["get_artifact_version(filename)"]
        ART_LIST["list_artifacts()"]
    end

    subgraph "Credential Methods"
        CRED_SAVE["save_credential(auth_config)"]
        CRED_LOAD["load_credential(auth_config)"]
        CRED_AUTH["get_auth_response(auth_config)"]
        CRED_REQ["request_credential(auth_config)"]
    end

    subgraph "Memory Methods"
        MEM_SESS["add_session_to_memory()"]
        MEM_EVT["add_events_to_memory(events)"]
        MEM_ADD["add_memory(memories)"]
        MEM_SEARCH["search_memory(query)"]
    end

    subgraph "Tool Methods"
        TOOL_CONF["request_confirmation(hint, payload)"]
    end

    subgraph "UI Methods"
        UI_RENDER["render_ui_widget(ui_widget)"]
    end

    ART_SAVE --> DELTA["Records artifact_delta"]
    CRED_REQ --> AUTH_DELTA["Records requested_auth_configs"]
    TOOL_CONF --> CONF_DELTA["Records requested_tool_confirmations"]
    UI_RENDER --> UI_DELTA["Records render_ui_widgets"]
```

### Artifact Operations

```mermaid
sequenceDiagram
    participant Tool as Tool / Callback
    participant Ctx as Context
    participant AS as ArtifactService

    Tool->>Ctx: save_artifact("report.pdf", data)
    Ctx->>AS: save_artifact(app, user, session, filename, data)
    AS-->>Ctx: version number
    Ctx->>Ctx: event_actions.artifact_delta["report.pdf"] = version
    Ctx-->>Tool: version

    Tool->>Ctx: load_artifact("report.pdf")
    Ctx->>AS: load_artifact(app, user, session, filename)
    AS-->>Ctx: Part data
    Ctx-->>Tool: Part
```

### Credential Flow

```mermaid
flowchart TD
    A["Tool needs OAuth token"] --> B["ctx.request_credential(auth_config)"]
    B --> C{function_call_id set?}
    C -- No --> ERR["Raise ValueError"]
    C -- Yes --> D["AuthHandler.generate_auth_request()"]
    D --> E["Store in event_actions.requested_auth_configs"]
    E --> F["Event sent to client for user auth"]

    G["User completes auth flow"] --> H["Auth response stored in session state"]
    H --> I["Next invocation: ctx.get_auth_response()"]
    I --> J["AuthHandler extracts credential from state"]
```

Two paths:
- **Tool context** (`function_call_id` set): Use `request_credential()` for interactive OAuth
- **Callback context**: Use `save_credential()` / `load_credential()` for direct credential management

### Memory Operations

| Method | Purpose |
|--------|---------|
| `add_session_to_memory()` | Save entire current session to memory service |
| `add_events_to_memory(events)` | Save specific events to memory |
| `add_memory(memories)` | Save explicit `MemoryEntry` items |
| `search_memory(query)` | Search across stored memories |

All methods delegate to the configured `BaseMemoryService` and raise `ValueError` if no memory service is available.

### Tool Confirmation

```mermaid
flowchart TD
    A["Tool execution"] --> B["ctx.request_confirmation(hint, payload)"]
    B --> C{function_call_id set?}
    C -- No --> ERR["Raise ValueError"]
    C -- Yes --> D["Create ToolConfirmation(hint, payload)"]
    D --> E["Store in event_actions.requested_tool_confirmations"]
    E --> F["Invocation pauses, waits for user confirmation"]
```

### UI Widget Rendering

```mermaid
flowchart TD
    A["ctx.render_ui_widget(widget)"] --> B{Widget ID already exists?}
    B -- Yes --> ERR["Raise ValueError (duplicate ID)"]
    B -- No --> C["Append to event_actions.render_ui_widgets"]
```

UI widgets provide rendering metadata for rich interactive components (e.g., MCP App iframes).

## Usage Contexts

| Context Type | Used By | State Access |
|-------------|---------|-------------|
| `ReadonlyContext` | `InstructionProvider`, `BaseToolset.get_tools_with_prefix()` | Read-only |
| `Context` (as `CallbackContext`) | `before/after_agent_callback` | Read-write (no function_call_id) |
| `Context` (as `ToolContext` subclass) | Tool execution | Read-write (with function_call_id) |
