# BaseAgent — Foundation for All Agents

**Source:** `src/google/adk/agents/base_agent.py`

## Purpose

`BaseAgent` is the abstract base class for every agent type in ADK. It defines the agent tree structure (parent/child relationships), the callback lifecycle, agent state management for resumable execution, and the `run_async`/`run_live` entry points that all concrete agents must implement.

## Class Overview

```mermaid
classDiagram
    class BaseAgent {
        +name: str
        +description: str
        +parent_agent: Optional[BaseAgent]
        +sub_agents: list[BaseAgent]
        +before_agent_callback: Optional[BeforeAgentCallback]
        +after_agent_callback: Optional[AfterAgentCallback]

        +run_async(parent_context) → AsyncGenerator[Event]
        +run_live(parent_context) → AsyncGenerator[Event]
        +clone(update) → BaseAgent
        +find_agent(name) → Optional[BaseAgent]
        +find_sub_agent(name) → Optional[BaseAgent]
        +from_config(config, path) → BaseAgent
        #_run_async_impl(ctx) → AsyncGenerator[Event]
        #_run_live_impl(ctx) → AsyncGenerator[Event]
        -_handle_before_agent_callback(ctx) → Optional[Event]
        -_handle_after_agent_callback(ctx) → Optional[Event]
        -_create_invocation_context(parent_ctx) → InvocationContext
        -_load_agent_state(ctx, state_type) → Optional[AgentState]
        -_create_agent_state_event(ctx) → Event
    }

    class BaseAgentState {
        <<experimental>>
    }

    class LlmAgent {
        +model: str | BaseLlm
        +instruction: str | InstructionProvider
        +tools: list[ToolUnion]
        ...callbacks...
    }

    class LoopAgent {
        +max_iterations: Optional[int]
    }

    class SequentialAgent
    class ParallelAgent
    class LangGraphAgent
    class RemoteA2aAgent

    BaseAgent <|-- LlmAgent
    BaseAgent <|-- LoopAgent
    BaseAgent <|-- SequentialAgent
    BaseAgent <|-- ParallelAgent
    BaseAgent <|-- LangGraphAgent
    BaseAgent <|-- RemoteA2aAgent
```

## Agent Lifecycle — `run_async()`

The `run_async()` method is `@final` — subclasses cannot override it. Instead, they implement `_run_async_impl()`. This ensures the callback lifecycle is always respected.

```mermaid
flowchart TD
    A["run_async(parent_context)"] --> B["Start tracing span"]
    B --> C["_create_invocation_context()"]
    C --> D["_handle_before_agent_callback()"]

    D --> E{Callback returned content?}
    E -- Yes --> F["yield event, end_invocation=True"]
    E -- No --> G{end_invocation?}
    G -- Yes --> EXIT["Return"]
    G -- No --> H["_run_async_impl(ctx)"]

    H --> I["yield events from impl"]
    I --> J{end_invocation?}
    J -- Yes --> EXIT
    J -- No --> K["_handle_after_agent_callback()"]
    K --> L{Callback returned content?}
    L -- Yes --> M["yield additional event"]
    L -- No --> EXIT

    style F fill:#fff3e0
    style M fill:#fff3e0
```

The same pattern applies to `run_live()` for bidirectional streaming.

## Callback System

```mermaid
flowchart TD
    subgraph "Before Agent Callback"
        BA1["Run plugin callbacks"]
        BA1 --> BA2{Plugin returned content?}
        BA2 -- No --> BA3["Run canonical callbacks (in order)"]
        BA3 --> BA4{Any callback returned content?}
        BA2 -- Yes --> BA5
        BA4 -- Yes --> BA5["Create Event with content, set end_invocation=True"]
        BA4 -- No --> BA6{State delta?}
        BA6 -- Yes --> BA7["Create state-only Event"]
        BA6 -- No --> BA8["Return None"]
    end
```

Key behaviors:
- Callbacks are called **in order** until one returns non-None content
- When a `before_agent_callback` returns content, the **agent run is skipped entirely**
- When an `after_agent_callback` returns content, it's **appended as an additional event**
- Both sync and async callbacks are supported (auto-awaited if async)
- Callback parameter must be named `callback_context` (enforced)

## Validation Rules

```mermaid
flowchart LR
    subgraph "Name Validation"
        V1["Must be valid Python identifier"]
        V2["Cannot be 'user' (reserved)"]
    end

    subgraph "Sub-agent Validation"
        V3["Names must be unique"]
        V4["Cannot have multiple parents"]
    end

    V1 & V2 --> PASS["Agent created"]
    V3 & V4 --> PASS
```

| Rule | Enforcement |
|------|-------------|
| Name is Python identifier | `@field_validator('name')` — raises `ValueError` |
| Name not `"user"` | `@field_validator('name')` — raises `ValueError` |
| Unique sub-agent names | `@field_validator('sub_agents')` — logs warning |
| Single parent constraint | `model_post_init` — raises `ValueError` if parent already set |

## Agent Tree Navigation

```mermaid
flowchart TD
    A["find_agent(name)"] --> B{self.name == name?}
    B -- Yes --> C["Return self"]
    B -- No --> D["find_sub_agent(name)"]
    D --> E["DFS through sub_agents"]
    E --> F{Found?}
    F -- Yes --> G["Return agent"]
    F -- No --> H["Return None"]
```

- `root_agent` property: Walks parent chain to find root
- `find_agent()`: Finds by name in self + descendants (DFS)
- `find_sub_agent()`: Finds by name in descendants only

## Clone Semantics

```mermaid
flowchart TD
    A["clone(update)"] --> B{update has 'parent_agent'?}
    B -- Yes --> ERR["Raise ValueError"]
    B -- No --> C["model_copy(update)"]
    C --> D["Shallow-copy list fields"]
    D --> E{sub_agents in update?}
    E -- No --> F["Recursively clone sub_agents"]
    E -- Yes --> G["Re-parent provided sub_agents"]
    F & G --> H["Set parent_agent = None"]
    H --> I["Return cloned agent"]
```

Clone creates a deep copy of the agent tree — sub-agents are recursively cloned to prevent shared state, and the cloned root has no parent.

## Agent State (Experimental)

`BaseAgentState` supports resumable execution for composite agents. State is serialized to JSON and stored in `InvocationContext.agent_states`.

```mermaid
stateDiagram-v2
    [*] --> Running : Agent starts
    Running --> Paused : should_pause_invocation()
    Paused --> Running : Invocation resumed
    Running --> Completed : end_of_agent=True
    Completed --> [*]

    note right of Paused
        agent_state persisted
        in InvocationContext
    end note
```

## Configuration System

Each agent type has a corresponding config class for YAML-based agent definition:

| Agent | Config Class | Key Fields |
|-------|-------------|------------|
| `BaseAgent` | `BaseAgentConfig` | `name`, `description`, `sub_agents`, callbacks |
| `LlmAgent` | `LlmAgentConfig` | `model`, `instruction`, `tools`, schemas |
| `LoopAgent` | `LoopAgentConfig` | `max_iterations` |
| `SequentialAgent` | `SequentialAgentConfig` | (inherits base) |
| `ParallelAgent` | `ParallelAgentConfig` | (inherits base) |

The `from_config()` classmethod constructs agents from these configs, resolving sub-agent references, callbacks, and tool references.
