# LangGraphAgent — LangGraph Integration

**Source:** `src/google/adk/agents/langgraph_agent.py`

## Purpose

`LangGraphAgent` wraps a LangGraph `CompiledGraph` to run within the ADK agent framework. It bridges LangGraph's message-based execution with ADK's event-based system, supporting both single-turn and multi-turn conversations via LangGraph's checkpointer.

## Class Overview

```mermaid
classDiagram
    class LangGraphAgent {
        +graph: CompiledGraph
        +instruction: str

        #_run_async_impl(ctx) → AsyncGenerator[Event]
        -_get_messages(events) → list[Message]
        -_get_conversation_with_agent(events) → list[Message]
    }

    class BaseAgent {
        +name: str
    }

    BaseAgent <|-- LangGraphAgent
```

## Execution Flow

```mermaid
flowchart TD
    A["_run_async_impl(ctx)"] --> B["Get LangGraph config with thread_id = session.id"]
    B --> C["Get current graph state"]
    C --> D{Graph has existing messages?}

    D -- No --> E{instruction set?}
    E -- Yes --> F["messages = [SystemMessage(instruction)]"]
    E -- No --> G["messages = []"]
    D -- Yes --> G

    F --> H["Append _get_messages(session.events)"]
    G --> H

    H --> I["graph.invoke({'messages': messages}, config)"]
    I --> J["Extract result from final_state['messages'][-1].content"]
    J --> K["Yield Event with result"]
```

## Message Extraction Strategy

The agent uses two different strategies depending on whether the LangGraph graph has its own checkpointer:

```mermaid
flowchart TD
    A["_get_messages(events)"] --> B{graph.checkpointer exists?}
    B -- Yes --> C["_get_last_human_messages(events)"]
    B -- No --> D["_get_conversation_with_agent(events)"]
```

### With Checkpointer (LangGraph manages memory)

```mermaid
flowchart TD
    A["_get_last_human_messages(events)"] --> B["Iterate events in reverse"]
    B --> C{event.author == 'user'?}
    C -- Yes --> D["Collect HumanMessage"]
    C -- No --> E{Already collected messages?}
    E -- Yes --> F["Stop (hit non-user message)"]
    E -- No --> B
    D --> B
    F --> G["Return reversed messages"]
```

Only the **last consecutive user messages** are sent — the graph's checkpointer has the full history.

### Without Checkpointer (ADK manages memory)

```mermaid
flowchart TD
    A["_get_conversation_with_agent(events)"] --> B["Iterate all events"]
    B --> C{event has content?}
    C -- No --> B
    C -- Yes --> D{event.author?}
    D -- "'user'" --> E["Add HumanMessage"]
    D -- "self.name" --> F["Add AIMessage"]
    D -- "other" --> B
    E & F --> B
    B -- "Done" --> G["Return all messages"]
```

The **full conversation** between user and this specific agent is sent — other agents' messages are filtered out.

## ADK ↔ LangGraph Type Mapping

```mermaid
flowchart LR
    subgraph "ADK Events"
        E_USER["Event(author='user')"]
        E_AGENT["Event(author=agent_name)"]
    end

    subgraph "LangGraph Messages"
        LG_HUMAN["HumanMessage"]
        LG_AI["AIMessage"]
        LG_SYS["SystemMessage"]
    end

    E_USER --> LG_HUMAN
    E_AGENT --> LG_AI
    INSTRUCTION["instruction field"] --> LG_SYS
```

## Multi-Turn Support

```mermaid
sequenceDiagram
    participant User
    participant ADK as ADK Runner
    participant LGA as LangGraphAgent
    participant LG as LangGraph

    User->>ADK: "Hello"
    ADK->>LGA: run_async(ctx)
    LGA->>LG: invoke({messages: [SystemMessage, HumanMessage]}, config={thread_id})
    LG-->>LGA: final_state
    LGA-->>ADK: Event(content=result)

    User->>ADK: "Tell me more"
    ADK->>LGA: run_async(ctx)
    Note over LGA: graph.get_state(config) has history
    LGA->>LG: invoke({messages: [HumanMessage]}, config={thread_id})
    Note over LG: Checkpointer provides prior context
    LG-->>LGA: final_state
    LGA-->>ADK: Event(content=result)
```

The `thread_id` is set to `session.id`, giving each ADK session its own LangGraph conversation thread.

## Limitations

- **Concept implementation** — the class docstring notes this is currently a concept
- Only extracts `text` from the first part of event content
- No support for tool calls, function responses, or multi-part content
- `_run_live_impl` is not implemented (inherits `NotImplementedError` from `BaseAgent`)
- Requires `langchain_core` and `langgraph` as dependencies
