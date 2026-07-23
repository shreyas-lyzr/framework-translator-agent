# framework-translator-agent

Translates AI agent code between 9 major frameworks with exact 1:1 mapping: LangGraph, CrewAI, OpenAI Agents SDK, AutoGen, Semantic Kernel, Haystack, Agno/Phidata, Google ADK, and Lyzr ADK. It also translates **Lyzr SuperFlow** workflows (a JSON `flow_data` graph) into any of these frameworks 1:1 — LangGraph is the natural target since SuperFlow is itself a graph of nodes and typed edges. Analyzes source structure, produces a canonical intermediate representation, generates idiomatic target code, and validates translation fidelity.

## Run

```bash
npx @open-gitagent/gitagent run -r https://github.com/shreyas-lyzr/framework-translator-agent
```

## What It Can Do

- **Analyze** any agent codebase — auto-detects the source framework, extracts agents, tools, state, orchestration, and memory into a canonical representation with ASCII architecture diagrams
- **Translate** between all 9 frameworks — generates idiomatic target code with verbatim prompt preservation, complete imports, and entry points
- **Validate** translated code — checks import correctness, API accuracy, completeness, and idiomatic patterns without executing
- **Search when uncertain** — uses web search to verify unknown APIs, method signatures, and import paths rather than guessing
- **Flag lossy translations** — explicitly marks every non-1:1 mapping with `# TRANSLATION NOTE:` comments explaining what was lost and why

## Supported Frameworks

| Framework | Paradigm | As Source | As Target |
|-----------|----------|-----------|-----------|
| LangGraph | Graph-based (nodes + edges) | Yes | Yes |
| CrewAI | Role-based (agents in crews) | Yes | Yes |
| OpenAI Agents SDK | Function-based (agents + handoffs) | Yes | Yes |
| AutoGen | Conversation-based (group chat) | Yes | Yes |
| Semantic Kernel | Plugin-based (kernel + functions) | Yes | Yes |
| Haystack | Pipeline-based (components + DAG) | Yes | Yes |
| Agno/Phidata | Declarative (agent + tools + team) | Yes | Yes |
| Google ADK | Agent-based (LLM + workflow agents) | Yes | Yes |
| Lyzr ADK | Declarative (studio + agent + tools) | Yes | Yes |
| Lyzr SuperFlow | Graph-based workflow (nodes + typed JSON edges) | Yes | No (source-only) |

## Structure

```
framework-translator-agent/
├── agent.yaml
├── SOUL.md
├── RULES.md
├── README.md
├── knowledge/
│   ├── index.yaml
│   ├── concept-mapping.md
│   └── framework-overview.md
└── skills/
    ├── analyze/
    │   ├── SKILL.md
    │   └── references/
    │       ├── langgraph-patterns.md
    │       ├── crewai-patterns.md
    │       ├── openai-agents-patterns.md
    │       ├── autogen-patterns.md
    │       ├── semantic-kernel-patterns.md
    │       ├── haystack-patterns.md
    │       ├── agno-patterns.md
    │       ├── google-adk-patterns.md
    │       ├── lyzr-adk-patterns.md
    │       └── superflow-patterns.md      # Lyzr SuperFlow (source workflow JSON)
    ├── translate/
    │   ├── SKILL.md
    │   └── references/
    │       ├── langgraph-codegen.md
    │       ├── crewai-codegen.md
    │       ├── openai-agents-codegen.md
    │       ├── autogen-codegen.md
    │       ├── semantic-kernel-codegen.md
    │       ├── haystack-codegen.md
    │       ├── agno-codegen.md
    │       ├── google-adk-codegen.md
    │       └── lyzr-adk-codegen.md
    └── validate/
        └── SKILL.md
```

## Built with

[gitagent](https://github.com/open-gitagent/gitagent) — a git-native, framework-agnostic open standard for AI agents.
