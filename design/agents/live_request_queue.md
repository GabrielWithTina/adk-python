# LiveRequestQueue — Bidirectional Streaming Queue

**Source:** `src/google/adk/agents/live_request_queue.py`

## Purpose

`LiveRequestQueue` provides an async queue for bidirectional (live) streaming between a client and an agent. It wraps `asyncio.Queue` with typed methods for sending content, audio/video blobs, and activity signals. This is the primary interface for pushing real-time user input into a live agent session.

## Class Overview

```mermaid
classDiagram
    class LiveRequest {
        +content: Optional[Content]
        +blob: Optional[Blob]
        +activity_start: Optional[ActivityStart]
        +activity_end: Optional[ActivityEnd]
        +close: bool = False
    }

    class LiveRequestQueue {
        -_queue: asyncio.Queue
        +close()
        +send_content(content)
        +send_realtime(blob)
        +send_activity_start()
        +send_activity_end()
        +send(req: LiveRequest)
        +get() → LiveRequest
    }

    LiveRequestQueue o-- LiveRequest
```

## Request Types and Priority

When multiple fields are set in a `LiveRequest`, they are processed by priority:

```mermaid
flowchart LR
    subgraph "Priority (highest → lowest)"
        P1["activity_start"] --> P2["activity_end"] --> P3["blob"] --> P4["content"]
    end
```

| Method | Creates | Use Case |
|--------|---------|----------|
| `send_content(content)` | `LiveRequest(content=...)` | Turn-by-turn text/content messages |
| `send_realtime(blob)` | `LiveRequest(blob=...)` | Audio/video streaming data |
| `send_activity_start()` | `LiveRequest(activity_start=ActivityStart())` | Signal user started speaking/typing |
| `send_activity_end()` | `LiveRequest(activity_end=ActivityEnd())` | Signal user stopped speaking/typing |
| `close()` | `LiveRequest(close=True)` | Terminate the queue |

## Data Flow

```mermaid
sequenceDiagram
    participant Client as Client Application
    participant LRQ as LiveRequestQueue
    participant Flow as Live LLM Flow
    participant LLM as Live LLM Session

    Client->>LRQ: send_activity_start()
    Client->>LRQ: send_realtime(audio_blob)
    Client->>LRQ: send_realtime(audio_blob)
    Client->>LRQ: send_activity_end()

    loop Agent processing
        Flow->>LRQ: await get()
        LRQ-->>Flow: LiveRequest
        Flow->>LLM: Forward to live session
        LLM-->>Flow: Model response events
    end

    Client->>LRQ: close()
    Flow->>LRQ: await get()
    LRQ-->>Flow: LiveRequest(close=True)
    Note over Flow: Exit processing loop
```

## Queue Semantics

- **Non-blocking sends**: All `send_*` methods use `put_nowait()` — they never block the caller
- **Blocking receive**: `get()` is async and awaits the next request
- **Close signal**: `close()` enqueues a sentinel `LiveRequest(close=True)` — consumers check this to exit
- **No Python 3.13 shutdown**: The `close` field is a workaround since `queue.shutdown()` requires Python 3.13+

## Integration with InvocationContext

```mermaid
flowchart TD
    A["InvocationContext"] --> B["live_request_queue: Optional[LiveRequestQueue]"]
    B --> C["Set by Runner for live mode"]
    C --> D["Consumed by Live LLM Flow"]
    D --> E["Forwarded to Gemini Live API session"]
```

The queue is attached to `InvocationContext.live_request_queue` and is the link between external input and the live flow processing loop.
