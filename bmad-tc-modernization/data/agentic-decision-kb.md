# Deterministic-vs-Agent Decision Rules KB

Audience: the agentic architect agent. This is the **most important judgment file** in the pack.
It decides which ThreatConnect playbook nodes become deterministic control-plane plumbing and which
become probabilistic reasoning-plane agents.

**Two-plane architecture:**
- **Kestra = control plane (deterministic):** triggers, DAG, fan-out, retries, rate-limits,
  approval gates, audit, secrets, backfills.
- **ADK = reasoning plane (probabilistic):** triage, correlate, summarize, decide disposition,
  draft reports.

---

## 1. The default is deterministic

A node becomes an **ADK LLM agent ONLY IF** it requires at least one of:

1. **Judgment** — a verdict/disposition that isn't a fixed rule.
2. **Natural language** — parsing or generating free text.
3. **Ambiguity resolution** — messy/contradictory inputs needing interpretation.
4. **Synthesis / correlation** — combining multiple sources into a conclusion.

Otherwise the node is a **deterministic Kestra task** or an **ADK `FunctionTool` (no LLM)**. Most
playbook nodes — transforms, API calls, iteration, branching, CRUD — are deterministic plumbing and
**must NOT become agents.**

---

## 2. Node → target rules

| Node | Target | Why |
|---|---|---|
| Regex / JSON-path / JMESPath / string transform | Deterministic tool / Kestra task | Pure function, no judgment. |
| API enrichment call (VT/Shodan/Splunk/ServiceNow) | Deterministic `FunctionTool` | The *call* is deterministic; interpretation is separate. |
| "Is this malicious / what severity?" | **Triage agent** | Judgment. |
| "Summarize for analyst" / draft report | **Narrative/writer agent** | Natural language generation. |
| CRUD to TC (create/update indicator/group/tag) | Deterministic tool (behind approval gate) | Mechanical write. |
| Iterate over an array | Kestra `EachParallel`/`EachSequential` | Control flow, not reasoning. |
| Branch on a known condition | Kestra `If`/`Switch` | Deterministic logic. |
| **Exact** dedup (type+value+owner) | Deterministic tool | Rule-based. |
| **Fuzzy** dedup / entity resolution with judgment | Could be **dedup-judge agent** | Ambiguity. |
| Parse free-text email/report body | **Agent** (then deterministic downstream) | Natural language. |
| Correlate across multiple intel sources | **Correlation/pivot agent** | Synthesis. |
| Sleep / delay / merge / set variable | Kestra | Plumbing. |

---

## 3. Per-node scoring rubric

Score every node; the verdict falls out of the columns.

| node | needs judgment? | needs NL? | ambiguous? | deterministic alt exists? | verdict |
|---|---|---|---|---|---|
| Regex extract domains | no | no | no | yes | Kestra task |
| Call VirusTotal | no | no | no | yes | ADK tool (no LLM) |
| Create indicator in TC | no | no | no | yes | ADK tool (gated) |
| Iterate over IOC list | no | no | no | yes | Kestra EachParallel |
| Decide malicious/severity | yes | no | yes | no | **ADK agent** |
| Summarize incident | no | yes | no | no | **ADK agent** |
| Fuzzy-match duplicate adversaries | yes | no | yes | partial | **ADK agent** |
| Parse phishing email body | no | yes | yes | no | **ADK agent** |

**Rule:** any "yes" in judgment / NL / ambiguous **and** no clean deterministic alternative →
ADK agent. All-no or a deterministic alternative exists → Kestra task / ADK tool.

---

## 4. The 4 playbook archetypes

| Archetype | ~Pop. | Description | Target stack | Token cost |
|---|---|---|---|---|
| **A — pure deterministic ETL/enrichment** | ~40% | ingest → transform → enrich → write. No judgment. | **Kestra only**, zero agents. | **Zero** |
| **B — enrich + decide** | ~35% | enrich, then one disposition decision. | Kestra + **one reusable triage agent**. | Low (small model) |
| **C — multi-step investigation / correlation** | ~12% | pivot/correlate across sources, synthesize. | **ADK multi-agent** (orchestrator/correlation) + Kestra. | Higher |
| **D — human-workflow / case management** | ~10% | case lifecycle with human steps. | Kestra workflow + **Pause** + light agent assist (case-summarizer). | Low |

~75% of playbooks (A+B) use **zero or one** agent. Heavy multi-agent work is the minority (C).

---

## 5. The agent-count math

~6-10 **reusable, parameterized** agents serve **all ~60 playbooks** — **NOT 60 agents.**

- One `triage` agent (parameterized by instruction/threshold/owner) serves every archetype-B
  playbook (dozens).
- `enrichment-orchestrator`, `narrative-writer`, `correlation/pivot`, `dedup-judge`,
  `case-summarizer` cover the rest.

Reuse comes from **parameterization**, not duplication. If your design has more than ~10 distinct
agents, you are building one-per-app — stop and consolidate.

---

## 6. Anti-patterns to refuse

| Anti-pattern | Why it's wrong |
|---|---|
| One agent per app/node | Explodes count, cost, and eval surface. Reuse instead. |
| LLM doing regex / JSON-path / CRUD | Non-deterministic, slow, costly for a solved problem. |
| Skipping the shared TC model / tool libraries | Duplicated, drifting integration code. |
| Agents holding credentials | Violates least privilege; the TC client service holds creds. |
| Agents writing to TC without an approval gate | No human control over destructive/outbound actions. |
| No shadow-run before cutover | Unverified behavior hits production. |

---

## 7. Cost & governance guardrails

- **Archetype A = zero token cost** — never invoke an LLM in pure ETL.
- **Cache enrichment** results (per IOC/TTL) to avoid repeat provider + token spend.
- **Small model for triage, large model only for synthesis/narrative.**
- **Eval-gated promotion** — an agent ships only after passing its golden dataset.
- **Least privilege** — a dedicated TC client service holds TC creds; **agents never do**. Provider
  keys live as Kestra secrets / Secret Manager.
- **Approval gates** — `Pause` before every destructive/outbound action; `before_tool_callback`
  blocks writes unless `state['approved']`.
- **Shadow run** every flow (write-backs stubbed) before cutover; compare to the legacy playbook.

---

## 8. Decision flow (summary)

```
For each playbook node:
  1. Does it move/shape data, call an API, branch, iterate, or CRUD?
        -> Deterministic. Kestra task or ADK FunctionTool (no LLM). DONE.
  2. Does it need judgment, NL, ambiguity resolution, or cross-source synthesis,
     with NO clean deterministic alternative?
        -> ADK LLM agent. Reuse an existing archetype; parameterize. Gate any writes.
  3. Otherwise -> deterministic by default.
```

Deterministic stays deterministic. Agents are scarce, reusable, gated, and reserved for genuine
ambiguity and judgment.
