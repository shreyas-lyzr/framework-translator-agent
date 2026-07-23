# Canonical Intermediate Representation (CIR)

The CIR is the framework-agnostic lingua franca used during translation. Every source agent is decomposed into this representation before target code is generated.

## CIR Schema

When analyzing an agent, extract these sections:

### 1. Agent Inventory
For each agent/node/component:
- **Name**: identifier
- **Role/Purpose**: what this agent does
- **Instructions**: system prompt, backstory, or instructions (quote verbatim)
- **Model**: model name and configuration if specified
- **Tools**: list of tools attached to this agent

### 2. Tool Inventory
For each tool:
- **Name**: tool identifier
- **Parameters**: name, type, required/optional, description
- **Return Type**: what the tool returns
- **Implementation**: summary of what it does
- **External**: whether it calls external services (APIs, databases, files)

### 3. State Schema
- **Fields**: name, type, default value
- **Shared vs. Local**: which state is shared across agents vs. local
- **Persistence**: whether state is persisted between runs
- **Type System**: TypedDict, Pydantic, dict, dataclass, etc.

### 4. Orchestration Graph
- **Flow Type**: sequential, parallel, conditional, cyclic
- **Edges**: from → to, with condition if conditional
- **Entry Point**: where execution starts
- **Exit Point**: where execution ends
- **Loops**: any cyclic patterns (e.g., think-act-observe)
- **Delegation**: how agents hand off to each other

### 5. Memory Configuration
- **Type**: conversation history, vector store, key-value, long-term
- **Scope**: per-agent, shared, global
- **Backend**: in-memory, database, file system

### 6. External Dependencies
- **APIs**: external service calls with endpoints
- **Databases**: connection details and query patterns
- **File I/O**: files read or written
- **Environment Variables**: required env vars

---

## Cross-Framework Concept Mapping

| Concept | LangGraph | CrewAI | OpenAI Agents SDK | AutoGen | Semantic Kernel | Haystack | Agno | Google ADK |
|---------|-----------|--------|-------------------|---------|-----------------|----------|------|------------|
| **Agent definition** | Node function | `Agent(role, goal, backstory)` | `Agent(name, instructions)` | `AssistantAgent` / `ConversableAgent` | `Kernel` + `Plugin` | `@component` class | `Agent(model, instructions)` | `Agent(name, model, instruction, tools, sub_agents)` |
| **Tool definition** | Function called in node | `@tool` decorator | `@function_tool` decorator | `register_function()` | `@kernel_function` decorator | `@component` class | Function / `Toolkit` | Plain function (auto-wrapped) / `BaseTool` / `AgentTool` |
| **Orchestration** | `StateGraph` with `add_edge()` / `add_conditional_edges()` / `Command(goto=)` (v2.0) | `Process.sequential` / `Process.hierarchical` / Flows (`@start`/`@listen`/`@router`) | `handoff()` between agents | `GroupChat` + `GroupChatManager` | `Planner` / manual `kernel.invoke()` chain | `Pipeline.connect()` | `Team` / `Workflow` | `sub_agents` (LLM delegation) / `SequentialAgent` / `ParallelAgent` / `LoopAgent` |
| **State** | `TypedDict` passed through nodes | Shared crew context / `Flow[PydanticState]` | `RunContext` | Chat message history | `KernelArguments` | Pipeline input/output dict | Session state | Session key-value with `{var}` interpolation |
| **Conditional routing** | `add_conditional_edges(node, func, mapping)` / `Command(goto=)` (v2.0) | `Process.hierarchical` (manager decides) / `@router` in Flows | Handoff with conditions | `speaker_selection_method` function | Planner step selection | `Router` component | Route conditions in workflow | LLM-driven `sub_agents` selection based on `description` |
| **Parallel execution** | Multiple edges from same node | Tasks with `async_execution=True` / multiple `@start` in Flows | Multiple agents via Runner | Nested chat patterns | Parallel `kernel.invoke()` calls | Pipeline parallel branches | `Team` with parallel mode | `ParallelAgent(sub_agents=[...])` |
| **Memory** | State persistence via checkpointer | `memory=True` (short/long/entity) / `self.remember()`/`self.recall()` in Flows | `RunContext` state | `TeachableAgent` / chat history | Memory plugin (semantic/volatile) | `DocumentStore` | `Memory` + `Storage` backend | `MemoryService` (cross-session searchable) |
| **Human-in-the-loop** | `interrupt()` function (v2.0) / `interrupt_before`/`interrupt_after` (pre-v2.0) | `human_input=True` on task / `@human_feedback` in Flows | Guardrails + custom logic | `HumanProxyAgent` / `human_input_mode` | Manual approval step | Custom component | `human_input=True` | Custom tool with user input confirmation |
| **Entry point** | `graph.compile().invoke()` from `START` | `crew.kickoff()` / `flow.kickoff()` | `Runner.run(agent, input)` | `agent.initiate_chat()` | `kernel.invoke(function)` | `pipeline.run(data)` | `agent.run(message)` | `Runner(agent).run_async(session_id, message)` |
| **Sub-agents** | Sub-graphs via `add_node(compiled_graph)` | Delegation between crew agents | `handoff()` to another Agent | Nested `initiate_chat()` | Multiple plugins on kernel | Nested pipelines | `Team` members | `sub_agents=[...]` parameter |
| **Streaming** | `.astream()` / `.stream()` | Callback handlers / `stream=True` in Flows | `Runner.run_streamed()` | Callback-based | `kernel.invoke_stream()` | `Pipeline.run()` with streaming components | `agent.run(stream=True)` | Async event stream from `Runner.run_async()` |
| **Error handling** | Try/except in nodes, state rollback | Task-level retry config | Guardrails (input/output) | Error in chat history | Kernel filters/hooks | Component error propagation | Built-in retry | Custom `BaseAgent` error handling / callbacks |

## Translation Fidelity Labels

Use these labels in the concept mapping table during translation:

- **Direct**: Clean 1:1 mapping — the concept exists in both frameworks with equivalent semantics
- **Adapted**: The concept exists but requires structural restructuring (e.g., a graph edge becomes a handoff)
- **Lossy**: No direct equivalent — the closest approximation is used with a clear explanation of what is lost
- **Unsupported**: Cannot be translated — the concept will be omitted with an explanation and a suggestion for manual implementation

---

## Lyzr SuperFlow (Source Workflow) → CIR

SuperFlow is a **source-only** framework: the input is a `flow_data` **JSON graph**, not code. It decomposes
into the CIR cleanly and maps **1:1 to LangGraph** (both are node+edge graphs with typed state). The full
node catalog, expression translation, trigger/HITL/reliability handling and a worked example live in
`skills/analyze/references/superflow-patterns.md`. Summary of how SuperFlow constructs fill the CIR:

| CIR section | SuperFlow source |
|---|---|
| **Agent Inventory** | `lyzr.llm` (systemPrompt/model verbatim), `lyzr.agent` (managed agent id), `lyzr.taskDecomposition` (swarm) |
| **Tool Inventory** | `httpRequest`, `code` (JS body), `lyzr.tool`, sub-agent-flagged nodes; document nodes as external tools |
| **State Schema** | one channel **per node name** (holds that node's output items) + `input`/`trigger`; enables `{{ $('Node').json }}` refs |
| **Orchestration Graph** | `connections` (keyed by node name) → edges; `if`/`switch` → conditional edges; `splitInBatches` → loop; `merge` → fan-in |
| **HITL** | `waitForApproval` (message + typed form; approved=slot0 / rejected=slot1) |
| **External Dependencies** | HTTP endpoints + auth, Lyzr document/parse/extraction services, agent/tool ids, `envs`/credentials |

### SuperFlow → LangGraph quick fidelity table

| SuperFlow node | LangGraph target | Fidelity |
|---|---|---|
| `trigger` | `START` + initial `state["input"]` (manual/webhook/schedule → see reference) | Direct |
| `lyzr.llm` | node calling a `ChatModel` (ReAct via `create_react_agent` if looped) | Direct / Adapted |
| `lyzr.agent` | reconstructed LLM/ReAct node **or** call the Lyzr agent API | Adapted (KB/memory = Lossy) |
| `if` / `switch` | `add_conditional_edges(...)` | Direct |
| `merge` / `splitInBatches` (loop) | fan-in reducer / cyclic subgraph or `Send` | Adapted |
| `set` / `sort` / `limit` / `filter` / `renameKeys` | node with a Python transform | Direct |
| `httpRequest` | `httpx`/`requests` node (SigV4/OAuth2/mTLS = Adapted) | Direct |
| `code` (JS) | node re-expressing the logic in Python | Adapted / Lossy |
| `executeWorkflow` | compiled sub-graph as a node | Direct |
| `waitForApproval` | `interrupt(...)` (LG v2.0) / `interrupt_before` | Direct |
| `lyzr.parse` / `lyzr.extraction` / `lyzr.label` / `lyzr.guardrails` | doc/LLM/guardrail node or Lyzr API call | Adapted / Lossy |
| durable exactly-once, timed `wait` | checkpointer (resume) / `time.sleep` | Lossy — flag with `# TRANSLATION NOTE:` |

Non-LangGraph targets **adapt** the graph: CrewAI **Flows** (`@start`/`@listen`/`@router`), Haystack
`Pipeline.connect()`, OpenAI Agents SDK handoffs, Google ADK `SequentialAgent`/`ParallelAgent`/`LoopAgent`.
LangGraph is the recommended default target for SuperFlow because the graph structure is preserved exactly.
