# ThreatConnect Application & Feed-Ingestion Layer KB

Audience: a code-forensics agent reviewing **application-layer source** — the code that brings
feeds in and applies data logic *before/around* playbooks. This is the layer below the playbooks:
**feed → fetch → parse → normalize → data logic → write → (mirror to Data Lake)**. Reverse-engineer
it from source the same way you reverse-engineer a playbook's flow.

Core doctrine still holds: **almost all ingestion/data logic is deterministic** — it maps to Kestra
tasks, not agents. The agentic value lives over the consolidated Data Lake (correlation, triage,
prioritization), not inside the feed pipeline. Fully exit TcEx; re-implement only a clean TC v3
REST/Batch client.

## 1. ThreatConnect app types (the application layer)

TcEx apps declare a `runtimeLevel` in `install.json` / `app.yaml`. Identify it first — it tells you
where the entry point is and what the app does.

| runtimeLevel | What it is | Where the feed/data logic lives |
|---|---|---|
| **Organization / Job** | Scheduled or on-demand **batch ingestion** — the classic feed connector | `run()` / main loop: fetch → parse → normalize → Batch-write. **This is the primary feed layer.** |
| **ApiService** | Long-running HTTP API the platform calls | request handler / route dispatch |
| **WebhookTriggerService / TriggerService** | Service that fires on inbound webhook/event | the trigger/dispatch handler |
| **Playbook** | App invoked inside a playbook | `run()` — already covered by playbook decomposition |

The apps you most want for "what happens to feeds" are usually **Job apps** (batch ingestion) and
**Service apps** (push/streaming ingestion).

## 2. Anatomy of a feed pipeline (what to reconstruct)

```
source/connector → fetch → parse → normalize → enrich → dedup/score/tag/filter → write → mirror
   (TAXII/REST/      (auth,   (raw→    (→TCEntity, (lookups) (data logic — the rules)  (Batch  (Data
    webhook/email/    poll)    objects)  field map)                                      API)    Lake)
    file/DB)
```

Trace each stage in the source and record it. The middle stages (**normalize + data logic**) are the
"what data logic is built" answer; the last stage (**write/mirror**) is the "where the data sits"
answer.

## 3. Feed protocols and how they look in code

| Protocol | Tells / what to grep for |
|---|---|
| **TAXII 2.x / STIX 2.x** | `taxii2client`, collections, `get_objects`, `poll`, STIX bundle parsing; STIX `indicator`/`malware`/`relationship` → TC indicator/group mapping |
| **REST poll** | scheduled HTTP GET with cursor/since-token, pagination, API key/OAuth |
| **Webhook / push** | a service handler receiving POSTs (Service app) |
| **Email / mailbox** | IMAP/Graph mailbox poll, attachment/body parsing |
| **File / CSV / S3 drop** | file watchers, CSV/JSON parsers, object-store reads |
| **DB / queue pull** | JDBC/Kafka/SQS consumers |

## 4. Data-logic patterns to catalog (the rules)

These are the business rules to extract one by one — each becomes a deterministic Kestra task/ADK
tool in the target:

- **Field mapping / normalization** — raw fields → `TCEntity`/indicator fields; type coercion.
- **Indicator typing** — how raw values are classified (Address/Host/File/URL/EmailAddress…).
- **Confidence / rating / threat scoring** — formulas or lookup tables.
- **Tagging & attribute rules** — what tags/attributes/security labels get attached and when.
- **Association rules** — how indicators link to groups/adversaries/campaigns.
- **Dedup keys** — what makes a record "the same" (exact vs fuzzy — fuzzy is the rare agent candidate).
- **Filtering / suppression** — what gets dropped, allow/deny lists, thresholds, expiry/decay.
- **Enrichment calls** — external API lookups during ingestion.
- **Batch chunking** — Batch API grouping/size, create-vs-update semantics.

## 5. Where the data goes — the two sinks (answers "where does data sit")

1. **ThreatConnect** via the Batch API — `tcex.batch` / v3 bulk: creates/updates indicators, groups,
   tags, attributes, associations. Extract exactly what entities/fields are written.
2. **Data Lake mirror** — look for a *secondary* write: an S3/Blob put, Kafka/Kinesis produce, a
   JDBC/Snowflake/BigQuery insert, or a file drop. **This is gold** — the code reveals the lake's
   **schema, path/topic/table, and which fields are mirrored**. Capture it; it tells you whether
   Kestra can source inputs straight from the lake instead of rebuilding ingestion.

Flag whether the lake copy looks **complete and lossless** vs a partial/derived subset — that's the
single most important question for the A-vs-B target-state decision (and it needs sample
confirmation, see §7).

## 6. Extraction method (per app)

1. Read `install.json` / `app.yaml` → `runtimeLevel`, declared args (config surface), outputs.
2. Locate the **entry point** for that runtimeLevel (Job: `run()`/main loop; Service: handler).
3. Walk the pipeline: isolate **fetch / parse / normalize / data-logic / write / mirror** blocks.
4. Catalog every **data-logic rule** (§4) with its inputs and conditions.
5. Record **both sinks** (§5): TC writes and any Data Lake write (with schema/path).
6. Separate **config vs hardcoded** (install.json args / env / config file vs literals).
7. Classify each block **deterministic vs judgment** (see `agentic-decision-kb.md`) — expect almost
   all deterministic.
8. Map to target: ingestion trigger → Kestra trigger; fetch/parse/normalize/data-logic/write →
   deterministic Kestra tasks (or ADK tools); lake stays the input substrate; agentic only over the
   lake. Note TcEx-isms to strip per `tcex-coupling-kb.md`.

## 7. What source alone will NOT tell you (state it explicitly)

- **Live payloads / true schemas** — code shows logic, not data. Confirm against sample feed
  payloads or Data Lake samples.
- **Real schedule / deployment / volume** — unless in config/IaC; get the ops runbook.
- **Lake completeness** — whether the mirror is lossless vs derived — needs data reconciliation.
- **Out-of-band / manual steps** not represented in code.

Source gets you ~70–80% (all the logic and both sinks); the rest is runtime/sample confirmation.

## 8. Target mapping summary

| Application-layer element | Target |
|---|---|
| Scheduled Job ingestion | Kestra `Schedule` trigger + fetch task |
| Webhook/Service ingestion | Kestra `Webhook` trigger / polling trigger |
| Parse / normalize / field-map | Deterministic Kestra task or ADK `FunctionTool` |
| Scoring / tagging / filtering / dedup (exact) | Deterministic task |
| Dedup (fuzzy) / cross-feed correlation | Rare ADK agent — **and better done over the lake** |
| TC Batch write | Clean TC v3 REST/Batch client (no TcEx) |
| Data Lake mirror | Kestra task to the lake sink — or, if the lake already has it, **skip ingestion and read from the lake** |
