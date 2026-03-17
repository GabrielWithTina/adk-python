# Questions

- How Claude Code discover and spawn sub-agents?
- How to integrate with OTEL?
- How to resume agent and sub agents?

# Sub-agent design comparation

| Framework           | Default Sub-Agent Context   | How They Coordinate                          | Isolation Available?                         |
|--------------------|-----------------------------|-----------------------------------------------|----------------------------------------------|
| ADK                | Shared session + events     | Conversation history + session state          | Only ParallelAgent branches                  |
| OpenAI Agents SDK  | Shared chat history         | Handoff (swap system prompt, keep history)    | Handoff history compression (opt-in)         |
| LangGraph          | Shared state graph          | Shared messages list + state reducers         | Isolated subgraphs (explicit opt-in)         |
| AutoGen            | Isolated (Actor Model)      | Async message passing                         | Shared state is the gap, not isolation       |
| CrewAI             | Task-mediated               | Task outputs + shared Memory layer            | Agents don't see raw conversation            |
| Claude Code        | Isolated                    | Task prompt in, result out                    | Isolation is the default                     |

```shell
  Fully Shared                                              Fully Isolated
       |                                                          |
       ADK          OpenAI         LangGraph      AutoGen     Claude Code
       |            Agents SDK     (default)      |               |
       |                |              |          |               |
    Same session    Same chat      Shared state   Isolated agents  Fresh context
    Same events     history        graph with     with explicit    per task.
    Same state.     swap prompt.   reducers.      HandoffMessage,  Only final
    Sub-agent                      Opt-in         TeamState,       string
    sees all.                      isolation.     GroupChat.       returns.
                                                  No implicit
                                                  sharing.
```

|                    | AutoGen (isolated + state)                              | Claude Code (isolated + no state)                               |
|--------------------|----------------------------------------------------------|------------------------------------------------------------------|
| Coordination       | Agents can read shared state, group chat threads          | Parent must encode everything into the task prompt              |
| Continuity         | TeamState persists across requests                        | Each sub-agent starts from zero                                 |
| Traceability       | Agent states serialized, auditable                        | Intermediate work is lost (by design)                           |
| Context efficiency | State object is compact                                   | No overhead — but parent must re-summarize context              |
| Use case           | Multi-agent workflows that evolve over time               | One-shot task delegation where isolation prevents context rot   |
