# SequentialAgent — Ordered Sub-Agent Pipeline

**Source:** `src/google/adk/agents/sequential_agent.py`

## Purpose

`SequentialAgent` runs its sub-agents one after another in order. It supports resumable execution, pausing at long-running tools, and a special live mode where each sub-agent signals completion via a `task_completed()` tool.

## Class Overview

```mermaid
classDiagram
    class SequentialAgent {
        #_run_async_impl(ctx) → AsyncGenerator[Event]
        #_run_live_impl(ctx) → AsyncGenerator[Event]
        -_get_start_index(agent_state) → int
    }

    class SequentialAgentState {
        <<experimental>>
        +current_sub_agent: str
    }

    class BaseAgent {
        +sub_agents: list[BaseAgent]
    }

    BaseAgent <|-- SequentialAgent
    BaseAgentState <|-- SequentialAgentState
```

## Async Execution Flow

```mermaid
flowchart TD
    A["_run_async_impl(ctx)"] --> B{sub_agents empty?}
    B -- Yes --> EXIT["Return"]
    B -- No --> C["Load SequentialAgentState"]
    C --> D["_get_start_index(state) → start_index"]

    D --> E["For i in range(start_index, len(sub_agents))"]
    E --> F{Resuming this sub-agent?}
    F -- Yes --> SKIP["Skip state event (already logged)"]
    F -- No --> G{is_resumable?}
    G -- Yes --> H["Save SequentialAgentState<br/>Yield state event"]
    G -- No --> RUN
    H --> RUN

    SKIP --> RUN["sub_agent.run_async(ctx)"]
    RUN --> I["Yield events"]
    I --> J{should_pause_invocation?}
    J -- Yes --> PAUSE["Return (paused)"]
    J -- No --> K["resuming_sub_agent = False"]
    K --> E

    E -- "All done" --> L{is_resumable?}
    L -- Yes --> M["Set end_of_agent, yield state event"]
    L -- No --> EXIT
```

## Resume Logic

```mermaid
flowchart TD
    A["_get_start_index(agent_state)"] --> B{agent_state is None?}
    B -- Yes --> C["Return 0 (start from beginning)"]
    B -- No --> D{current_sub_agent empty?}
    D -- Yes --> E["Return len(sub_agents)<br/>(process was finished)"]
    D -- No --> F["Find index by name"]
    F --> G{Found?}
    G -- Yes --> H["Return index"]
    G -- No --> I["Log warning, return 0"]
```

## Live Mode — `task_completed()` Pattern

In live (bidirectional streaming) mode, there's no natural end to a sub-agent's turn since audio/video is continuous. The solution: inject a `task_completed()` function tool.

```mermaid
flowchart TD
    A["_run_live_impl(ctx)"] --> B["For each LlmAgent sub_agent"]
    B --> C["Inject task_completed function tool"]
    C --> D["Append instruction: call task_completed when done"]

    D --> E["For each sub_agent"]
    E --> F["sub_agent.run_live(ctx)"]
    F --> G["Yield events until task_completed called"]
    G --> E
```

```mermaid
sequenceDiagram
    participant User as User (Audio/Video)
    participant SA1 as Sub-Agent 1
    participant SA2 as Sub-Agent 2

    User->>SA1: Continuous audio stream
    SA1->>SA1: Process and respond
    SA1->>SA1: Call task_completed()
    Note over SA1: Signals completion

    User->>SA2: Stream continues
    SA2->>SA2: Process and respond
    SA2->>SA2: Call task_completed()
```

Key details:
- Only `LlmAgent` sub-agents get the tool (checked via `isinstance`)
- Tool is deduplicated by function name to avoid double-injection
- Instruction appended tells agent to call `task_completed` when done

## Comparison: Sequential vs Loop vs Parallel

```mermaid
flowchart LR
    subgraph Sequential["SequentialAgent"]
        S1["Agent A"] --> S2["Agent B"] --> S3["Agent C"]
    end

    subgraph Loop["LoopAgent"]
        L1["Agent A"] --> L2["Agent B"]
        L2 --> L1
    end

    subgraph Parallel["ParallelAgent"]
        P1["Agent A"]
        P2["Agent B"]
        P3["Agent C"]
    end
```

| Feature | SequentialAgent | LoopAgent | ParallelAgent |
|---------|----------------|-----------|---------------|
| Execution order | Fixed, one-by-one | Repeating cycle | Concurrent |
| Termination | All sub-agents complete | Escalation or max iterations | All sub-agents complete |
| Shared context | Yes (same branch) | Yes (same branch) | Isolated branches |
| Resumability | Yes | Yes | Yes |
| Live mode | Yes (with task_completed) | No | No |
