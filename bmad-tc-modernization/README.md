# BMAD Expansion Pack — ThreatConnect → Kestra + Google ADK Modernization

A BMAD expansion pack of expert agents and skills that **review ThreatConnect playbooks and custom TcEx apps in place** and produce a **deterministic-first conversion plan** to **Kestra** (orchestration) + **Google ADK** (agents). Everything runs locally against internal exports — nothing leaves the organization.

## The doctrine (read this first)

**Two planes.** Don't 1:1 port playbooks into agents.

- **Kestra = control plane (deterministic):** triggers, scheduling, the DAG, fan-out/iteration, retries/backoff, rate-limiting, human-approval gates, audit, secrets, backfills. Replaces the *wiring* of a playbook.
- **Google ADK = reasoning plane (probabilistic):** triage, correlation, summarization, disposition, report drafting. Replaces only the apps that *exercise judgment*.

**Deterministic stays deterministic.** Regex, JSON-path, API enrichment, and TC CRUD become deterministic Kestra tasks or ADK `FunctionTool`s — never LLM calls. An agent is justified **only** by judgment, natural language, or ambiguity.

**You fully exit TcEx.** Carry forward only knowledge, re-expressed as a clean TC v3 REST/Batch client.

**~6–10 reusable agents serve all 60 playbooks**, not 60 agents. One parameterized triage agent serves dozens.

**Stack: local Kestra, remote models, no GCP.** Kestra runs on-prem; the ADK runner is co-located on-prem and Kestra calls it over HTTP. Models are remote (Azure OpenAI + AWS Bedrock + others), reached through the internal internal **SDK wrapper over ADK** by tier — agents never configure model creds. Default family is Claude-on-Bedrock for security reasoning. Model-per-role recommendations and topology in `data/deployment-constraints-kb.md`.

| Archetype | Share | Target stack |
|---|---|---|
| A · pure deterministic ETL/enrichment | ~40% | Kestra only — zero agents, zero token cost |
| B · enrich + decide | ~35% | Kestra + one reusable triage agent |
| C · multi-step investigation/correlation | ~12% | ADK multi-agent + Kestra |
| D · human-workflow / case management | ~10% | Kestra + Pause gates + light agent assist |

## The team

| Agent | File | Role |
|---|---|---|
| 🎼 Convoy — Conductor | `agents/tc-conductor.md` | Orchestrator / conversion lead — routes work, owns the master plan, batching, cutover governance |
| 🏺 Digger — Playbook Archaeologist | `agents/tc-playbook-archaeologist.md` | Reverse-engineers playbook exports: trigger, apps, variables, components, TC writes |
| 🔬 Forge — Code Forensics | `agents/tc-code-forensics.md` | Reads custom TcEx app source, extracts logic, strips coupling |
| 🧭 Axiom — Agentic Architect | `agents/tc-agentic-architect.md` | The deterministic-vs-agent split, target design, guardrails |
| 🔀 Flux — Kestra Engineer | `agents/tc-kestra-engineer.md` | Flows, triggers, control flow, subflows, retries, approval gates |
| 🤖 Vertex — ADK Engineer | `agents/tc-adk-engineer.md` | ADK agents, tools, state, callbacks, deployment |
| 🛠️ Solo — TC-Convert (all-in-one) | `agents/tc-convert.md` | Single fallback agent wrapping every skill for quick one-off reviews |

Team bundle: `agent-teams/tc-modernization-team.yaml`.

## The skills (shared dependencies)

**Knowledge bases — `data/`**
- `tc-playbook-kb.md` — ThreatConnect playbook reference + variable typing + data model
- `tcex-coupling-kb.md` — TcEx patterns → clean replacements (what to strip)
- `kestra-patterns-kb.md` — TC constructs → Kestra flow patterns, with example flows
- `adk-patterns-kb.md` — ADK agent/tool patterns + the reusable TI agent archetypes
- `agentic-decision-kb.md` — the deterministic-vs-agent decision rules (the core judgment file)
- `deployment-constraints-kb.md` — **local Kestra, remote models, no GCP**: model-per-role recommendation matrix (Claude-on-Bedrock default; Azure OpenAI alternatives), access via the internal ADK wrapper by tier, on-prem topology, session/artifact stores

**Task workflows — `tasks/`**
`inventory-playbook` → `classify-archetype` → `decompose-logic` → `design-target` → `produce-conversion-plan` → `shadow-run-validation`

**Templates — `templates/`**
`playbook-analysis-tmpl.yaml`, `conversion-design-tmpl.yaml`, `migration-plan-tmpl.yaml`

**Checklists — `checklists/`**
`playbook-readiness-checklist.md`, `conversion-quality-checklist.md`

## How to use it

The lifecycle, per playbook, then rolled up:

1. **Inventory** (Archaeologist `*analyze`) — fill the analysis doc from the export.
2. **Code forensics** (Forge `*review-app`) — for any custom TcEx apps.
3. **Classify** (Architect `*archetype`) — assign A/B/C/D.
4. **Decompose** (Architect `*split`) — deterministic vs agent per node.
5. **Design** (Architect `*design` → Kestra `*flow` + ADK `*agent`) — the target.
6. **Plan** (Conductor `*plan`) — roll up to the program migration plan.
7. **Validate** (Conductor `*validate`) — shadow-run parity + eval before cutover.

Quick single-playbook pass? Use **Solo** (`tc-convert`) end-to-end.

## Importing into your BMAD install

This is a standard BMAD expansion-pack layout. Drop the whole `bmad-tc-modernization/` folder into your BMAD project's `expansion-packs/` directory (or wherever your install resolves expansion packs), then run BMAD's install/build step so the agents, team, tasks, templates, checklists, and data are bundled and the `*tcmod` slash prefix is registered. Each agent file is self-contained — the YAML block carries the full persona and its dependency map (`{root}/{type}/{name}`).

If your install uses a different layout, the files are portable: agent personas are in `agents/`, shared skills in `data/`/`tasks/`/`templates/`/`checklists/`, and nothing references absolute paths internally.

## Guardrails baked in

- Agents hold no credentials — a TC client service does.
- Approval gates (Kestra `Pause`) before any destructive/outbound action.
- ADK callbacks enforce policy (block unapproved writes, redact PII, allow-lists).
- Eval-gated promotion — no agent ships without passing golden-set thresholds.
- Shadow-run parity required before cutover; decommission only after sign-off.
- Internal: all analysis is local; no playbook or source is exfiltrated.
