# dbt Patterns for the Intel Data Model KB

Reference for the agentic architect / data engineer deciding **where deterministic logic belongs** when rebuilding a ThreatConnect-style threat-intel platform on a modern stack. Confidentiality note: examples are generic; treat any sample identifiers as local-only / internal to the organization.

## 1. dbt's role and the lane rule

The target stack has four lanes. Each owns a distinct responsibility; do not blur them.

| Layer | Responsibility | Nature |
|-------|----------------|--------|
| **Data Lake** | Storage / system of record; raw landed feeds + modeled outputs | Persistence |
| **dbt** | **Deterministic data model + transformation layer — data at rest** | SQL ELT, batch |
| **Kestra** | Orchestration, integration, reactive action across systems | Procedural, event-driven |
| **ADK agents** | Reasoning, judgment, natural language, ambiguity resolution | Probabilistic |

**THE RULE**

> - Deterministic transformation of **data at rest** → **dbt**
> - Deterministic **action across systems** → **Kestra**
> - **Judgment / natural language / ambiguity** → **agent**

Most of what a ThreatConnect playbook calls "logic" — scoring, indicator typing, tagging rules, dedup, normalization — is **not** orchestration and **not** reasoning. It is **data transformation** and belongs in **dbt**. Putting it in the orchestrator makes it untestable and imperative; putting it in an agent makes deterministic rules non-reproducible. Default to dbt for anything expressible as SQL over tables already in the lake.

## 2. What belongs in dbt

| ThreatConnect concept | dbt realization |
|-----------------------|-----------------|
| Raw feed ingest cleanup | Typed **staging models** (`stg_*`): cast, trim, standardize, rename |
| Indicators / Groups / Relationships / Attributes / Tags / Security Labels | **Marts** — one modeled table per intel entity |
| Batch dedup | **Incremental models** + **surrogate keys**; merge/upsert (`unique_key`) |
| Indicator versioning / history | dbt **snapshots** (SCD type 2) — captures change over time |
| Confidence / rating / threat score | SQL in models or a dedicated **scoring mart** |
| Indicator typing (Address/Host/File/URL/EmailAddress…) | SQL **classification** via regex / pattern rules |
| Tagging / attribute / association rules over data at rest | SQL `case`/join logic in intermediate or mart models |
| Enrichment using data **already in the lake** | Joins in **intermediate** (`int_*`) models |
| Data quality + governance | dbt **tests** (`not_null`, `unique`, `accepted_values`, `relationships`), `docs`, lineage |

Key idea: if the inputs and outputs are both **tables in the lake** and the rule is **deterministic**, it is a dbt model — full stop.

## 3. What does NOT belong in dbt

Route to **Kestra**:
- API calls / external enrichment **fetch** (VirusTotal, WHOIS, GeoIP, etc.)
- Webhooks, event / real-time reactions, queue consumers
- **Write-backs** to external systems (SIEM / SOAR / ticketing / EDR)
- Notifications (email, Slack, PagerDuty)
- Any runtime loop, retry policy, or non-SQL procedural logic

Route to **agents**:
- Judgment and triage / disposition
- Correlation across **ambiguous** signals
- Natural-language summarization or analyst-style write-ups
- Fuzzy / probabilistic matching where rules cannot be fully enumerated

> dbt is **batch ELT**. For low-latency reactive paths ("block this IP now"), stay event-driven in **Kestra** — never try to make dbt do real-time. dbt models the world as it settles; Kestra acts the moment something happens.

## 4. Suggested dbt project layout

Standard four-tier flow: `sources → staging → intermediate → marts`.

```text
models/
├── staging/
│   ├── _sources.yml          # raw feed tables landed in the lake
│   ├── stg_otx_indicators.sql
│   ├── stg_misp_events.sql
│   └── stg_feed_indicators.sql
├── intermediate/
│   ├── int_indicators_deduped.sql     # surrogate key + dedup
│   └── int_indicators_enriched.sql    # joins to lake-resident context
├── marts/
│   ├── indicators.sql
│   ├── groups.sql
│   ├── relationships.sql
│   └── scored_indicators.sql          # computed rating/confidence
snapshots/
└── indicators_snapshot.sql            # SCD2 versioning
tests/
└── ...                                # singular + generic tests
```

- **staging**: 1:1 with a source, light typing/renaming, no business logic.
- **intermediate**: dedup, joins, reusable transforms.
- **marts**: the consumable intel data model (entities + scoring).
- **snapshots**: history/versioning of mutable entities.

## 5. dbt ↔ Kestra integration

Kestra **orchestrates** dbt; dbt **transforms**. Use the Kestra dbt plugin (`io.kestra.plugin.dbt.cli.DbtCLI` for `dbt build` / `run` / `test`, or `DbtCloud` for dbt Cloud jobs).

**Canonical flow:**

```text
Kestra trigger (schedule/event)
  → land/refresh raw feeds in lake
  → run dbt models  (transform: normalize, dedup, type, score)
  → read modeled marts
  → enrich (API) / invoke agent (judgment) / write-back (SIEM/SOAR)
  → notify
```

dbt owns the middle transform; Kestra owns everything around it.

```yaml
id: intel-pipeline
namespace: tc.modernization

tasks:
  - id: dbt_build
    type: io.kestra.plugin.dbt.cli.DbtCLI
    commands:
      - dbt build --target prod   # run + test in one pass
    profiles: |
      intel:
        target: prod
        outputs:
          prod:
            type: duckdb
            path: "{{ vars.lake_path }}"

  - id: enrich_and_writeback
    type: io.kestra.plugin.scripts.python.Script
    # reads dbt marts, calls external APIs / agents, writes back
```

## 6. Illustrative SQL

**6a. Staging — raw → typed indicator (with typing):**

```sql
-- models/staging/stg_feed_indicators.sql
with src as (select * from {{ source('feeds', 'raw_indicators') }})
select
    trim(lower(value))                                as indicator_value,
    case
        when value ~ '^\d{1,3}(\.\d{1,3}){3}$'        then 'Address'
        when value ~ '^[a-f0-9]{32,64}$'              then 'File'
        when value ~ '^https?://'                     then 'URL'
        when value ~ '@'                              then 'EmailAddress'
        else 'Host'
    end                                               as indicator_type,
    cast(first_seen as timestamp)                     as first_seen_at,
    source_name                                       as source
from src
where value is not null
```

**6b. Intermediate — dedup with surrogate key (incremental):**

```sql
-- models/intermediate/int_indicators_deduped.sql
{{ config(materialized='incremental', unique_key='indicator_sk') }}
select
    {{ dbt_utils.generate_surrogate_key(['indicator_value','indicator_type']) }} as indicator_sk,
    indicator_value,
    indicator_type,
    min(first_seen_at) as first_seen_at,
    max(first_seen_at) as last_seen_at,
    count(*)           as observation_count
from {{ ref('stg_feed_indicators') }}
{% if is_incremental() %}
where first_seen_at > (select max(last_seen_at) from {{ this }})
{% endif %}
group by 1, 2, 3
```

**6c. Mart — computed rating/score:**

```sql
-- models/marts/scored_indicators.sql
select
    indicator_sk,
    indicator_value,
    indicator_type,
    observation_count,
    least(100,
        observation_count * 10
        + case when indicator_type = 'File' then 20 else 0 end
        + case when last_seen_at > current_date - 7 then 15 else 0 end
    ) as confidence,
    case when confidence >= 75 then 'High'
         when confidence >= 40 then 'Medium'
         else 'Low' end as threat_rating
from {{ ref('int_indicators_deduped') }}
```

## 7. Mapping: ThreatConnect data-logic element → target

| TC element | Target | Notes |
|------------|--------|-------|
| Normalize raw feed | **dbt** (`stg_*`) | typing, casting, standardization |
| Dedup (exact) | **dbt** | incremental + surrogate key |
| Score / rating / confidence | **dbt** | scoring mart |
| Indicator typing | **dbt** | SQL classification |
| Tagging / attribute rules (over data at rest) | **dbt** | `case`/join logic |
| Snapshot / versioning | **dbt** | snapshots (SCD2) |
| Data quality checks | **dbt** | tests + docs |
| API / external enrichment fetch | **Kestra** | network I/O |
| Event / schedule trigger | **Kestra** | orchestration |
| Write-back (SIEM/SOAR/ticketing) | **Kestra** | cross-system action |
| Notify (email/Slack/page) | **Kestra** | side effect |
| Real-time block / reactive path | **Kestra** | event-driven, low latency |
| Fuzzy dedup / entity resolution | **agent** | better run **over the lake/marts** dbt produced |
| Correlation across ambiguous signals | **agent** | judgment |
| Disposition / triage / NL summary | **agent** | reasoning |

Note for the agent lanes: agents perform best **on top of** the clean, deduped marts dbt produces — let dbt do the deterministic heavy lifting first, then reason over the result.

## 8. Why this matters

Pushing the deterministic data logic into dbt — where it is **tested, versioned in git, centralized, documented, and reproducible** — is what makes the rebuild **auditable and bank-grade**. Every score, type assignment, and dedup decision becomes a reviewable SQL model with lineage and data-quality tests, rather than logic buried in opaque playbook steps or non-deterministic agent prompts. Agents and ad-hoc orchestrator code cannot give you reproducibility or an audit trail; dbt can. Keep the lanes honest and the modernization stays defensible.
