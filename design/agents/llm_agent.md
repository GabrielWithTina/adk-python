# LlmAgent — LLM-Powered Agent

**Source:** `src/google/adk/agents/llm_agent.py`

## Purpose

`LlmAgent` (aliased as `Agent`) is the primary agent type in ADK. It wraps an LLM model with tools, instructions, callbacks, and transfer capabilities. It delegates execution to an LLM flow that handles the reason-act loop — calling the model, executing tools, and managing agent transfers.

## Class Overview

```mermaid
classDiagram
    class LlmAgent {
        +DEFAULT_MODEL: ClassVar[str] = "gemini-2.5-flash"
        +model: str | BaseLlm
        +instruction: str | InstructionProvider
        +global_instruction: str | InstructionProvider
        +static_instruction: Optional[ContentUnion]
        +tools: list[ToolUnion]
        +generate_content_config: Optional[GenerateContentConfig]
        +disallow_transfer_to_parent: bool
        +disallow_transfer_to_peers: bool
        +include_contents: "default" | "none"
        +input_schema: Optional[type[BaseModel]]
        +output_schema: Optional[SchemaType]
        +output_key: Optional[str]
        +planner: Optional[BasePlanner]
        +code_executor: Optional[BaseCodeExecutor]

        +before_model_callback: Optional[BeforeModelCallback]
        +after_model_callback: Optional[AfterModelCallback]
        +on_model_error_callback: Optional[OnModelErrorCallback]
        +before_tool_callback: Optional[BeforeToolCallback]
        +after_tool_callback: Optional[AfterToolCallback]
        +on_tool_error_callback: Optional[OnToolErrorCallback]

        +set_default_model(model) → None
        +canonical_model → BaseLlm
        +canonical_instruction(ctx) → tuple[str, bool]
        +canonical_tools(ctx) → list[BaseTool]
    }

    class BaseAgent {
        +name: str
        +sub_agents: list[BaseAgent]
    }

    BaseAgent <|-- LlmAgent
```

## Execution Flow

```mermaid
flowchart TD
    A["_run_async_impl(ctx)"] --> B["Load agent state"]
    B --> C{Resuming? Sub-agent to resume?}

    C -- Yes --> D["Run sub-agent to resume"]
    D --> E["Set end_of_agent, yield state event"]

    C -- No --> F["_llm_flow.run_async(ctx)"]
    F --> G["Yield events from flow"]
    G --> H["__maybe_save_output_to_state()"]
    H --> I{should_pause_invocation?}
    I -- Yes --> J["Set should_pause flag"]
    I -- No --> G

    J --> K{Pause flagged?}
    K -- Yes --> EXIT["Return (paused)"]
    K -- No --> L{is_resumable?}
    L -- Yes --> M["Check last events for pause"]
    M --> N["Set end_of_agent, yield state event"]
    L -- No --> EXIT
```

## LLM Flow Selection

The `_llm_flow` property determines which execution flow to use based on agent configuration:

```mermaid
flowchart TD
    A["_llm_flow"] --> B{disallow_transfer_to_parent<br/>AND disallow_transfer_to_peers<br/>AND no sub_agents?}
    B -- Yes --> C["SingleFlow()"]
    B -- No --> D["AutoFlow()"]
```

| Flow | When Used | Behavior |
|------|-----------|----------|
| `SingleFlow` | No transfers possible, no sub-agents | Simple LLM loop without transfer tools |
| `AutoFlow` | Default | Adds `transfer_to_agent` tool, manages agent routing |

## Model Resolution

```mermaid
flowchart TD
    A["canonical_model"] --> B{model is BaseLlm?}
    B -- Yes --> C["Return model directly"]
    B -- No --> D{model is non-empty string?}
    D -- Yes --> E["LLMRegistry.new_llm(model)"]
    D -- No --> F["Walk parent chain"]
    F --> G{Parent is LlmAgent with model?}
    G -- Yes --> H["Return parent.canonical_model"]
    G -- No --> I["_resolve_default_model()"]
    I --> J["Return default (gemini-2.5-flash)"]
```

The model resolution chain: **agent's own model → ancestor agent's model → class default model**.

`set_default_model()` overrides the class-level default for all agents.

## Instruction System

```mermaid
flowchart LR
    subgraph "Instruction Types"
        direction TB
        SI["static_instruction<br/>Literal content, no placeholders<br/>→ system_instruction (cacheable)"]
        GI["global_instruction (DEPRECATED)<br/>Shared across all agents<br/>Root agent only"]
        DI["instruction<br/>Dynamic, supports {placeholders}<br/>→ system_instruction or user content"]
    end

    subgraph "Resolution"
        direction TB
        R1["String → return as-is"]
        R2["InstructionProvider(ctx) → call function"]
        R3["Async provider → await result"]
    end
```

### Instruction Placement Logic

| `static_instruction` | `instruction` goes to |
|---|---|
| `None` | `system_instruction` |
| Set | User content (after static content) |

This separation enables context caching — static content at the front of the prompt is cacheable while dynamic instructions change per request.

## Callback Pipeline

```mermaid
flowchart TD
    subgraph "Model Callbacks"
        BM["before_model_callback(ctx, llm_request)"]
        BM --> BM_R{Returned LlmResponse?}
        BM_R -- Yes --> SKIP_MODEL["Skip model call, use returned response"]
        BM_R -- No --> MODEL["Call LLM"]
        MODEL --> AM["after_model_callback(ctx, llm_response)"]
        AM --> AM_R{Returned LlmResponse?}
        AM_R -- Yes --> USE_OVERRIDE["Use overridden response"]
        AM_R -- No --> USE_ACTUAL["Use actual response"]
        MODEL -.->|Error| OME["on_model_error_callback(ctx, request, error)"]
    end

    subgraph "Tool Callbacks"
        BT["before_tool_callback(tool, args, tool_ctx)"]
        BT --> BT_R{Returned dict?}
        BT_R -- Yes --> SKIP_TOOL["Skip tool execution, use returned result"]
        BT_R -- No --> TOOL["Execute tool"]
        TOOL --> AT["after_tool_callback(tool, args, tool_ctx, result)"]
        AT --> AT_R{Returned dict?}
        AT_R -- Yes --> USE_TOOL_OVERRIDE["Use overridden result"]
        AT_R -- No --> USE_TOOL_ACTUAL["Use actual result"]
        TOOL -.->|Error| OTE["on_tool_error_callback(tool, args, tool_ctx, error)"]
    end
```

All callbacks support single function or list of functions. When a list is provided, callbacks execute in order until one returns non-None.

## Tool Resolution

```mermaid
flowchart TD
    A["canonical_tools(ctx)"] --> B["For each tool in self.tools"]
    B --> C{Tool type?}

    C -- "BaseTool" --> D["Use directly"]
    C -- "callable" --> E["Wrap in FunctionTool"]
    C -- "BaseToolset" --> F["get_tools_with_prefix(ctx)"]

    D & E & F --> G["Collect all resolved tools"]

    subgraph "Special Handling (multiple tools)"
        H["GoogleSearchTool → GoogleSearchAgentTool"]
        I["VertexAiSearchTool → DiscoveryEngineSearchTool"]
    end
```

Tool resolution runs concurrently via `asyncio.gather` for all tool unions.

## Agent Transfer

When sub-agents exist and transfers are allowed, the LLM flow adds a `transfer_to_agent` tool:

```mermaid
flowchart TD
    A["LLM calls transfer_to_agent(agent_name)"] --> B["Flow yields transfer event"]
    B --> C["Parent LlmAgent detects transfer"]
    C --> D["_get_subagent_to_resume()"]
    D --> E["Resume target sub-agent"]

    subgraph "Transfer Restrictions"
        F["disallow_transfer_to_parent = True<br/>→ Cannot go back to parent"]
        G["disallow_transfer_to_peers = True<br/>→ Cannot transfer to siblings"]
    end
```

## Output Schema & State

```mermaid
flowchart TD
    A["Event from LLM"] --> B{output_key set?}
    B -- No --> EXIT["Pass through"]
    B -- Yes --> C{is_final_response?}
    C -- No --> EXIT
    C -- Yes --> D["Extract text from parts"]
    D --> E{output_schema set?}
    E -- Yes --> F["validate_schema(output_schema, text)"]
    E -- No --> G["Use raw text"]
    F & G --> H["event.actions.state_delta[output_key] = result"]
```

When `output_schema` is set, the agent can **only reply** — no tools, transfers, or sub-agents.

## Validation Rules

| Rule | Enforcement |
|------|-------------|
| `generate_content_config.tools` must be empty | Raises `ValueError` — use `LlmAgent.tools` instead |
| `generate_content_config.system_instruction` must be empty | Raises `ValueError` — use `LlmAgent.instruction` |
| `generate_content_config.response_schema` must be empty | Raises `ValueError` — use `LlmAgent.output_schema` |
| Dual thinking config warning | `model_post_init` warns if both `generate_content_config.thinking_config` and `planner.thinking_config` are set |
