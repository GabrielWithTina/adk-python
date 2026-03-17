# InvocationContext — Per-Invocation Execution State

**Source:** `src/google/adk/agents/invocation_context.py`

## Purpose

`InvocationContext` holds all the mutable and immutable state for a single invocation — from the user message through final response. It carries service references, session data, agent states for resumability, cost tracking, and configuration. Every agent in the call chain receives its own view of this context.

## Class Overview

```mermaid
classDiagram
    class InvocationContext {
        +invocation_id: str
        +branch: Optional[str]
        +agent: BaseAgent
        +session: Session
        +user_content: Optional[Content]

        +artifact_service: Optional[BaseArtifactService]
        +session_service: BaseSessionService
        +memory_service: Optional[BaseMemoryService]
        +credential_service: Optional[BaseCredentialService]

        +agent_states: dict[str, dict]
        +end_of_agents: dict[str, bool]
        +end_invocation: bool

        +run_config: Optional[RunConfig]
        +resumability_config: Optional[ResumabilityConfig]
        +events_compaction_config: Optional[EventsCompactionConfig]
        +plugin_manager: PluginManager

        +live_request_queue: Optional[LiveRequestQueue]
        +active_streaming_tools: Optional[dict]
        +transcription_cache: Optional[list]

        +set_agent_state(name, agent_state, end_of_agent)
        +reset_sub_agent_states(agent_name)
        +populate_invocation_agent_states()
        +increment_llm_call_count()
        +should_pause_invocation(event) → bool
        +is_resumable → bool
        +_get_events(...) → list[Event]
        +_find_matching_function_call(event) → Optional[Event]
    }

    class _InvocationCostManager {
        -_number_of_llm_calls: int
        +increment_and_enforce_llm_calls_limit(run_config)
    }

    InvocationContext *-- _InvocationCostManager
```

## Invocation Lifecycle

```mermaid
flowchart TD
    A["Runner creates InvocationContext"] --> B["Populate agent states from history"]
    B --> C["Agent.run_async(ctx)"]
    C --> D["Agent creates child ctx via _create_invocation_context()"]
    D --> E["_run_async_impl(child_ctx)"]
    E --> F["Events yielded back to Runner"]
    F --> G{Agent transfer?}
    G -- Yes --> D
    G -- No --> H["Invocation complete"]
```

## Invocation Structure

The docstring defines the precise execution model:

```
┌─────────────────────── invocation ──────────────────────────┐
┌──────────── llm_agent_call_1 ────────────┐ ┌─ agent_call_2 ─┐
┌──── step_1 ────────┐ ┌───── step_2 ──────┐
[call_llm] [call_tool] [call_llm] [transfer]
```

| Concept | Scope |
|---------|-------|
| **Invocation** | User message → final response. Contains 1+ agent calls. |
| **Agent call** | Single `agent.run()` execution. Ends when `run()` returns. |
| **Step** | One LLM call + its tool executions. |

## Agent State Management

```mermaid
flowchart TD
    A["set_agent_state(name)"] --> B{end_of_agent?}
    B -- Yes --> C["end_of_agents[name] = True<br/>Delete agent_states[name]"]
    B -- No --> D{agent_state provided?}
    D -- Yes --> E["agent_states[name] = state.model_dump()<br/>end_of_agents[name] = False"]
    D -- No --> F["Delete both entries<br/>(allows re-run)"]
```

```mermaid
flowchart TD
    A["reset_sub_agent_states(agent_name)"] --> B["Find agent by name"]
    B --> C["For each sub_agent"]
    C --> D["set_agent_state(sub_agent.name)<br/>(clears state)"]
    D --> E["Recurse into sub_agent's children"]
```

## Resumability

```mermaid
flowchart TD
    A["populate_invocation_agent_states()"] --> B{is_resumable?}
    B -- No --> EXIT["Return"]
    B -- Yes --> C["Iterate current invocation events"]
    C --> D{event.actions.end_of_agent?}
    D -- Yes --> E["end_of_agents[author] = True<br/>Delete agent_state"]
    D -- No --> F{agent_state in event?}
    F -- Yes --> G["Set agent_state<br/>end_of_agents = False"]
    F -- No --> H{Author not 'user' AND has content<br/>AND no existing state?}
    H -- Yes --> I["Set empty BaseAgentState"]
    H -- No --> C
```

### Pause vs End

```mermaid
flowchart TD
    A["should_pause_invocation(event)"] --> B{is_resumable?}
    B -- No --> FALSE["Return False"]
    B -- Yes --> C{Event has long_running_tool_ids<br/>AND function_calls?}
    C -- No --> FALSE
    C -- Yes --> D["Check if any function_call.id<br/>is in long_running_tool_ids"]
    D -- Yes --> TRUE["Return True (pause)"]
    D -- No --> FALSE
```

- **Pausing**: Invocation suspends, can resume later. Agent state is preserved.
- **Ending**: Invocation terminates. `end_invocation = True`.

## Branching

The `branch` field isolates parallel sub-agents from seeing each other's events:

```mermaid
flowchart TD
    PAR["ParallelAgent"] --> A["Branch: parent.agent_a"]
    PAR --> B["Branch: parent.agent_b"]

    A --> A_EVENTS["Events only visible<br/>within this branch"]
    B --> B_EVENTS["Events only visible<br/>within this branch"]
```

Format: `parent_agent.sub_agent` (dot-separated ancestry).

## LLM Call Cost Tracking

```mermaid
flowchart TD
    A["increment_llm_call_count()"] --> B["_number_of_llm_calls++"]
    B --> C{run_config.max_llm_calls > 0<br/>AND count > limit?}
    C -- Yes --> D["Raise LlmCallsLimitExceededError"]
    C -- No --> E["Continue"]
```

Default limit: **500 LLM calls** per invocation (configurable via `RunConfig.max_llm_calls`).

## Event Filtering

```mermaid
flowchart TD
    A["_get_events(current_invocation, current_branch)"] --> B["Start with session.events"]
    B --> C{current_invocation?}
    C -- Yes --> D["Filter by invocation_id"]
    C -- No --> E
    D --> E{current_branch?}
    E -- Yes --> F["Filter by branch"]
    E -- No --> G["Return results"]
    F --> G
```

## Key Properties

| Property | Source |
|----------|--------|
| `is_resumable` | `resumability_config.is_resumable` |
| `app_name` | `session.app_name` |
| `user_id` | `session.user_id` |
