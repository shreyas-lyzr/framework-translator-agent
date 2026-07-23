# Lyzr SuperFlow Framework Patterns Reference

Use this reference to identify, decompose, and extract CIR from a **Lyzr SuperFlow** workflow.

SuperFlow is not source *code* — it is a **JSON workflow definition** (`flow_data`): a durable, DAG-based
visual workflow that chains agents, LLM calls, tools, HTTP, code, and control-flow nodes. Because it is a
directed graph of nodes and edges with typed state flowing along the edges, it maps **most faithfully to
LangGraph** (graph of node functions + conditional edges + a `TypedDict` state). Every mapping below gives
the **LangGraph** target first (the 1:1 path) and notes the adaptation for other frameworks.

> SuperFlow is a **source** here (translate SuperFlow → LangGraph / any framework). The input the user
> provides is a `flow_data` JSON object (from the SuperFlow API `GET /workflows/{id}` → `.flow_data`, or an
> exported workflow), **not** a `.py` file.

---

## 1. Detection Fingerprints (JSON, not imports)

A payload is SuperFlow when **any** of these hold:

- A JSON object with a top-level `flow_data`, **or** top-level `nodes` (array) + `connections` (object).
- Node objects whose `type` is prefixed **`lyzr-nodes-base.`** (e.g. `lyzr-nodes-base.trigger`,
  `lyzr-nodes-base.lyzr.llm`, `lyzr-nodes-base.if`).
- `connections` is an object **keyed by source node `name`** (not id), with `main` / `error` arrays of
  `{ node, type, index }` targets.
- Expressions written as **`{{ ... }}`** using `$json`, `$('NodeName')`, `$node["Name"]`, `$input`.
- Node objects shaped `{ id, name, type, typeVersion, parameters, position:[x,y] }`.

Report: `Detected framework: **Lyzr SuperFlow** (confidence: high)` when node `type`s carry the
`lyzr-nodes-base.` prefix. If you only see `nodes`+`connections` without the prefix, confidence is medium —
confirm with the user.

---

## 2. `flow_data` Anatomy

```jsonc
{
  "name": "Document Processing Flow",
  "nodes": [ /* Node[] — non-empty, EXACTLY ONE trigger */ ],
  "connections": { /* keyed by source node NAME */ },
  "settings": { /* workflow-level config */ }
}
```

### Node object

```jsonc
{
  "id": "n1",                          // unique id
  "name": "Classify Ticket",           // human label — THIS is what connections + expressions reference
  "type": "lyzr-nodes-base.lyzr.llm",  // node type (see §4)
  "typeVersion": 1,
  "parameters": { /* node config; strings may contain {{ }} expressions */ },
  "position": [520, 160],              // editor coords — IGNORE for translation
  "credentials": { /* optional */ },
  "settings": { "retryOnFail": true, "handleErrors": true /* optional per-node */ }
}
```

### Connections

Keyed by the **source node `name`**; `main` and `error` are arrays indexed by **output slot**, each slot a
list of target objects:

```jsonc
"connections": {
  "Classify Ticket": {
    "main":  [ [ { "node": "Route", "type": "main", "index": 0 } ] ],
    "error": [ [ { "node": "Notify Ops", "type": "main", "index": 0 } ] ]  // only if handleErrors:true
  }
}
```

- **Output slot index** matters for multi-output nodes: `If` uses slot `0`=true, `1`=false; `Switch` uses one
  slot per rule; `Wait for Approval` uses `0`=approved, `1`=rejected; `Loop` uses `0`=loop-body, `1`=done.
- **Error edges** (`error[...]`) fire only when a node sets `handleErrors: true` and fails after retries.

---

## 3. State & Data-Flow Model (the key 1:1 idea)

SuperFlow passes **JSON "items" (arrays of objects)** along each edge. Inside a node:

- `{{ $json }}` / `{{ $json.field }}` = the **current input item** (the upstream node's output).
- `{{ $('NodeName').json.field }}` = the stored output of **any named upstream node** (not just the direct
  parent).

**Faithful LangGraph state:** model a `TypedDict` where **each node name is a channel** holding that node's
last output, plus an `input` channel for the trigger payload. This makes `$('NodeName')` references resolve
directly to `state["NodeName"]`:

```python
class FlowState(TypedDict, total=False):
    input: list[dict]                 # trigger items ($json at the entry)
    # one channel per node name (spaces/punct → snake_case), holding that node's output items:
    classify_ticket: list[dict]
    route: list[dict]
    # ...
```

> **TRANSLATION NOTE convention:** SuperFlow items are **arrays**; most expressions implicitly target the
> **first item**. When emitting target code, model channels as `list[dict]` and index `[0]` where the
> expression is scalar, or map over the list where the node fans out per item.

### Expression translation table

| SuperFlow expression | Meaning | LangGraph / Python target |
|---|---|---|
| `{{ $json }}` | current input item | `state["<parent_channel>"][0]` (or the item in a per-item map) |
| `{{ $json.user.email }}` | field of current item | `state["<parent>"][0]["user"]["email"]` |
| `{{ $('Search').json.results }}` | named node output | `state["search"][0]["results"]` |
| `{{ $node["Search"].json }}` | named node output (alt syntax) | `state["search"][0]` |
| `{{ $input.all() }}` | all input items | `state["<parent>"]` |
| `{{ $json.price * 1.18 }}` | JS expression | `state[...][0]["price"] * 1.18` |
| `{{ a ? 'x' : 'y' }}` | JS ternary | Python conditional expression |

Gotchas to preserve as `# TRANSLATION NOTE:` — missing fields resolve to `null` (use `.get()`); a syntax
error leaves the literal `{{ }}` text; **`$items` is NOT valid in expressions** (only inside a Code node).

---

## 4. Node Catalog → CIR + LangGraph Mapping (1:1)

Types are given without the `lyzr-nodes-base.` prefix. **Fidelity:** Direct / Adapted / Lossy / Unsupported.

### 4a. Control flow

| SuperFlow node (`type`) | Behavior | LangGraph target | Fidelity |
|---|---|---|---|
| **Trigger** (`trigger`) | Single entry; emits input items | `START` edge + initial `state["input"]`; see §5 for manual/webhook/schedule | Direct |
| **If** (`if`) | Route to true/false by rule or AI | `add_conditional_edges(node, cond, {"true": a, "false": b})`; output slot 0=true, 1=false | Direct |
| **Switch** (`switch`) | First-match multi-route | `add_conditional_edges(node, router, {rule_i: target_i})` | Direct |
| **Merge** (`merge`) | Combine multiple inputs (append/position/key/SQL) | Fan-in node with a reducer channel (`Annotated[list, operator.add]`) or a merge node function | Adapted |
| **Loop** (`splitInBatches`) | Iterate per item/batch | `Send` map-reduce **or** a cyclic subgraph: body node → back-edge via conditional edge until exhausted; slot 0=body, 1=done | Adapted |
| **Filter** (`filter`) | Keep matching items | Node returning the filtered `list` (`[i for i in items if cond]`) | Direct |
| **Wait** (`wait`) | Pause fixed duration | Node `time.sleep()` for short waits | Lossy — SuperFlow waits are **durable/free**; plain `sleep` holds the process. Note it. |
| **Stop and Error** (`stopAndError`) | Halt with error | `raise RuntimeError(msg)` in a node (or `Command(goto=END)`) | Direct |
| **No-Op** (`noOp`) | Pass-through | Omit, or a pass-through node returning input unchanged | Direct |

### 4b. AI

| SuperFlow node | Behavior | LangGraph target | Fidelity |
|---|---|---|---|
| **LLM** (`lyzr.llm`) | Single call or ReAct loop w/ sub-agent tools | Node: `ChatModel.invoke([SystemMessage(systemPrompt), HumanMessage(query)])`. If `maxIterations>1` or sub-agents present → `create_react_agent(model, tools)` | Direct (single) / Adapted (loop) |
| **AI Agent** (`lyzr.agent`) | Run a managed Lyzr agent by id | Reconstruct as an LLM/ReAct node using the agent's model+prompt+tools, **or** a node that calls the Lyzr agent API | Adapted — hidden KB/memory/guardrails on the managed agent are **Lossy**; emit `# TRANSLATION NOTE:` |
| **Task Decomposition / Swarm** (`lyzr.taskDecomposition`) | Split → parallel subtasks → aggregate | Fan-out `add_conditional_edges(planner, fan_out, [worker...])` → `aggregator` (or `Send`) | Adapted |
| **Tool** (`lyzr.tool`) | Direct platform tool call | A `@tool` function / `ToolNode` (or HTTP call to the tool) | Adapted |
| **A2A Agent** (`lyzr.a2aAgent`) | Call remote Agent2Agent server | Node doing the A2A HTTP call (preserve endpoint) | Adapted |
| **Sub-agent** (any AI node flagged `Run as sub-agent`, purple edge) | Callable tool of a parent LLM | Register as a tool bound to the parent LLM node (parent = ReAct loop) | Adapted |

### 4c. Compute & I/O

| SuperFlow node | Behavior | LangGraph target | Fidelity |
|---|---|---|---|
| **HTTP Request** (`httpRequest`) | Auth'd HTTP (Bearer/Basic/API-key/OAuth2/SigV4/JWT/mTLS) | Node using `httpx`/`requests`; reproduce method/url/headers/body/auth | Direct — SigV4/OAuth2 auto-refresh/mTLS are **Adapted** (add the client lib) |
| **Code** (`code`) | Sandboxed **JavaScript** (10 s); `$input/$json/$items`; last expr = output | Re-express the JS logic **in Python** inside a node | Adapted / Lossy — JS→Python is a rewrite; flag non-trivial logic |
| **Execute Workflow** (`executeWorkflow`) | Run a sub-workflow | Compiled **sub-graph** added as a node (`add_node("sub", subgraph)`) | Direct |

### 4d. Data transform

| SuperFlow node | LangGraph target (node body) | Fidelity |
|---|---|---|
| **Set / Edit Fields** (`set`) | return dict with assigned channels (`{k: expr}`) | Direct |
| **Aggregate** (`aggregate`) | reduce items into one (`aggregateIndividualFields` / `aggregateAllItemData`) | Adapted |
| **Sort** (`sort`) | `sorted(items, key=...)` (multi-field) | Direct |
| **Limit** (`limit`) | `items[:n]` / `items[-n:]` per `keep` | Direct |
| **Remove Duplicates** (`removeDuplicates`) | dedupe by all/selected fields | Direct |
| **Rename Keys** (`renameKeys`) | key remap | Direct |
| **TOON** (`toon`) | encode/decode JSON↔TOON (needs a lib) | Adapted |

### 4e. Utility, Document, Safety, HITL

| SuperFlow node | LangGraph target | Fidelity |
|---|---|---|
| **Date & Time** (`dateTime`) | `datetime`/`zoneinfo` in a node | Direct |
| **Crypto** (`crypto`) | `hashlib`/`hmac`/`cryptography` in a node | Direct |
| **XML** (`xml`) | `xmltodict` / stdlib in a node | Direct |
| **Parse** (`lyzr.parse`) | doc-parsing lib or API call (standard/advanced/agentic tiers) | Adapted / Lossy — the managed parser tier is external |
| **Extraction** (`lyzr.extraction`) | LLM structured-output node bound to the JSON Schema | Adapted |
| **Label / Classify** (`lyzr.label`) | LLM classification node against the rule set | Adapted |
| **Guardrails** (`lyzr.guardrails`) | Guardrail node (`add_guardrail` in LG v2.0) or a validation node that routes on failure | Adapted |
| **Wait for Approval** (`waitForApproval`) | **`interrupt({...})`** (LG v2.0) / `interrupt_before=[node]`. Approved→slot 0 edge, rejected→slot 1; merge form values into state | Direct (HITL) |

---

## 5. Triggers → Entry Point

Exactly one `trigger` node. Its **mode** determines how the LangGraph app is driven:

| Trigger mode | LangGraph translation |
|---|---|
| **Manual** | `app.invoke({"input": items}, config)` with a `thread_id` |
| **Webhook** (POST + `X-Webhook-Secret`) | Wrap `app.invoke` in a FastAPI/Flask route; verify the secret header. `# TRANSLATION NOTE:` re-create the endpoint |
| **Schedule** (cron/hourly/daily/…+ timezone) | Drive `app.invoke` from a scheduler (APScheduler/cron). `# TRANSLATION NOTE:` SuperFlow schedules are **crash-safe / never-miss-a-tick**; a plain cron is best-effort |

Downstream references to the trigger (`{{ $('Trigger').json.field }}`) become `state["trigger"][0]["field"]`
(or `state["input"][0][...]`).

---

## 6. Reliability Semantics → LangGraph

SuperFlow guarantees are stronger than plain LangGraph; preserve intent and flag the gap:

| SuperFlow feature | LangGraph mapping | Note |
|---|---|---|
| Durable journaling + **exactly-once** per step | Checkpointer (`MemorySaver`/`SqliteSaver`/`PostgresSaver`) enables resume | LG resumes from last checkpoint, not per-step exactly-once → `# TRANSLATION NOTE:` |
| **Per-node retries** + backoff (durable) | `add_node(..., retry=RetryPolicy(max_attempts=n))` or try/except loop | Retry counters aren't durable across process death in plain LG |
| **Error edges** (`handleErrors`) | try/except in the node routing to the error target via `Command(goto=...)` or an `error` channel + conditional edge | Direct-ish |
| Resource-free durable **waits/approvals** | `interrupt()` + checkpointer (resume later) | Approval is Direct; long timed waits are Lossy |
| OTel tracing (tokens/cost) | LangSmith / OTel callbacks | Optional |

---

## 7. Connections → Edges (rules)

1. For each source node `Name`, for each `main[slot][k] = {node: Target}`:
   - **single main slot** → `graph.add_edge("Name", "Target")`.
   - **`If`** (slots 0/1) → `add_conditional_edges("Name", cond, {"true":"T","false":"F"})`.
   - **`Switch`** (slots 0..n) → `add_conditional_edges("Name", router, {rule_i: target_i})`.
   - **`Wait for Approval`** (0=approved,1=rejected) → conditional on the resume value.
   - **`Loop`** (0=body,1=done) → body edges + a back-edge; `add_conditional_edges` on "more items?".
2. `error[slot]` targets (only when `handleErrors:true`) → error routing (see §6).
3. A node with **no outgoing** `main` → `add_edge("Name", END)`.
4. The `trigger` → `add_edge(START, "<first node>")`.
5. Fan-out (one source → multiple targets in a slot) → multiple `add_edge`s (parallel) into a later merge.

---

## 8. Decomposition Guide — Extracting CIR from `flow_data`

1. **Confirm it's SuperFlow** (§1). Parse the JSON; locate `nodes` and `connections`.
2. **Find the trigger.** Record its mode → CIR entry point (§5).
3. **Inventory nodes.** For each node record `name`, `type` (strip prefix), and `parameters`. Classify by
   category (§4): AI agent/LLM → CIR *Agent Inventory*; tool/http/code → CIR *Tool Inventory*; set/aggregate/
   sort/etc. → data transforms; if/switch/merge/loop → orchestration; waitForApproval → HITL.
4. **Extract Agent Inventory.** For each `lyzr.llm` / `lyzr.agent`: name, purpose, **verbatim** `systemPrompt`
   / agent id, `provider`/`model`, and the sub-agent tool set (downstream nodes flagged *Run as sub-agent*).
5. **Extract Tool Inventory.** `httpRequest` (method/url/auth/body), `code` (the JS body → note for rewrite),
   `lyzr.tool`, document nodes. Mark `External? = Yes` for HTTP/document/managed nodes.
6. **Extract State Schema.** One channel per node name (holds its output) + `input`/`trigger`. Record every
   `{{ $('X') }}` cross-reference so no channel is dropped.
7. **Extract Orchestration Graph.** Walk `connections` (§7). Draw the ASCII diagram: sequential chains,
   `If`/`Switch` branches (with the rule text), `Loop` cycles, fan-out/merge, approval pauses.
8. **Extract HITL.** Each `waitForApproval`: message, form schema (fields/types), notified emails, and the
   approved/rejected targets → CIR `hitl{}`.
9. **Extract External Dependencies.** HTTP endpoints + auth, document/parse services, `envs`/credentials,
   Lyzr agent/tool ids.
10. **Emit CIR** (per `knowledge/concept-mapping.md`): `agents[]`, `tools[]`, `edges[]`, `state{}` (channels),
    `memory{}` (checkpointer), `hitl{}` (interrupts), `architecture` (label: linear pipeline / branch /
    map-reduce / approval-gated / cyclic).

Flag every **Lossy/Unsupported** item (managed-agent KB/memory, durable waits, exactly-once, JS Code bodies,
Lyzr document tiers) as a `# TRANSLATION NOTE:` in the generated code.

---

## 9. Worked Example (flow_data → CIR → LangGraph)

**Input `flow_data`** (trigger → classify LLM → If → two branches):

```jsonc
{
  "name": "Ticket Router",
  "nodes": [
    { "id":"t","name":"Trigger","type":"lyzr-nodes-base.trigger","typeVersion":1,"parameters":{"mode":"webhook"},"position":[0,0] },
    { "id":"c","name":"Classify","type":"lyzr-nodes-base.lyzr.llm","typeVersion":1,
      "parameters":{"provider":"openai","model":"gpt-4o-mini",
        "systemPrompt":"Classify the ticket as 'billing' or 'tech'. Reply with one word.",
        "query":"{{ $('Trigger').json.message }}"},"position":[220,0] },
    { "id":"i","name":"Route","type":"lyzr-nodes-base.if","typeVersion":1,
      "parameters":{"conditions":[{"left":"{{ $('Classify').json.response }}","op":"equals","right":"billing"}]},"position":[440,0] },
    { "id":"b","name":"Billing","type":"lyzr-nodes-base.lyzr.agent","typeVersion":1,"parameters":{"agentId":"agt_billing","query":"{{ $('Trigger').json.message }}"},"position":[660,-80] },
    { "id":"h","name":"Tech","type":"lyzr-nodes-base.lyzr.agent","typeVersion":1,"parameters":{"agentId":"agt_tech","query":"{{ $('Trigger').json.message }}"},"position":[660,80] }
  ],
  "connections": {
    "Trigger": { "main": [[{"node":"Classify","type":"main","index":0}]] },
    "Classify":{ "main": [[{"node":"Route","type":"main","index":0}]] },
    "Route":   { "main": [ [{"node":"Billing","type":"main","index":0}], [{"node":"Tech","type":"main","index":0}] ] }
  }
}
```

**CIR (abridged):** entry=webhook; agents=[Classify (gpt-4o-mini, prompt verbatim), Billing (agt_billing),
Tech (agt_tech)]; edges: START→Classify→Route; Route --billing--> Billing, --else--> Tech; both → END;
state channels: input, trigger, classify, route.

**Architecture diagram:**

```
[START] -> [Classify(LLM)] -> [Route(If)] --response=="billing"--> [Billing(agent)] -> [END]
                                          --else----------------> [Tech(agent)]    -> [END]
```

**LangGraph (target — via `translate/references/langgraph-codegen.md`):**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage

class FlowState(TypedDict, total=False):
    trigger: dict          # webhook payload ($('Trigger').json)
    classify: dict         # Classify node output
    result: str

llm = ChatOpenAI(model="gpt-4o-mini")

def classify(state: FlowState) -> dict:
    msg = state["trigger"]["message"]                       # {{ $('Trigger').json.message }}
    r = llm.invoke([SystemMessage("Classify the ticket as 'billing' or 'tech'. Reply with one word."),
                    HumanMessage(msg)])
    return {"classify": {"response": r.content.strip().lower()}}

def route(state: FlowState) -> str:                         # If node → conditional edge
    return "billing" if state["classify"]["response"] == "billing" else "tech"

def billing(state: FlowState) -> dict:
    # TRANSLATION NOTE: SuperFlow AI Agent 'agt_billing' is a managed Lyzr agent (its model, prompt,
    # tools, KB and memory live in Lyzr). Reconstruct or call the Lyzr agent API to preserve behavior.
    return {"result": call_lyzr_agent("agt_billing", state["trigger"]["message"])}

def tech(state: FlowState) -> dict:
    return {"result": call_lyzr_agent("agt_tech", state["trigger"]["message"])}

g = StateGraph(FlowState)
g.add_node("classify", classify)
g.add_node("billing", billing)
g.add_node("tech", tech)
g.add_edge(START, "classify")
g.add_conditional_edges("classify", route, {"billing": "billing", "tech": "tech"})
g.add_edge("billing", END)
g.add_edge("tech", END)
app = g.compile()
# TRANSLATION NOTE: Trigger mode=webhook — wrap app.invoke in a POST endpoint that checks X-Webhook-Secret.
```

---

## 10. Cheat-Sheet

- **Graph, not code** → LangGraph is the natural 1:1 target; other targets adapt the graph (CrewAI Flows
  `@start/@listen/@router`, Haystack `Pipeline.connect`, OpenAI Agents handoffs, ADK `SequentialAgent`).
- **connections keyed by node name** → renaming a node rewires edges + breaks `{{ $('Name') }}`.
- **Items are arrays; expressions target item[0]** → model channels as lists; index/​map accordingly.
- **One channel per node name** so `$('Node')` cross-references survive.
- **Managed nodes (agent/parse/extraction/guardrails) are external** → Adapted/Lossy; never silently drop.
- **waitForApproval → `interrupt()`**; **error edges → try/except routing**; **schedule → cron + note**.
