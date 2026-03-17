# LoopAgent — Iterative Sub-Agent Execution

**Source:** `src/google/adk/agents/loop_agent.py`

## Purpose

`LoopAgent` repeatedly runs its sub-agents in sequence until either a sub-agent escalates or `max_iterations` is reached. It supports resumable execution with state tracking for the current sub-agent and loop count.

## Class Overview

```mermaid
classDiagram
    class LoopAgent {
        +max_iterations: Optional[int]
        #_run_async_impl(ctx) → AsyncGenerator[Event]
        -_get_start_state(agent_state) → tuple[int, int]
    }

    class LoopAgentState {
        <<experimental>>
        +current_sub_agent: str
        +times_looped: int
    }

    class BaseAgent {
        +sub_agents: list[BaseAgent]
    }

    BaseAgent <|-- LoopAgent
    BaseAgentState <|-- LoopAgentState
```

## Execution Flow

```mermaid
flowchart TD
    A["_run_async_impl(ctx)"] --> B{sub_agents empty?}
    B -- Yes --> EXIT["Return"]
    B -- No --> C["Load LoopAgentState"]
    C --> D["_get_start_state() → times_looped, start_index"]

    D --> LOOP{"times_looped < max_iterations?<br/>(or no limit)"}
    LOOP -- No --> EXIT

    LOOP -- Yes --> E["For i in range(start_index, len(sub_agents))"]
    E --> F{Resumable AND not resuming?}
    F -- Yes --> G["Save LoopAgentState(sub_agent, times_looped)<br/>Yield state event"]
    F -- No --> H

    G --> H["sub_agent.run_async(ctx)"]
    H --> I["Yield events"]
    I --> J{event.actions.escalate?}
    J -- Yes --> SHOULD_EXIT["should_exit = True"]
    J -- No --> K{should_pause_invocation?}
    K -- Yes --> PAUSE["pause_invocation = True"]
    K -- No --> E

    SHOULD_EXIT & PAUSE --> L{exit or pause?}
    L -- Yes --> M["Break inner loop"]
    L -- No --> E

    E -- "All sub-agents done" --> N["start_index = 0, times_looped++"]
    N --> O["Reset all sub-agent states"]
    O --> LOOP

    M --> P{pause_invocation?}
    P -- Yes --> EXIT
    P -- No --> Q{is_resumable?}
    Q -- Yes --> R["Set end_of_agent, yield state event"]
    Q -- No --> EXIT
```

## Loop Termination Conditions

```mermaid
flowchart LR
    subgraph "Exit Triggers"
        T1["Sub-agent escalates<br/>(event.actions.escalate = True)"]
        T2["max_iterations reached<br/>(times_looped >= max_iterations)"]
        T3["Invocation paused<br/>(long-running tool call)"]
    end
```

| Condition | Behavior |
|-----------|----------|
| Sub-agent escalates | Breaks out of loop immediately |
| `max_iterations` reached | Exits loop naturally |
| `max_iterations` not set | Runs indefinitely until escalation |
| Invocation pause | Returns without yielding end state |

## Resume from State

```mermaid
flowchart TD
    A["_get_start_state(agent_state)"] --> B{agent_state is None?}
    B -- Yes --> C["Return (0, 0)"]
    B -- No --> D["times_looped = agent_state.times_looped"]
    D --> E{current_sub_agent set?}
    E -- No --> F["start_index = 0"]
    E -- Yes --> G["Find index by name in sub_agents"]
    G --> H{Found?}
    H -- Yes --> I["start_index = found index"]
    H -- No --> J["Log warning, start_index = 0"]
```

When resuming:
1. Skip already-completed iterations via `times_looped`
2. Resume from the specific sub-agent via `start_index`
3. If a sub-agent was removed, restart from beginning

## Sub-Agent State Reset

After each complete loop iteration, all sub-agent states are reset:

```mermaid
flowchart TD
    A["Loop iteration complete"] --> B["ctx.reset_sub_agent_states(self.name)"]
    B --> C["For each sub_agent"]
    C --> D["Clear agent_state and end_of_agent"]
    D --> E["Recurse into sub_agent's children"]
```

This ensures each loop iteration starts with fresh sub-agent state.

## Example Usage

```python
loop_agent = LoopAgent(
    name="review_loop",
    max_iterations=3,
    sub_agents=[
        Agent(name="writer", instruction="Write content..."),
        Agent(name="reviewer", instruction="Review and escalate if good..."),
    ],
)
```

The reviewer agent can call `escalate()` to break out of the loop when the content meets quality standards.

## Live Mode

`_run_live_impl` raises `NotImplementedError` — live mode is not supported for LoopAgent.
