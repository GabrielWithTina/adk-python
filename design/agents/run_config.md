# RunConfig — Runtime Behavior Configuration

**Source:** `src/google/adk/agents/run_config.py`

## Purpose

`RunConfig` provides runtime configuration for agent execution — streaming modes, LLM call limits, speech settings, thread pool configuration for tools, and session management. It is passed through `InvocationContext` and controls how the runner orchestrates agent execution.

## Class Overview

```mermaid
classDiagram
    class RunConfig {
        +speech_config: Optional[SpeechConfig]
        +response_modalities: Optional[list[str]]
        +streaming_mode: StreamingMode
        +save_live_blob: bool
        +max_llm_calls: int = 500
        +tool_thread_pool_config: Optional[ToolThreadPoolConfig]
        +output_audio_transcription: Optional[AudioTranscriptionConfig]
        +input_audio_transcription: Optional[AudioTranscriptionConfig]
        +realtime_input_config: Optional[RealtimeInputConfig]
        +enable_affective_dialog: Optional[bool]
        +proactivity: Optional[ProactivityConfig]
        +session_resumption: Optional[SessionResumptionConfig]
        +context_window_compression: Optional[ContextWindowCompressionConfig]
        +support_cfc: bool = False
        +custom_metadata: Optional[dict]
        +get_session_config: Optional[GetSessionConfig]
    }

    class StreamingMode {
        <<enum>>
        NONE = None
        SSE = "sse"
        BIDI = "bidi"
    }

    class ToolThreadPoolConfig {
        +max_workers: int = 4
    }

    RunConfig *-- StreamingMode
    RunConfig *-- ToolThreadPoolConfig
```

## Streaming Modes

```mermaid
flowchart LR
    subgraph "StreamingMode.NONE"
        N1["Runner"] --> N2["Single complete response per turn"]
        N2 --> N3["No partial events"]
    end

    subgraph "StreamingMode.SSE"
        S1["Runner"] --> S2["Partial events (streaming chunks)"]
        S2 --> S3["Aggregated final event"]
        S2 --> S4["Typewriter effect for UI"]
    end

    subgraph "StreamingMode.BIDI"
        B1["runner.run_live()"]
        B1 --> B2["Separate code path entirely"]
    end
```

### SSE Event Types

| Event Type | `event.partial` | Content | Display? |
|-----------|----------------|---------|----------|
| Partial text | `True` | Streaming text chunk | Yes (typewriter) |
| Partial function call | `True` | Function call args building | No (internal) |
| Aggregated final | `False` | Complete response | Depends on pattern |

### SSE Display Patterns

```mermaid
flowchart TD
    A["SSE Event Received"] --> B{event.partial?}
    B -- Yes --> C{Has text AND no function_call?}
    C -- Yes --> D["Pattern A: Display partial text"]
    C -- No --> E["Skip (function call streaming)"]

    B -- No --> F["Pattern B: Display final only"]

    D --> NOTE["Warning: Deduplicate with final event"]
```

## Tool Thread Pool

```mermaid
flowchart TD
    A["Tool execution in live mode"] --> B{tool_thread_pool_config set?}
    B -- No --> C["Run in main event loop"]
    B -- Yes --> D["Run in ThreadPoolExecutor"]
    D --> E["max_workers threads"]

    subgraph "GIL Considerations"
        F["Helps: I/O, C extensions, sleep"]
        G["Does NOT help: Pure Python CPU"]
    end
```

## LLM Call Limit

```mermaid
flowchart TD
    A["max_llm_calls = 500 (default)"] --> B{Value check}
    B --> C{"> 0 and < sys.maxsize"}
    C -- Yes --> D["Limit enforced"]
    B --> E{"<= 0"}
    E --> F["Warning: unbounded calls"]
    B --> G{"== sys.maxsize"}
    G --> H["Raise ValueError"]
```

The limit is enforced by `InvocationContext._InvocationCostManager`, which raises `LlmCallsLimitExceededError` when exceeded.

## Configuration Fields Summary

| Field | Default | Purpose |
|-------|---------|---------|
| `streaming_mode` | `NONE` | Event streaming behavior |
| `max_llm_calls` | `500` | Safety limit on LLM calls per invocation |
| `speech_config` | `None` | Voice configuration for live agents |
| `response_modalities` | `None` | Output modalities (default: AUDIO for live) |
| `save_live_blob` | `False` | Save audio/video to session + artifacts |
| `tool_thread_pool_config` | `None` | Thread pool for tool execution |
| `support_cfc` | `False` | Compositional Function Calling (experimental) |
| `enable_affective_dialog` | `None` | Emotion-aware responses |
| `proactivity` | `None` | Model proactivity configuration |
| `session_resumption` | `None` | Transparent session resumption |
| `context_window_compression` | `None` | LLM input compression |
| `get_session_config` | `None` | Control event fetching on session load |
| `custom_metadata` | `None` | User-defined metadata for the invocation |

## Deprecated Fields

| Field | Replacement |
|-------|-------------|
| `save_live_audio` | `save_live_blob` |
| `save_input_blobs_as_artifacts` | `SaveFilesAsArtifactsPlugin` |

The `check_for_deprecated_save_live_audio` model validator auto-migrates `save_live_audio` to `save_live_blob`.
