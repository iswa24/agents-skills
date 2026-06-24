# ThreatConnect Playbook Reference KB

Reference for an analyst agent decoding a ThreatConnect (TC) playbook export prior to
re-platforming onto the **two-plane architecture**:

- **Kestra = control plane (deterministic):** triggers, scheduling, the DAG, fan-out/iteration,
  retries/backoff, rate-limiting, human-approval gates, audit, secrets, backfills.
- **Google ADK = reasoning plane (probabilistic):** agents that triage, correlate, summarize,
  decide disposition, draft reports.

Most playbook nodes are deterministic plumbing and become Kestra tasks or non-LLM ADK
`FunctionTool`s. Agents are reserved for ambiguity, natural language, and judgment. You fully
exit TcEx; you re-implement only a clean **TC v3 REST + Batch** client.

---

## 1. Triggers and event semantics

| TC Trigger | Fires on / semantics | Kestra mapping |
|---|---|---|
| **TC Event** | Indicator/Group **created/updated/deleted** in an owner; payload = the changed entity. | Polling `Schedule` against TC v3 `/v3/indicators?tql=...` filtered by `dateAdded`/`lastModified`, OR a TC outbound webhook → `io.kestra.plugin.core.trigger.Webhook`. |
| **HTTP/Webhook** | Inbound HTTP POST; body becomes playbook input. | `io.kestra.plugin.core.trigger.Webhook` (key in URL). |
| **Timer/Cron** | Schedule (interval or cron). | `io.kestra.plugin.core.trigger.Schedule` (cron expression). |
| **Mailbox** | New email to a TC mailbox address; parses headers/body/attachments. | `Schedule` polling IMAP/Graph, or a mail→webhook gateway → `Webhook`. NL parsing of the body → ADK agent. |
| **UserAction** | Analyst clicks a custom action in TC UI on a selected entity. | `Webhook` (UI action posts), or TC Cases API + Kestra `Pause` for the human step. |
| **PlaybookComponent** | Invoked as a sub-playbook by a parent. | `io.kestra.plugin.core.flow.Subflow`. |
| **TaskComplete** | A TC Case **task** transitions to complete. | `Schedule` polling Cases API, or webhook → resume a paused flow. |
| **DataStore** | Change in TC DataStore (KV store). | Replace DataStore with Kestra KV store / external DB; poll via `Schedule`. |

> Event triggers deliver the entity that changed. Webhook/Mailbox deliver raw payloads that
> usually need a deterministic normalize step before any agent sees them.

---

## 2. App categories

**Standard / built-in apps** — TC-provided utilities (Create Indicator, Get Group, Copy
Playbook to Case, Send Email). Mostly deterministic → Kestra task calling a TC REST tool.

**Logic apps** (control flow, never become agents):

| App | Purpose | Target |
|---|---|---|
| Iterator | loop over an array | Kestra `EachSequential`/`EachParallel` |
| Logic / Decision | branch on condition | Kestra `If`/`Switch` |
| Set Variable | assign/transform a value | task `outputs` / Pebble vars |
| Merge | combine branch outputs | Kestra join / variable assembly |
| Sleep | delay | Kestra `Pause` with delay |
| Fail / Stop | terminate path | Kestra `Fail` / `errors` block |

**Transform apps** (pure, deterministic → ADK `FunctionTool` or inline Python):
Regex, JSON Path / JMESPath, Find & Replace, Build TCEntity, String ops (split/join/case/format).

**Enrichment / integration apps** (deterministic API calls → `FunctionTool` per provider):
VirusTotal, Shodan, URLhaus, Splunk, ServiceNow, MS Graph. The *call* is deterministic; only the
*interpretation* of results ("is this malicious?") is agent work.

**Custom TcEx apps** — Python (or older Java) apps. Extract pure business logic, drop the TcEx
harness (see `tcex-coupling-kb.md`).

---

## 3. Playbook variables and typing

Variable format: `#App:<ID>:<name>!<Type>` — e.g. `#App:1234:malicious_score!String`.

| TC Type | Meaning | Pydantic / output mapping |
|---|---|---|
| String | scalar text | `str` |
| StringArray | list of strings | `list[str]` |
| Binary | raw bytes (file content) | `bytes` → Kestra/ADK **artifact** |
| BinaryArray | list of byte blobs | `list[bytes]` → artifacts |
| KeyValue | `{key, value}` | `dict` / `KeyValue` model |
| KeyValueArray | list of KeyValue | `list[KeyValue]` |
| TCEntity | one TC object | `TCEntity` model |
| TCEntityArray | list of TC objects | `list[TCEntity]` |
| TCEnhancedEntity | TCEntity + attributes/tags/associations | `TCEnhancedEntity` model |

**TCEntity structure** (shared Pydantic model):

```python
class TCEntity(BaseModel):
    id: int | None = None
    value: str            # e.g. "1.2.3.4" or "Adversary Name"
    type: str             # "Address", "Host", "File", "Adversary", ...
    rating: float | None = None
    confidence: int | None = None
    ownerName: str | None = None
    webLink: str | None = None
```

`Binary`/`BinaryArray` carry actual **file content** — map to ADK `ArtifactService` artifacts and
Kestra file outputs, never inline into LLM context.

**Mapping:** each TC variable → a field on a shared Pydantic model and a Kestra task `output`.
Downstream tasks reference `{{ outputs.<task>.vars.<name> }}`. Agents receive the typed model, not
the raw `#App:...!Type` strings.

---

## 4. Components (reusable sub-playbooks)

A TC **Component** is a reusable sub-playbook with declared inputs/outputs. Two targets:

- **Deterministic component** → Kestra **Subflow** (`io.kestra.plugin.core.flow.Subflow`).
- **Judgment component** (e.g. a reusable "triage indicator" sub-playbook) → a **reusable ADK
  agent** invoked from the parent flow.

Components are the primary reuse seam: a handful of components typically back dozens of playbooks,
mirroring the "~6-10 reusable agents serve all 60 playbooks" principle.

---

## 5. TC data model & writes

**Indicators:** Address, Host, File, URL, EmailAddress, ASN, CIDR, Mutex, Registry Key, User Agent.
**Groups:** Adversary, Campaign, Incident, Event, Threat, Intrusion Set, Report, Document, Signature,
Email, Task.
**Decorations:** Tags, Attributes (typed metadata), Security Labels (TLP/handling), Associations
(indicator↔group, group↔group).

**Writes to TC** = create/update Indicators/Groups and attach Tags/Attributes/Security
Labels/Associations.

**Batch API semantics (v3 `/v3/batch`):**

| Concern | Behavior |
|---|---|
| Dedup | Batch dedupes on `(type, value, owner)`; re-submitting an indicator updates it. |
| Create vs update | Same summary → upsert (rating/confidence/attributes merged per settings). |
| Throughput | Bulk insert thousands of items per job; async job + poll status. |
| Use when | High-volume ingest/enrichment write-back. Single writes use v3 REST endpoints. |

Re-implement batch in your own thin client (`batch_create(items)` → submit + poll). Do **not** rely
on TcEx batch helpers.

---

## 6. Org / Spaces / Workflow / Cases

- **Org / Owner / Spaces** → namespacing in Kestra (namespace per owner) + TC owner scoping on the
  REST client.
- **Workflow / Cases (case management)** → Kestra **workflow** for orchestration + TC **Cases API**
  for the case record. Human steps (assign, review, approve) → Kestra **`Pause`** until a webhook /
  TaskComplete resumes.

Pattern: open Case (REST) → enrich/triage (deterministic + agent) → `Pause` for analyst approval →
write disposition + close Case (REST).

---

## 7. How to read a playbook export

A TC playbook export (XML or JSON) contains:

1. **Trigger node** — type + config (cron, webhook key, event filter, mailbox).
2. **App nodes** — each with `appId`, app name/version, and configured inputs (literal values or
   `#App:<ID>:<name>!Type` references).
3. **Edges** — directed connections defining execution order and pass/fail branches.
4. **Variables** — every `#App:<ID>:<name>!Type` produced/consumed; trace producer→consumer.
5. **Outputs** — terminal writes (Create Indicator/Group, Batch, Send Email, Case update).

**Tracing procedure:**

1. Start at the trigger; note the input payload shape.
2. Follow edges in topological order.
3. For each app: classify it (logic / transform / enrichment / custom / write).
4. Resolve each input variable to its producing node.
5. Mark every node deterministic vs. judgment (see `agentic-decision-kb.md`).
6. Identify terminal TC writes → these become deterministic write tools behind an approval gate.

---

## 8. Mapping summary

| TC construct | Target |
|---|---|
| TC Event trigger | Kestra `Schedule` poll or `Webhook` |
| HTTP/Webhook trigger | Kestra `Webhook` |
| Timer/Cron trigger | Kestra `Schedule` |
| Mailbox trigger | Kestra `Schedule` (mail poll) + ADK parser agent |
| UserAction / Case task | Kestra `Webhook` + `Pause` + TC Cases API |
| PlaybookComponent trigger | Kestra `Subflow` |
| Iterator | Kestra `EachSequential` / `EachParallel` |
| Logic / Decision | Kestra `If` / `Switch` |
| Set Variable / Merge | Kestra task outputs / Pebble vars |
| Sleep | Kestra `Pause` (delay) |
| Fail / Stop | Kestra `Fail` / `errors` block |
| Regex / JSON Path / String / Build TCEntity | ADK `FunctionTool` (no LLM) / inline Python |
| Enrichment app (VT/Shodan/Splunk/...) | ADK `FunctionTool` per provider |
| "Is this malicious / what severity" | ADK triage **agent** |
| "Summarize for analyst" | ADK narrative **agent** |
| Create/Update Indicator/Group, Batch | Deterministic TC REST/Batch tool (behind approval gate) |
| Component | Kestra Subflow (deterministic) or reusable ADK agent (judgment) |
| TC variable typing | Shared Pydantic model + Kestra outputs / ADK `session.state` & artifacts |
| Org / Space | Kestra namespace + TC owner scoping |

**Rule of thumb:** if a node moves or shapes data, it is Kestra/tool. If a node *decides* or *writes
prose*, it is an ADK agent. Deterministic stays deterministic.
