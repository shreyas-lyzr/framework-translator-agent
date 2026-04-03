---
name: translate
description: "Translate AI agent code from one framework to another. Generates idiomatic target framework code from a Canonical Intermediate Representation (CIR). Supports 8 frameworks: LangGraph, CrewAI, OpenAI Agents SDK, AutoGen, Semantic Kernel, Haystack, Agno/Phidata, and Google ADK. Use when the user wants to convert, port, migrate, rewrite, or translate agent code between any of these frameworks. Triggers on: translate, convert, port, migrate, rewrite, translate to langgraph, convert to crewai, port to openai, migrate to autogen, rewrite in haystack, convert to google adk, change framework."
allowed-tools: "Read, Write, Edit, Bash, Grep, Glob, WebSearch, Agent"
metadata:
  version: "1.0.0"
  category: "developer-tools"
---

# Translate Agent Code

Generate target framework code from a Canonical Intermediate Representation (CIR).

## Prerequisites

If no CIR exists yet (the user jumped straight to "translate this"), run the **analyze** skill first to produce one. Do not translate without a CIR — it ensures nothing is missed.

## Step 1: Confirm Source and Target

Identify:
- **Source framework**: from the CIR or the user's input
- **Target framework**: ask the user if not specified

Supported targets: LangGraph, CrewAI, OpenAI Agents SDK, AutoGen, Semantic Kernel, Haystack, Agno, Google ADK

Confirm: `Translating from **{source}** to **{target}**. Proceed?`

## Step 2: Load Target Reference

Read the corresponding codegen reference for the target framework:
- `references/langgraph-codegen.md`
- `references/crewai-codegen.md`
- `references/openai-agents-codegen.md`
- `references/autogen-codegen.md`
- `references/semantic-kernel-codegen.md`
- `references/haystack-codegen.md`
- `references/agno-codegen.md`
- `references/google-adk-codegen.md`

Load **only** the target framework's reference file.

## Step 3: Build Concept Mapping Table

Using `knowledge/concept-mapping.md` and the CIR, create a mapping table for every element:

| # | Source ({source}) | CIR Concept | Target ({target}) | Fidelity |
|---|------------------|-------------|-------------------|----------|
| 1 | `StateGraph` node `process_input` | Agent: Input Processor | `Agent(name="input_processor", instructions=...)` | Direct |
| 2 | `add_conditional_edges` | Conditional Routing | `handoff(target_agent, condition)` | Adapted |
| 3 | `MemorySaver` checkpointer | State Persistence | N/A — manual implementation needed | Lossy |

Fidelity labels:
- **Direct** — clean 1:1 mapping, equivalent semantics
- **Adapted** — concept exists but requires restructuring
- **Lossy** — no direct equivalent, closest approximation used
- **Unsupported** — cannot be translated, will be omitted

Present this table and **pause for user confirmation** before generating code. This is the most important review step — the user should agree with the mapping decisions before code is written.

## Step 4: Generate Target Code

Write complete target framework code following these rules:

### Rule 1: All imports at top
Include every import the code needs. Do not leave any import for the user to guess.

### Rule 2: Idiomatic patterns
Follow the target framework's conventions. The output should look like it was written by an expert in that framework:
- LangGraph: functions as nodes, TypedDict state, explicit edges
- CrewAI: role/goal/backstory agents, task descriptions, crew assembly
- OpenAI Agents SDK: Agent objects with instructions, @function_tool, handoffs
- AutoGen: system_message agents, GroupChat, initiate_chat
- Semantic Kernel: Kernel + plugins, @kernel_function methods
- Haystack: @component classes with run(), Pipeline.connect()
- Agno: Agent(model, instructions, tools), Team for multi-agent
- Google ADK: Agent(name, model, instruction, tools, sub_agents), SequentialAgent/ParallelAgent/LoopAgent for workflows

### Rule 3: Preserve prompts verbatim
System prompts, instructions, role descriptions, backstories — copy them exactly. These are the agent's behavior definition. Do not paraphrase, summarize, or "improve" them.

### Rule 4: Preserve tool implementations
Translate the tool registration mechanism (e.g., `@tool` → `@function_tool`) but keep the function body identical. If the tool calls external services, preserve the exact API calls.

### Rule 5: Mark lossy translations
Add `# TRANSLATION NOTE: {explanation}` comments wherever the translation is not 1:1:

```python
# TRANSLATION NOTE: LangGraph's conditional_edges with multiple targets mapped to
# sequential handoff checks. Original had simultaneous evaluation; this is sequential.
```

### Rule 6: Include entry point
The generated code must include the equivalent of the source's kickoff/run/invoke call so it is immediately runnable.

### Rule 7: Search when uncertain
If you are not confident about a method signature, import path, or API pattern in the target framework, use **WebSearch** before writing the code. Search for:
- `"{method_name}" {framework} documentation`
- `site:docs.{framework_domain} {class_name}`
- `{framework} {concept} example python`

Never guess at APIs — always verify.

## Step 5: Framework-Specific Translation Rules

### Targeting LangGraph
- State must be a `TypedDict` (or use `MessagesState` for chat agents)
- Each agent becomes a node function: `def agent_name(state: State) -> dict:` or `-> Command` (v2.0)
- Tools: bind to model with `.bind_tools()`, use `ToolNode` for execution
- Orchestration: `graph.add_edge()` for sequential, `graph.add_conditional_edges()` for branching, or `Command(goto=)` (v2.0)
- Cycles: LangGraph supports them — use for ReAct loops
- HITL: use `interrupt()` function inside nodes (v2.0) instead of `interrupt_before`/`interrupt_after`
- Caching: `graph.add_node("name", fn, cache_policy=CachePolicy(ttl=N))` for expensive nodes
- Entry: `graph.compile(checkpointer=MemorySaver()).invoke(initial_state)`
- Multi-agent: use sub-graphs via `graph.add_node("subagent", compiled_subgraph)`

### Targeting CrewAI
- Each agent needs `role` (job title), `goal` (objective), `backstory` (context) — synthesize from instructions if source doesn't have them
- Tasks wrap work: `Task(description=..., expected_output=..., agent=agent)`
- Sequential: `Process.sequential` with tasks in order
- Hierarchical: `Process.hierarchical` adds a manager agent
- Flows: use `Flow[State]` with `@start`/`@listen`/`@router` for event-driven orchestration
- Flow persistence: `@persist` for durable state; `@human_feedback` for HITL
- Flow memory: `self.remember()`/`self.recall()` for cross-step memory
- Tools: `@tool` decorator or pass tool functions/objects to Agent
- Config: optionally generate `agents.yaml` + `tasks.yaml` for YAML-based setup
- Entry: `crew.kickoff(inputs={...})` or `flow.kickoff()`

### Targeting OpenAI Agents SDK
- Agents: `Agent(name=..., instructions=..., tools=[...], handoffs=[...])`
- Tools: `@function_tool` decorated functions with type-annotated parameters
- Multi-agent: `handoff(agent)` in agent's handoffs list
- Guardrails: `InputGuardrail(guardrail_function)` / `OutputGuardrail(...)` if source had validation
- Structured output: `output_type=PydanticModel` if source had typed output
- Entry: `result = Runner.run(agent, input="...")`
- Note: OpenAI models only — add a comment if source used non-OpenAI models

### Targeting AutoGen
- Agents: `AssistantAgent(name, system_message, llm_config)` for AI agents
- User proxy: `UserProxyAgent(name, human_input_mode, code_execution_config)` for human/code agents
- Multi-agent: `GroupChat(agents=[...], max_round=N)` + `GroupChatManager(groupchat, llm_config)`
- Tools: `agent.register_function(function_map={...})` or `@register_function`
- Two-agent: `user_proxy.initiate_chat(assistant, message=...)`
- Speaker selection: custom function for conditional routing
- LLM config: `{"model": "gpt-4", "api_key": "..."}` or `config_list`

### Targeting Semantic Kernel
- Create `Kernel()` and add AI service: `kernel.add_service(OpenAIChatCompletion(...))`
- Agents become plugin classes with `@kernel_function` methods
- Register: `kernel.add_plugin(MyPlugin(), plugin_name="...")`
- Invoke: `result = await kernel.invoke(plugin_name="...", function_name="...", arguments=KernelArguments(...))`
- Multi-agent: Use SK Agent Framework if available, otherwise manual orchestration
- Memory: `TextMemoryPlugin` for semantic memory

### Targeting Haystack
- Each processing step becomes a `@component` class with `run()` method
- Must declare `@component.output_types(output_name=Type)`
- Pipeline: `pipeline = Pipeline()`, `pipeline.add_component("name", ComponentClass())`
- Connect: `pipeline.connect("comp1.output", "comp2.input")`
- No cycles — Haystack pipelines are DAGs. If source has loops, unroll or add a comment
- Routing: `ConditionalRouter` for branching logic
- Entry: `result = pipeline.run({"component_name": {"input_name": value}})`

### Targeting Agno
- Agent: `Agent(model=OpenAIChat(id="gpt-4o"), instructions=[...], tools=[...], storage=SqliteStorage(...))`
- Tools: pass functions directly or use Toolkit classes
- Multi-agent: `Team(agents=[...], mode="coordinate")` or `Workflow` with steps
- Memory: `Memory(db=SqliteMemoryDb())` for persistent memory
- Knowledge: `knowledge_base=PDFKnowledgeBase(...)` for RAG
- Entry: `agent.run(message)` or `agent.print_response(message)`

### Targeting Google ADK
- Agent: `Agent(name="...", model="gemini-2.5-flash", instruction="...", tools=[...], sub_agents=[...])`
- Tools: plain functions with docstrings (auto-wrapped by ADK), or `BaseTool` subclass, or `AgentTool(agent=...)` to wrap an agent as a tool
- Multi-agent: `sub_agents=[...]` parameter; the LLM decides which sub_agent to invoke based on `description`
- Workflow: `SequentialAgent`, `ParallelAgent`, `LoopAgent` for deterministic control flow without LLM
- State: session key-value store; use `{var}` interpolation in `instruction` strings; `output_key=` for sequential data flow
- Memory: `InMemoryMemoryService()` for cross-session searchable recall
- Entry: `Runner(agent=agent, app_name="app", session_service=InMemorySessionService()).run_async(session_id, user_id, message)`
- Note: Default models are Gemini; add `# TRANSLATION NOTE:` if source used non-Google models (use `LiteLlm` wrapper)

## Step 6: Output

Write generated code to file(s):
- Single framework: `{agent_name}_{target}.py`
- CrewAI with YAML: also generate `agents.yaml` and `tasks.yaml`
- Semantic Kernel: may need plugin files

Present:
1. The generated code with syntax highlighting
2. A summary of translation decisions
3. A list of all `# TRANSLATION NOTE:` items (lossy/adapted mappings)
4. Required pip packages: `pip install {packages}`
5. Required environment variables
6. Suggested next step: `Run the **validate** skill to verify the translation`
