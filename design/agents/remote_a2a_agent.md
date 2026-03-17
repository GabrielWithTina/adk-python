# RemoteA2aAgent — Agent-to-Agent Protocol Client

**Source:** `src/google/adk/agents/remote_a2a_agent.py`

## Purpose

`RemoteA2aAgent` enables an ADK agent to communicate with remote agents via the Agent-to-Agent (A2A) protocol. It handles agent card resolution, HTTP client management, message conversion between ADK events and A2A messages, streaming responses, and session state tracking across requests.

## Class Overview

```mermaid
classDiagram
    class RemoteA2aAgent {
        -_agent_card: Optional[AgentCard]
        -_agent_card_source: Optional[str]
        -_a2a_client: Optional[A2AClient]
        -_httpx_client: Optional[AsyncClient]
        -_timeout: float
        -_is_resolved: bool
        -_genai_part_converter: GenAIPartToA2APartConverter
        -_a2a_part_converter: A2APartToGenAIPartConverter
        -_a2a_client_factory: Optional[A2AClientFactory]
        -_full_history_when_stateless: bool
        -_config: A2aRemoteAgentConfig

        #_run_async_impl(ctx) → AsyncGenerator[Event]
        -_ensure_resolved()
        -_ensure_httpx_client() → AsyncClient
        -_resolve_agent_card() → AgentCard
        -_handle_a2a_response(response, ctx) → Optional[Event]
        -_handle_a2a_response_v2(response, ctx) → Optional[Event]
        -_construct_message_parts_from_session(ctx) → tuple
        -_create_a2a_request_for_user_function_response(ctx) → Optional[A2AMessage]
        +cleanup()
    }

    class BaseAgent {
        +name: str
    }

    BaseAgent <|-- RemoteA2aAgent
```

## Agent Card Resolution

The agent card can be provided in three ways:

```mermaid
flowchart TD
    A["RemoteA2aAgent(agent_card=...)"] --> B{Type?}

    B -- "AgentCard object" --> C["Use directly"]
    B -- "URL string (http/https)" --> D["_resolve_agent_card_from_url()"]
    B -- "File path string" --> E["_resolve_agent_card_from_file()"]

    D --> F["A2ACardResolver.get_agent_card()"]
    E --> G["Read JSON file, parse AgentCard"]

    C & F & G --> H["_validate_agent_card()"]
    H --> I["Check URL is valid"]
    I --> J["Auto-populate description if empty"]
    J --> K["Initialize A2A client"]
```

Resolution is lazy — happens on first `_run_async_impl` call.

## Request Construction

```mermaid
flowchart TD
    A["_run_async_impl(ctx)"] --> B["_ensure_resolved()"]
    B --> C["_create_a2a_request_for_user_function_response(ctx)"]
    C --> D{Function response found?}

    D -- Yes --> E["Convert function response event to A2AMessage<br/>Attach task_id, context_id from metadata"]

    D -- No --> F["_construct_message_parts_from_session(ctx)"]
    F --> G["Walk session events in reverse"]
    G --> H{Hit previous remote response?}
    H -- Yes, stateful --> I["Stop (remote has history)"]
    H -- Yes, stateless + full_history --> J["Continue collecting"]
    H -- No --> K["Convert event parts to A2A parts"]
    K --> G

    E & I & J --> L["Execute interceptors"]
    L --> M["a2a_client.send_message(request)"]
```

## Response Handling

```mermaid
flowchart TD
    A["A2A Response"] --> B{Response type?}

    B -- "tuple (Task, Update)" --> C{Update type?}
    C -- "None (initial/complete)" --> D["convert_a2a_task_to_event()"]
    C -- "TaskStatusUpdateEvent" --> E["convert_a2a_message_to_event()"]
    C -- "TaskArtifactUpdateEvent" --> F{Full artifact?}
    F -- Yes --> G["convert_a2a_task_to_event()"]
    F -- No --> H["Skip (partial update)"]

    B -- "A2AMessage" --> I["convert_a2a_message_to_event()"]

    D & E & G & I --> J["Attach metadata (task_id, context_id)"]
    J --> K["Mark in-progress as thought"]

    subgraph "V2 Response Handler"
        V2["Uses config converters instead of hardcoded functions"]
    end
```

### Streaming Task States

| Task State | Event Treatment |
|-----------|----------------|
| `submitted` / `working` | Parts marked as `thought=True` |
| `completed` | Normal content |
| Artifact update (full) | Converted to event |
| Artifact update (partial) | Skipped |

## Metadata Flow

```mermaid
flowchart LR
    subgraph "Outbound Metadata"
        REQ["a2a:request → full A2A message dump"]
    end

    subgraph "Response Metadata"
        TASK_ID["a2a:task_id"]
        CTX_ID["a2a:context_id"]
        RESP["a2a:response → full task/message dump"]
    end

    subgraph "Error Metadata"
        ERR["a2a:error → error message"]
        STATUS["a2a:status_code → HTTP status"]
    end
```

All metadata is stored in `event.custom_metadata` with the `a2a:` prefix.

## Stateful vs Stateless Agents

```mermaid
flowchart TD
    A["Construct message from session"] --> B["Walk events in reverse"]
    B --> C{Previous remote response?}
    C -- Yes --> D{Has context_id?}

    D -- Yes --> E["Stateful: Stop here<br/>(remote has the history)"]
    D -- No --> F{full_history_when_stateless?}
    F -- Yes --> G["Continue collecting all events"]
    F -- No --> H["Stop here (legacy behavior)"]
```

## Interceptor Pipeline

```mermaid
flowchart TD
    A["Before request interceptors"] --> B["Modify A2AMessage or return Event"]
    B --> C{Returned Event?}
    C -- Yes --> D["Yield event, skip remote call"]
    C -- No --> E["Send to remote"]
    E --> F["After request interceptors"]
    F --> G["Modify Event or return None"]
    G --> H{None?}
    H -- Yes --> I["Skip event"]
    H -- No --> J["Yield event"]
```

The `use_legacy=False` flag injects a `_new_integration_extension_interceptor` that signals the server to use the new integration path.

## Error Handling

```mermaid
flowchart TD
    A["_run_async_impl"] --> B{Error type?}
    B -- "Resolution failure" --> C["Yield error Event, return"]
    B -- "A2AClientHTTPError" --> D["Yield error Event with status_code"]
    B -- "Other Exception" --> E["Yield error Event"]

    C & D & E --> F["Error logged, metadata attached"]
```

## Resource Cleanup

```mermaid
flowchart TD
    A["cleanup()"] --> B{_httpx_client_needs_cleanup?}
    B -- Yes --> C["httpx_client.aclose()"]
    B -- No --> EXIT

    C --> D{Success?}
    D -- Yes --> E["Set _httpx_client = None"]
    D -- No --> F["Log warning, set None"]
```

The HTTP client is only cleaned up if the agent created it (not if it was injected externally).

## Live Mode

`_run_live_impl` raises `NotImplementedError` — live A2A streaming is not supported.
