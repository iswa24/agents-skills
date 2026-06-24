# Classify Playbook Archetype

## Purpose
Classify an inventoried playbook into one of four archetypes (A/B/C/D) that determine
its target shape. The core rule governs: **deterministic stays deterministic; agents are
reserved for judgment, natural language, and ambiguity.** This task consumes the
inventory from `inventory-playbook.md` and writes the archetype + rationale back into
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

5. **Estimate reusable-agent applicability.** From the shared agent library (~6–10
   agents serving all 60 playbooks), list which reusable agents this playbook would use
   (e.g., triage/disposition agent, enrichment-summarizer, investigation planner,
   case-assist). Do NOT propose a bespoke agent unless no shared agent fits — note the
   gap if so.

6. **Estimate the token-cost tier.** Assign Low / Medium / High based on agent count,
   reasoning depth, and expected invocation volume from the trigger. Archetype A = none.

7. **Write rationale into the analysis doc.** Record the archetype, the per-dimension
   tally, the chosen reusable agents, the cost tier, and a 2–4 line rationale that cites
   the specific nodes that drove the decision.

## Elicitation

[[LLM: Only stop for the user if the archetype is genuinely ambiguous after scoring —
e.g., a borderline B-vs-C with real multi-hop judgment, or a hidden human step.]]
**elicit: true (conditional)** — If borderline, present the tally and your two candidate
archetypes and ask the user to confirm before writing.

## Output
- Archetype A/B/C/D + rationale recorded in `playbook-analysis-tmpl.yaml`.
- List of applicable reusable agents and the token-cost tier.

## Done When
- Every node has a judgment/NL/ambiguity score.
- A single archetype is assigned with cited rationale.
