# Review Application & Feed Layer

## Purpose

Reverse-engineer one application-layer / feed-ingestion app from its source code to answer two
questions the playbook layer cannot: **what does this application do with feeds, and what data logic
is built into it?** The output is a feed → parse → normalize → data-logic → write → mirror map, a
catalog of every data rule, both data sinks (ThreatConnect and the Data Lake), and a Data-Lake gap
assessment. Everything runs locally — source never leaves the organization.

[[LLM: Almost all ingestion and data logic is DETERMINISTIC and maps to Kestra tasks, not agents.
Do not invent agentic steps here. The agentic value lives over the consolidated Data Lake, which you
flag separately. Default reference: data/feed-ingestion-kb.md.]]

## SEQUENTIAL Task Execution

1. **Confirm scope and handling.** Elicit from the user:
   - The location of the application/feed app source (local path / internal artifact).
   - Confirmation that this is **the organization's own custom-app / integration / feed-connector
     source** (built on TcEx) — NOT ThreatConnect Inc.'s proprietary platform source. If it's vendor
     platform code, stop and flag the IP/license scope concern.
   - Confirmation you may read it locally. [[LLM: All source, feeds, and analysis stay
     internal/local-only. Never transmit or upload any artifact to "verify" it.]]

2. **Identify the app type.** Read `install.json` / `app.yaml`. Record `runtimeLevel`
   (Organization/Job, ApiService, WebhookTriggerService/TriggerService, Playbook), declared args
   (the config surface), and declared outputs. Locate the **entry point** for that runtimeLevel
   (Job: `run()` / main loop; Service: the request/trigger handler). Reference feed-ingestion-kb.md §1.

3. **Map ingestion.** For each feed the app handles, capture: feed name/source, **protocol**
   (TAXII/STIX, REST poll, webhook, email, file/CSV, DB/queue), endpoint(s), auth, polling cadence,
   and payload format. Reference feed-ingestion-kb.md §3.

4. **Trace parse → normalize → data logic.** Walk the pipeline and isolate the fetch / parse /
   normalize / data-logic / write blocks. **Catalog every data-logic rule** (field mapping,
   indicator typing, confidence/rating/scoring, tagging, attributes, security labels, association
   rules, dedup keys, filtering/suppression/expiry, enrichment calls, batch chunking) — one row each,
   with its inputs and conditions. Reference feed-ingestion-kb.md §4.

5. **Identify both sinks.** Capture every write:
   - **ThreatConnect** (Batch API / v3 bulk): which entities and fields are created/updated.
   - **Data Lake mirror**: find any secondary write (S3/Blob put, Kafka/Kinesis produce,
     JDBC/Snowflake/BigQuery insert, file drop). Extract the **schema, path/topic/table, and mirrored
     fields**. Note whether the lake copy looks complete/lossless or partial/derived.
   Reference feed-ingestion-kb.md §5.

6. **Separate config vs hardcoded.** Record which behavior is config-driven (install.json args / env
   / config file) vs literal in code — this scopes what is parameterizable in Kestra.

7. **Classify deterministic vs judgment.** Apply the rubric in agentic-decision-kb.md to each data
   rule. Expect almost all to be deterministic. Flag the rare genuine judgment point (e.g. fuzzy
   dedup, cross-feed correlation) — and note it is better done over the lake than inside ingestion.

8. **Map to target and strip TcEx.** For each element, assign the target (Kestra trigger / Kestra
   task / ADK tool / TC v3 client) per feed-ingestion-kb.md §8, and list the TcEx-isms to strip per
   tcex-coupling-kb.md. Call out the key unlock: **if the Data Lake already holds this app's output,
   Kestra can read inputs from the lake and this ingestion may not need rebuilding at all.**

9. **Record gaps and unknowns.** List what the source can't confirm (live schemas, real schedule,
   lake completeness, out-of-band steps) and what's needed to close each (sample payloads, runbook,
   data reconciliation). Reference feed-ingestion-kb.md §7.

10. **Fill the template.** Populate `application-analysis-tmpl.yaml`. Save the filled doc next to the
    source, internal/local. [[LLM: Present the data-flow map and the lake-gap assessment back to the
    user before finalizing — these drive the A-vs-B target-state decision.]]
