# ParallelAgent — Concurrent Isolated Execution

**Source:** `src/google/adk/agents/parallel_agent.py`

## Purpose

`ParallelAgent` runs all its sub-agents concurrently with isolated branches. Each sub-agent operates in its own context branch, preventing cross-contamination of conversation history. Events are merged back via an async queue with backpressure control.

## Class Overview

```mermaid
classDiagram
    class ParallelAgent {
        #_run_async_impl(ctx) → AsyncGenerator[Event]
    }

    class BaseAgent {
        +sub_agents: list[BaseAgent]
    }

    BaseAgent <|-- ParallelAgent
```

## Execution Flow

```mermaid
flowchart TD
    A["_run_async_impl(ctx)"] --> B{sub_agents empty?}
    B -- Yes --> EXIT["Return"]
    B -- No --> C["Load agent state"]
    C --> D{Resumable AND no prior state?}
    D -- Yes --> E["Save initial BaseAgentState<br/>Yield state event"]
    D -- No --> F

    E --> F["Create branch contexts for each sub-agent"]
    F --> G["Filter out already-finished sub-agents"]
    G --> H["merge_func(agent_runs)"]

    H --> I["Yield merged events"]
    I --> J{should_pause_invocation?}
    J -- Yes --> PAUSE["Return"]
    J -- No --> I

    I -- "All done" --> K{All sub-agents ended?}
    K -- Yes --> L["Set end_of_agent, yield state event"]
    K -- No --> EXIT

    H -.->|finally| M["Close all agent_run generators"]
```

## Branch Isolation

```mermaid
flowchart TD
    A["ParallelAgent 'checker'"] --> B["_create_branch_ctx_for_sub_agent()"]

    B --> C["Sub-Agent A<br/>branch: 'checker.agent_a'"]
    B --> D["Sub-Agent B<br/>branch: 'checker.agent_b'"]

    C --> E["Events tagged with branch"]
    D --> F["Events tagged with branch"]

    E & F --> G["_get_events(current_branch=True)<br/>only sees own branch"]
```

Each sub-agent gets an isolated `InvocationContext` with a unique branch path. The branch format is `{parent_branch}.{parent_name}.{sub_agent_name}`.

## Event Merging — Queue-Based Backpressure

```mermaid
flowchart TD
    subgraph "Per Sub-Agent Task"
        T1["process_an_agent(events)"]
        T1 --> E1["event = next(events)"]
        E1 --> Q1["queue.put((event, resume_signal))"]
        Q1 --> W1["await resume_signal.wait()"]
        W1 --> E1
        E1 -- "Done" --> S1["queue.put(sentinel)"]
    end

    subgraph "Consumer (Main Loop)"
        C1["event, signal = queue.get()"]
        C1 --> C2{Is sentinel?}
        C2 -- Yes --> C3["sentinel_count++"]
        C2 -- No --> C4["yield event"]
        C4 --> C5["signal.set()"]
        C5 --> C1
        C3 --> C6{All agents done?}
        C6 -- No --> C1
        C6 -- Yes --> DONE["Exit"]
    end
```

Key design:
- Each sub-agent runs as an asyncio task, putting events on a shared queue
- After putting an event, the producer **waits** for the consumer to signal completion
- This ensures backpressure — sub-agents don't run ahead of event processing
- A sentinel object marks when a sub-agent finishes

## Python Version Compatibility

```mermaid
flowchart TD
    A["merge_func selection"] --> B{Python >= 3.11?}
    B -- Yes --> C["_merge_agent_run()<br/>Uses asyncio.TaskGroup"]
    B -- No --> D["_merge_agent_run_pre_3_11()<br/>Manual task management"]
```

| Version | Implementation | Error Handling |
|---------|---------------|----------------|
| Python 3.11+ | `asyncio.TaskGroup` context manager | Automatic cancellation on error |
| Python 3.10 | Manual `asyncio.create_task` + cleanup | Explicit cancellation in `finally` block, `propagate_exceptions()` check |

## Resumability

```mermaid
flowchart TD
    A["Resuming ParallelAgent"] --> B["Load agent states"]
    B --> C["For each sub_agent"]
    C --> D{end_of_agents[sub_agent.name]?}
    D -- Yes --> E["Skip (already finished)"]
    D -- No --> F["Include in agent_runs"]

    G["All sub-agents finished?"] --> H["Set end_of_agent for ParallelAgent"]
```

On resume, only sub-agents that haven't completed are re-run. The ParallelAgent marks itself as `end_of_agent` only when all sub-agents have individually finished.

## Resource Cleanup

The `finally` block in `_run_async_impl` ensures all async generator resources are properly closed:

```python
finally:
    for sub_agent_run in agent_runs:
        await sub_agent_run.aclose()
```

This prevents resource leaks even if the parent cancels execution.

## Live Mode

`_run_live_impl` raises `NotImplementedError` — parallel live streaming is not supported.
