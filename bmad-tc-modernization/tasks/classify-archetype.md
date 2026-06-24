# Classify Playbook Archetype

## Purpose
Classify an inventoried playbook on TWO independent axes that together determine its
target shape and its migration order:
1. **Archetype (A/B/C/D)** — how much agentic work it needs. Core rule: **deterministic
   stays deterministic; agents are reserved for judgment, NL, and ambiguity** — and most
   deterministic "logic" is data transformation that belongs in **dbt**, not the orchestrator.
2. **TC-coupling (move-now vs blocked)** — whether it can leave ThreatConnect now or
   depends on the not-yet-built serving/system-of-record layer.
This task consumes the inventory from `inventory-playbook.md` and writes both back into
the analysis doc.

## Archetype Definitions
- **A — Pure deterministic ETL.** No judgment; fixed rules and transforms.
  Target: **Kestra only.** No agents.
- **B — Enrich + decide.** Deterministic enrichment with a single bounded judgment
  (triage/disposition). Target: **Kestra + one triage agent.**
- **C — Multi-step investigation.** Open-ended, multi-hop reasoning over ambiguous
  evidence. Target: **ADK multi-agent + Kestra** orchestration.
- **D — Human-workflow / cases.** Case management with human approval/handoff.
  Target: **Kestra + Pause/approval gates + a light assist agent.**

## SEQUENTIAL Task Execution

1. **Review the inventory.** Load the filled `playbook-analysis-tmpl.yaml`. Confirm the
   trigger, node list, integrations, and TC writes are present. If inventory is
   incomplete, return to `inventory-playbook.md` first.

2. **Score every node for judgment / NL / ambiguity.** For each node ask three
   questions and score 0/1 each:
   - Does it require **judgment** (a decision not reducible to fixed rules)?
   - Does it process or produce **natural language** (free text in/out)?
   - Does it handle **ambiguity** (unstructured, conflicting, or open-ended input)?
   Reference `agentic-decision-kb.md` for the scoring rubric and worked examples.
   [[LLM: Be skeptical of "looks like judgment" nodes. A threshold compare, a lookup,
   a regex, or a rules table is DETERMINISTIC even if a human used to eyeball it. Score
   1 only when there is genuinely no deterministic alternative.]]

3. **Tally the scores.** Sum across nodes. Note how many nodes scored any 1, and on
   which dimension. Record whether the judgment is single/bounded vs. multi-hop/open.

4. **Assign the archetype.** Apply the mapping:
   - All-zero tally → **A**.
   - Exactly one bounded judgment/triage node → **B**.
   - Multiple judgment nodes or multi-hop investigation → **C**.
   - Presence of human approval/handoff/case lifecycle → **D** (combine with B/C scoring
     for the agent weight).
   [[LLM: When a playbook straddles two archetypes, pick the higher-agency one ONLY if
   the judgment is real; otherwise prefer the lower-agency archetype to keep cost and
   risk down.]]

5. **Classify TC-coupling (move-now vs blocked).** Independent of archetype, decide
   whether this playbook can leave ThreatConnect now:
   - **Lake-decoupled (move now)** — its inputs are already in the Data Lake (or
     producible via dbt) AND its outputs go to the lake or external systems (SIEM/SOAR/
     ticketing/notify). Nothing depends on TC-internal state.
   - **TC-coupled (blocked)** — it triggers on TC events, reads/writes TC's curated
     indicator/group graph, or calls TC-internal apps. It cannot fully leave until the
     replacement serving / system-of-record layer exists.
   [[LLM: Use the inventory's trigger, data sources, and TC-writes. The decisive question
   is: are this playbook's inputs in the lake, and do its outputs avoid TC-internal state?
   When unsure, mark TC-coupled and name the exact dependency.]]
   Record the verdict and, for blocked playbooks, the specific TC dependency that blocks it.

6. **Estimate reusable-agent applicability.** From the shared agent library (~6–10
   agents serving all 60 playbooks), list which reusable agents this playbook would use
   (e.g., triage/disposition agent, enrichment-summarizer, investigation planner,
   case-assist). Do NOT propose a bespoke agent unless no shared agent fits — note the
   gap if so.

7. **Estimate the token-cost tier.** Assign Low / Medium / High based on agent count,
   reasoning depth, and expected invocation volume from the trigger. Archetype A = none.

8. **Write rationale into the analysis doc.** Record the archetype, the TC-coupling
   verdict (and any blocking dependency), the per-dimension tally, the chosen reusable
   agents, the cost tier, and a 2–4 line rationale that cites the specific nodes that
   drove the decision.

## Elicitation

[[LLM: Only stop for the user if the archetype is genuinely ambiguous after scoring —
e.g., a borderline B-vs-C with real multi-hop judgment, or a hidden human step.]]
**elicit: true (conditional)** — If borderline, present the tally and your two candidate
archetypes and ask the user to confirm before writing.

## Output
- Archetype A/B/C/D + rationale recorded in `playbook-analysis-tmpl.yaml`.
- TC-coupling verdict (lake-decoupled = move-now, or TC-coupled = blocked + the
  blocking dependency).
- List of applicable reusable agents and the token-cost tier.

## Done When
- Every node has a judgment/NL/ambiguity score.
- A single archetype is assigned with cited rationale.
- The playbook is marked move-now or blocked, with the dependency named if blocked.
