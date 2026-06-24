# Deployment & Model-Selection KB — local Kestra, remote models, internal ADK wrapper

Fixed constraints for this program:

- **Kestra runs locally / on-prem.** It is self-hosted, not a managed cloud service.
- **No local model hosting.** Models are accessed **remotely** (Azure OpenAI + AWS Bedrock, and
  more). Only the model API call egresses; everything else stays internal.
- **Model access is via the internal SDK wrapper on top of Google ADK.** Agents request a
  model **by name/tier through the wrapper** — they do NOT hand-configure `LiteLlm`, credentials,
  endpoints, or a gateway. That plumbing is owned by the wrapper and is left as-is.
- Google ADK is therefore used framework-only; the wrapper sits exactly where `LiteLlm` would.

So this KB answers the one question that's actually open: **which model to request for which agent.**

## 1. Topology (what runs where)

| Component | Where it runs |
|---|---|
| Kestra (executor + workers) | **Local / on-prem** |
| ADK agent runner (the reasoning services Kestra calls over HTTP) | **Co-located on-prem** with Kestra — keep the Kestra→ADK hop local |
| Models | **Remote** (Azure OpenAI + Bedrock + others), reached through the internal ADK wrapper |
| Session/state store | **Local Postgres** (`DatabaseSessionService`) — not a cloud-managed DB |
| Artifacts (`Binary`/file content, reports) | **Local object store / filesystem** (e.g. on-prem S3-compatible/MinIO) |
| Secrets | Kestra secrets + the internal on-prem secret store |

Only outbound model calls leave the box. Playbooks, indicators, and custom-app source never leave
the organization; minimize what goes into a prompt and redact with `before_model_callback`.

## 2. How agents pick a model (through the wrapper)

Agents stay model-agnostic in code: they ask the wrapper for a **role tier** (e.g. `triage`,
`reasoning`, `writer`) and the wrapper resolves it to an approved deployment. Pin the tier→model
mapping in one config file so a swap is a one-line change and never touches agent logic. Treat the
mapping below as the **recommended default**, then validate each tier against the eval golden set
before promotion (a model swap = a re-eval).

## 3. Recommended models by agent role

You have broad access; the discipline is **don't use a frontier model where a small one suffices**.
Default family is **Claude on Bedrock** for security/threat reasoning — careful, conservative,
excellent instruction-following and tool/function calling, low hallucination on security calls.
Azure OpenAI is the secondary/fallback and useful where quota or a specific capability favors it.

| Agent role (volume) | Recommended primary | Why | Solid alternative |
|---|---|---|---|
| **Triage / disposition** (highest volume) | **Claude Haiku 4.5** (Bedrock) | fast, cheap, strong structured output + instruction-following; ~80% of all calls live here | GPT-4o-mini or o4-mini (Azure) |
| **Enrichment orchestrator** (tool use / routing) | **Claude Sonnet 4.x** (Bedrock) | best-in-class tool/function calling and reliable JSON | GPT-4o (Azure) |
| **Correlation / investigation** (hardest reasoning) | **Claude Opus 4.x** (Bedrock) with extended thinking | deep multi-step reasoning, pivoting, campaign clustering | o3 (Azure reasoning) |
| **Narrative / report writer** | **Claude Sonnet 4.x** (Bedrock) | analyst-grade writing at moderate cost | GPT-4o (Azure) |
| **Dedup-judge / fuzzy match** | **Claude Haiku 4.5** | lightweight judgment, cheap | GPT-4o-mini |
| **Structured extraction** (`output_schema`) | **Claude Sonnet 4.x** or **GPT-4o** | reliable schema adherence | — |
| **Embeddings** (similarity / dedup / RAG over TI) | **Titan Text Embeddings v2** (Bedrock) or **text-embedding-3-large** (Azure) | vector matching for correlation and near-dup detection | — |

### Selection principles
- **Tier to the work.** Triage and dedup are high-volume and low-judgment → Haiku-class. Reserve
  Opus/o3 strictly for archetype-C investigation; it is the minority of playbooks.
- **Extended thinking / reasoning mode only for correlation**, never for triage — the latency and
  cost don't pay off on a yes/no disposition.
- **One family by default** (Claude on Bedrock) keeps prompt behavior, tool-calling, and output
  formatting consistent across agents; reach for Azure models per-role only when there's a reason.
- **Cost math:** archetype-A playbooks call no model at all (zero token cost). Of the rest, the bulk
  is triage on a small model — keep it there and the program stays cheap.
- **Governance:** only approved deployments are allowed; pin the approved set and have an
  allow-list callback reject anything off-list. Re-run the golden-set eval whenever a tier's model
  changes.

## 4. Networking & data-egress (bank-grade)

- The only outbound path is the model API call, made by the internal ADK wrapper over the organization's
  approved private/egress route. No model runs locally; no other data egresses.
- `before_model_callback` redacts PII/secrets from prompts; an allow-list callback enforces the
  approved-model set.
- Durable case state lives in ThreatConnect Cases (via the TC client), not in ADK memory.

## 5. Decisions still open (confirm before the pilot)
1. The exact **approved deployment names/IDs** the wrapper exposes for each tier (triage / reasoning / writer / embeddings).
2. Whether **Opus-class** is approved for the correlation tier, or correlation must fall back to Sonnet + extended thinking.
3. On-prem **Postgres** and **object-store** endpoints for the ADK session/artifact services.
