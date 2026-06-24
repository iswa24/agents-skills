# ThreatConnect Modernization — Feasibility & Approach (Decision Brief)

## Bottom line up front

**Yes — the capability can be rebuilt on Data Lake + dbt + Kestra + agents, and starting with
playbooks is reasonable.** The stack is sound: the data already lands in the lake, dbt is the right
home for the data model and data logic, Kestra replaces playbook orchestration, and agents add the
intelligence ThreatConnect (TC) can't. The work is **not** "make everything an agent" — agentic is a
thin, high-value layer on a deterministic foundation. The real decision isn't technical feasibility;
it's **scope**: are we moving orchestration + intelligence off TC while keeping a system of record,
or building and owning a full platform? Those are very different commitments. Recommended path below
de-risks both and delivers value in the first phase.

## The stack and the division of labor

| Layer | Tool | Responsibility |
|---|---|---|
| Storage / system of record | **Data Lake** | holds everything TC ingests (already in place) |
| Data model + data logic | **dbt** | normalization, the intel data model, dedup, versioning, confidence/rating scoring, indicator typing — *most of a playbook's "logic"*, as tested, versioned SQL |
| Orchestration / integration | **Kestra** | triggers, API/enrichment calls, write-backs, notifications, approval gates; **runs dbt as a step** |
| Reasoning | **ADK agents** | only the judgment: triage, correlate, prioritize, narrative |

**Lane rule:** transform data at rest → dbt; act across systems → Kestra; judge/interpret → agent.
Keep these lanes clean and the architecture is auditable and bank-grade — exactly what agents and
ad-hoc playbook code cannot give us.

## What a rebuilt playbook looks like

Take a typical playbook — *enrich a new indicator, score it, tag if malicious, notify*:
- **dbt** maintains the indicator model with enrichment joins and the computed rating (the scoring
  rules that used to live *inside* the playbook — now centralized, tested, reusable).
- **Kestra** triggers, refreshes dbt, calls external enrichment, routes to a **triage agent** only
  for the malicious/not decision, then writes back and notifies — with an approval gate if the action
  is destructive.

The deterministic ~80% moves to dbt + Kestra; the agent does the ~20% that needs judgment.

## The one layer not in the four tools

Lake + dbt + Kestra + agents give us ingestion, data model, orchestration, and reasoning. They do
**not** yet provide what TC also gives us off the shelf:

- **Analyst serving + UI** — search, pivot, the relationship graph, dashboards. A dbt table is not an
  analyst workbench.
- **Write-back / manual curation** — analysts adding indicators, marking false positives.
- **Cases / workflow, plus RBAC, audit, and data governance** for the intel itself.

None of this blocks a playbooks-first phase, but it is the part that keeps us tied to TC until built.
It must be **named, scoped, and owned now** — not discovered mid-program.

## Playbooks first — done right

Playbooks split on one axis that decides migration order:

- **Lake-decoupled (move now)** — inputs already in the lake, outputs to the lake or external systems.
  These prove the stack and deliver the early win.
- **TC-coupled (blocked)** — trigger on TC events, or read/write TC's curated graph. These cannot
  fully leave until the replacement serving / system-of-record layer exists.

The reverse-engineering effort (playbook + application-layer review) produces this split, so the
deliverable becomes: *"Here are the N playbooks we move now; here are the M blocked on the serving
layer — which is the real platform decision."*

## Recommended roadmap (strangler-fig — value early, TC retired last)

1. **Phase 1 — Intelligence over the lake.** Keep TC as system of record. Stand up dbt models +
   Kestra flows + the agentic layer over the Data Lake. Migrate the lake-decoupled playbooks. *Months,
   not years; delivers the capability TC can't.*
2. **Phase 2 — Replacement system of record.** Build the serving graph/index over the lake; shadow-run
   against TC for parity.
3. **Phase 3 — Analyst experience + cases.** Serving UI, curation/write-back, case workflow, RBAC.
4. **Phase 4 — Cut over and decommission TC.**

## Decisions needed from leadership

1. **Target state:** orchestration + intelligence off TC with a retained system of record, *or* a full
   platform rebuild we own (data model + UI + compliance + maintenance)? — a ~2-quarter modernization
   vs a ~2-year platform program.
2. **Owner** for the serving / UI / system-of-record layer (the part outside the four tools).
3. **Build vs buy** for the analyst experience and system of record.

## Risks & mitigations

| Risk | Mitigation |
|---|---|
| "Make it all agentic" → slow, non-reproducible, unauditable | Lane rule: deterministic in dbt/Kestra; agents scarce, gated, eval-tested |
| Hidden scope (serving/UI/RBAC) surfaces late | Name and own it now; phase it explicitly (Phases 2–3) |
| TC-coupled playbooks stall and look like failure | Classify by coupling up front; move the decoupled set first |
| Big-bang rewrite → years before value | Strangler-fig; Phase 1 ships intelligence over the lake fast |
| Backing into "we became a TIP vendor" unintentionally | Make build-vs-buy an explicit, eyes-open decision |
