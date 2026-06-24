# Google ADK Agent Patterns for Threat Intel KB

Audience: an ADK engineer agent building the **reasoning plane**. ADK = Google Agent Development Kit
(open-source, Python). ADK agents handle ONLY judgment, natural language, ambiguity, and
synthesis/correlation. Deterministic plumbing stays in Kestra or in non-LLM ADK `FunctionTool`s.
Reuse is the rule: ~6-10 parameterized agents serve all 60 playbooks — not 60 agents.

---

## 1. Agent types

| Type | Use |
|---|---|
| `LlmAgent` | Core reasoning unit: instruction + tools + model. The triage/writer/correlation brains. |
| `SequentialAgent` | Run sub-agents in fixed order (pipeline: gather → investigate → report). |
| `ParallelAgent` | Run sub-agents concurrently (fan-out enrichment). |
| `LoopAgent` | Repeat sub-agent(s) until a condition/escalation (iterative pivoting). |
| custom `BaseAgent` | Bespoke control logic when workflow agents don't fit. |
| multi-agent via `sub_agents` | Compose agents into a hierarchy; parent delegates. |

---

## 2. Tools

| Tool | Use |
|---|---|
| `FunctionTool` | Plain typed Python fn + docstring; ADK auto-generates the schema from signature/types/docstring. The workhorse — wraps deterministic enrichment/CRUD. |
| Built-in tools | e.g. code execution, search (where permitted). |
| OpenAPI tools | Generate tools from an OpenAPI spec (e.g. a vendor API). |
| `MCPToolset` | Connect Model Context Protocol servers as tool sources. |
| agent-as-a-tool | Wrap an agent so another agent can call it as a tool. |
| long-running tools | Tools that return progressively / await external completion. |

`tool_context: ToolContext` parameter gives a tool access to `state`, artifacts, and `actions`
(e.g. `tool_context.actions.escalate`, transfer). Use it to read an `approved` flag or write
artifacts.

---

## 3. Sessions / State / Memory / Artifacts

| Facility | Notes |
|---|---|
| `session.state` | Mutable dict scoped to a session. Prefixes: `app:` (app-wide), `user:` (per-user), `temp:` (not persisted). |
| `SessionService` | `InMemorySessionService` (dev); **`DatabaseSessionService` (prod)** backed by a local/on-prem PostgreSQL. `VertexAiSessionService` is GCP-only — not used here. |
| `MemoryService` | Long-term recall across sessions. |
| `ArtifactService` | Binary/file storage (TC `Binary`/file content, reports) — keep out of LLM context. |
| `output_key` | Writes an agent's final output into `session.state[output_key]`. |
| `output_schema` | A Pydantic model forcing **structured** output (verdicts, dispositions). |

---

## 4. Models — remote Azure + Bedrock via the internal ADK wrapper (no Gemini/Vertex)

Models are **remote** (Azure OpenAI + AWS Bedrock + others); none run locally. Agents do NOT
configure `LiteLlm`, credentials, or endpoints directly — they reach models through the **internal
SDK wrapper on top of ADK**, requesting a **role tier** (e.g. `triage`, `reasoning`,
`writer`). The wrapper sits where `LiteLlm` would and owns all model plumbing; leave it as-is.

So in agent code, take the model object from the wrapper rather than constructing one:

```python
from internal_adk import model_for   # the internal wrapper; resolves an approved deployment by tier

triage_model = model_for("triage")     # -> small/fast model (e.g. Claude Haiku 4.5 on Bedrock)
synth_model  = model_for("reasoning")  # -> large model (e.g. Claude Sonnet/Opus 4.x on Bedrock)
```

**Which model per tier is in `deployment-constraints-kb.md` (§3 recommendation matrix).**
Discipline: small/fast model for triage and dedup (~80% of calls); large model only for
synthesis/narrative; Opus-class/reasoning models only for archetype-C correlation. Default family is
Claude on Bedrock for security reasoning; Azure OpenAI as the per-role alternative.

---

## 5. Callbacks as guardrails

| Callback | Guardrail use |
|---|---|
| `before_model_callback` | Redact PII before it reaches the LLM; enforce prompt allow-lists. |
| `before_tool_callback` | **Block production writes unless `state['approved']` is true**; enforce tool allow-lists. |
| `after_tool_callback` | Sanitize/validate tool results; redact sensitive fields. |

```python
def block_unapproved_writes(tool, args, tool_context):
    if tool.name.startswith("tc_write") and not tool_context.state.get("approved"):
        return {"error": "write blocked: no approval flag in state"}
    return None  # allow
```

Approval flags are set by the Kestra `Pause` gate, not by the agent itself.

---

## 6. Runners, event loop, deployment, eval

- **`Runner`** drives the agent **event loop**: yields `Event`s (model calls, tool calls, state
  deltas) until final response. The HTTP wrapper (pattern A in `kestra-patterns-kb.md`) calls the
  Runner.
- **Deployment (local/on-prem, no GCP):** Agent Engine/Cloud Run are GCP-only and **not used**. The
  ADK Runner is a plain Python app — containerize it (FastAPI/uvicorn) and run it **on-prem,
  co-located with the local Kestra**, so the Kestra→ADK HTTP hop stays internal. Only the model call
  egresses (via the internal wrapper). See `deployment-constraints-kb.md`.
- **Evaluation framework:** golden datasets (input → expected tool calls / final response);
  eval-gate promotion — an agent ships only after passing its eval set.

---

## 7. Three reusable TI agent archetypes

```python
from google.adk.agents import LlmAgent, SequentialAgent, ParallelAgent, LoopAgent
from google.adk.tools import FunctionTool
from google.adk.models.lite_llm import LiteLlm
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from pydantic import BaseModel
```

**(1) Triage / disposition** — `LlmAgent` with a Pydantic `output_schema`. One parameterized
instance serves dozens of playbooks.

```python
class TriageResult(BaseModel):
    verdict: str            # malicious | suspicious | benign
    confidence: float       # 0..1
    severity: int           # 1..5
    recommended_action: str
    rationale: str

triage_agent = LlmAgent(
    name="triage",
    model=LiteLlm(model="bedrock/anthropic.claude-3-5-haiku-20241022-v1:0"),  # small model
    instruction="Assess the indicator and enrichment context. Return a disposition.",
    output_schema=TriageResult,
    output_key="triage_result",
)
```

**(2) Enrichment orchestrator** — `ParallelAgent` gathers VT/Shodan/OTX concurrently, feeding a
`SequentialAgent` that investigates.

```python
def vt_lookup(ioc: str) -> dict: ...      # deterministic FunctionTool
def shodan_lookup(ip: str) -> dict: ...
def otx_lookup(ioc: str) -> dict: ...

gather = ParallelAgent(
    name="gather",
    sub_agents=[
        LlmAgent(name="vt", model=..., tools=[FunctionTool(vt_lookup)], output_key="vt"),
        LlmAgent(name="shodan", model=..., tools=[FunctionTool(shodan_lookup)], output_key="shodan"),
        LlmAgent(name="otx", model=..., tools=[FunctionTool(otx_lookup)], output_key="otx"),
    ],
)
investigate = SequentialAgent(name="investigate", sub_agents=[gather, triage_agent])
```

**(3) Narrative / report writer** — `LlmAgent` (large model) turning structured findings into analyst prose.

```python
report_agent = LlmAgent(
    name="narrative",
    model=LiteLlm(model="bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0"),  # large model
    instruction="Write a concise analyst report from {triage_result} and enrichment state.",
    output_key="report",
)
```

Run via `Runner` over an `InMemorySessionService` (dev) / `DatabaseSessionService` on local/on-prem Postgres (prod).

---

## 8. The reuse principle

~6-10 **reusable, parameterized** agents serve all ~60 playbooks. A single triage agent,
parameterized by instruction/thresholds/owner, serves dozens. Refuse one-agent-per-playbook.

**Candidate reusable agent library:**

| Agent | Role |
|---|---|
| `triage` | Disposition/severity verdict (structured output). |
| `enrichment-orchestrator` | Parallel gather + sequential investigate. |
| `narrative-writer` | Analyst report / case summary prose. |
| `correlation/pivot` | `LoopAgent` pivoting across related IOCs/groups. |
| `dedup-judge` | Fuzzy/ambiguous dedup (exact dedup stays deterministic). |
| `case-summarizer` | Summarize a TC Case for handoff/closure. |

Everything else — regex, JSON-path, exact dedup, API calls, CRUD, iteration — is a deterministic
Kestra task or non-LLM `FunctionTool`. See `agentic-decision-kb.md`.
