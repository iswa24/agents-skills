# Shadow-Run Validation

## Purpose
Validate a converted playbook against the original **before cutover** by running both in
parallel (shadow mode) on identical triggers/inputs. Deterministic outputs must match
exactly; agent outputs must meet disposition-quality thresholds against a golden set.
Cutover is gated on parity + eval thresholds + the `conversion-quality-checklist.md`.

## Inputs
- The original TC playbook (still live).
- The converted Kestra flow + any ADK agents/tools (from `design-target.md`).
- The golden set / acceptance thresholds for this playbook's agent decisions.

## SEQUENTIAL Task Execution

1. **Stand up shadow mode.** Configure the new Kestra+ADK stack to receive the SAME
   triggers/inputs as the live TC playbook, but with side-effects routed to a shadow
   sink (no production TC writes, no real tickets/notifications). The old TC playbook
   continues to own production effects during shadow-run.
   [[LLM: Shadow-run must NOT double-write to production. Intercept the new stack's TC
   writes and external integration calls and divert them to a sink so you can diff
   intended effects without causing real-world duplicates.]]

2. **Capture outputs from both.** For each trigger event, record from BOTH systems: the
   full intended TC writes (indicators/groups/tags/attributes/associations), external
   integration payloads, and any agent dispositions/rationale. Tag each pair with a
   correlation ID.

3. **Diff deterministic outputs for exact parity.** Compare the deterministic
   side-effects (TC writes, transforms, control-flow outcomes) field-by-field. These
   must match EXACTLY. Normalize only legitimately non-deterministic fields (timestamps,
   run IDs, ordering where order is semantically irrelevant) before diffing — and
   document each normalization.
   [[LLM: A deterministic mismatch is a conversion bug, never "close enough." Treat any
   non-normalized diff as a blocker and route it back to `decompose-logic.md` /
   `design-target.md` for correction.]]

4. **Evaluate agent outputs against the golden set.** For each agent decision, score the
   disposition quality against the golden set / acceptance thresholds (e.g., agreement
   rate, false-positive/false-negative rate on triage, rationale validity). Agent output
   is judged on meeting the threshold — not on byte-for-byte equality with the old
   system.
   [[LLM: Agent outputs are probabilistic; do NOT demand exact parity for them. Hold them
   to the eval thresholds. If the new agent decision differs from the old playbook but is
   *correct* per the golden set, that is a pass — and may indicate the old logic was
   wrong.]]

5. **Log discrepancies.** Record every deterministic mismatch and every agent decision
   below threshold, with correlation IDs, inputs, both outputs, and a triage note
   (conversion bug vs. golden-set gap vs. acceptable variance).

6. **Gate the cutover.** Cutover is permitted ONLY when ALL hold:
   - Deterministic outputs reach the required parity rate (target: exact on all
     non-normalized fields across the shadow window).
   - Agent eval scores meet or exceed acceptance thresholds.
   - `conversion-quality-checklist.md` passes in full.
   - The shadow window covered a representative volume/variety of triggers.

7. **Decommission only after sign-off.** Once gates pass, obtain explicit sign-off
   (see Elicitation), cut traffic to the new stack, monitor a burn-in window, then retire
   the old TC playbook. Do not delete the TC export — archive it internal for rollback.

## Elicitation

[[LLM: Cutover and decommission are irreversible-leaning, high-impact actions — never
self-authorize them.]]
**elicit: true** — STOP and present the user the shadow-run report (parity rate, agent
eval scores, discrepancy log, checklist results) and explicitly request sign-off BEFORE
cutover, and again BEFORE decommissioning the old TC playbook.

## Output
- Shadow-run report: deterministic parity results, agent eval scores, discrepancy log.
- A go/no-go cutover decision gated on parity + thresholds + checklist.
- Decommission record (post sign-off) with the TC export archived for rollback.

## Done When
- Deterministic parity and agent eval thresholds are both met.
- `conversion-quality-checklist.md` passes and the user has signed off.
