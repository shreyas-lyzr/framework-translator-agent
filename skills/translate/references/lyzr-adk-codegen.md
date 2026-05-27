# Lyzr ADK Code Generation Reference

> Target: `lyzr-adk` >= 0.1.x (2026 API). Install: `pip install lyzr-adk`.
> Import surface is `from lyzr import Studio`.

Generate idiomatic Lyzr ADK code from the CIR. Lyzr ADK is declarative and
studio-managed — a `Studio` owns agents, tools, knowledge bases (RAG), memory,
and guardrails.

---

## 1. Required Imports

```python
from lyzr import Studio          # the only import you usually need
# (optional) from pydantic import BaseModel   # for structured outputs
```

## 2. Minimal Agent

```python
from lyzr import Studio

studio = Studio()
agent = studio.create_agent(
    name="<agent name>",
    provider="openai/gpt-4o",        # "<provider>/<model>"; map from CIR model
    role="<persona / role>",         # from CIR role
    goal="<what it should achieve>", # from CIR goal
    instructions="<system prompt>",  # from CIR system instructions
)

response = agent.run("<user message>")
print(response.response)             # the text reply is response.response
```

## 3. Tools (no decorators)

Map each CIR tool to a plain Python function whose **docstring is the description**,
then register it with `add_tool`:

```python
def get_weather(city: str) -> dict:
    """Return the current weather for a city."""   # description for the LLM
    ...
    return {"temp_c": 21, "summary": "clear"}

agent.add_tool(get_weather)          # no @tool / @function_tool decorator needed
```

When translating FROM a decorator-based framework, strip the decorator and keep the
function + docstring + type hints.

## 4. RAG / Knowledge bases

```python
kb = studio.create_knowledge_base(name="docs")   # ingest documents into it
# ... add documents to kb ...
response = agent.run("Answer from the docs", knowledge_bases=[kb])
```

## 5. Memory (conversation context)

Pass a `session_id` so the agent remembers across turns:

```python
agent.run("Remember my name is Sam", session_id="user_123")
agent.run("What's my name?",         session_id="user_123")  # recalls "Sam"
```

## 6. Structured outputs

```python
from pydantic import BaseModel

class Ticket(BaseModel):
    summary: str
    priority: str

# pass the model so the agent returns a typed object
response = agent.run("Classify this issue: ...", response_model=Ticket)
```

## 7. Guardrails (RAI)

Lyzr ADK supports safety policies (toxicity, PII, secrets detection). Map any
CIR guardrails/compliance to the studio/agent's RAI policy configuration.

## 8. Multi-agent

Create multiple agents on the same studio and orchestrate between them:

```python
researcher = studio.create_agent(name="Researcher", provider="openai/gpt-4o",
                                 role="Researcher", goal="Gather facts", instructions="...")
writer     = studio.create_agent(name="Writer", provider="openai/gpt-4o",
                                 role="Writer", goal="Write the report", instructions="...")
facts  = researcher.run("Research X").response
report = writer.run(f"Write a report from: {facts}").response
```

## 9. Translation checklist (target = Lyzr ADK)

- [ ] One `Studio()` instance; one `create_agent(...)` per CIR agent.
- [ ] `role` / `goal` / `instructions` filled from the CIR persona/system prompt.
- [ ] Every CIR tool → plain function (+ docstring) + `agent.add_tool(...)`.
- [ ] CIR RAG → `knowledge_bases=[...]`; CIR memory → `session_id=`.
- [ ] CIR structured output → Pydantic `response_model`.
- [ ] Read replies via `response.response`.
- [ ] `provider` string composed as `"<provider>/<model>"` from the CIR model.
- [ ] If a Lyzr ADK feature has no source equivalent, omit it (don't invent); note unknowns and verify the API via web search before emitting.
