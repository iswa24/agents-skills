# Design Target Architecture (One Playbook)

## Purpose
Produce the concrete target design for one playbook: the **Kestra flow** (deterministic
control plane) plus any **ADK agents/tools** (reasoning plane, only where the
decomposition assigned judgment). Prefer reusing the shared agent library over creating
new agents. Output is a filled `conversion-design-tmpl.yaml`, validated against the
`conversion-quality-checklist.md`.

## Inputs
- Filled `playbook-analysis-tmpl.yaml` (inventory + archetype + decomposition table).

## SEQUENTIAL Task Execution

1. **Design the Kestra flow.** From the decomposition table, lay out the deterministic
   spine. Reference `kestra-patterns-kb.md`. Specify:
   - **Trigger** — map the TC trigger (UserAction/WebHook/Timer/TC-Data/Mailbox) to the
     Kestra trigger (webhook, schedule, polling, flow trigger).
   - **DAG** — tasks and their dependency edges from the node→edge graph.
   - **Control flow** — conditions/branches, Switch, sequential vs parallel.
   - **Fan-out** — map TC Iterators to Kestra `EachParallel`/`EachSequential`.
   - **Approval gates** — for archetype D, place `Pause`/manual-approval tasks.
   - **Error handling** — retries, timeouts, `errors`/fail branches, idempotency for TC
     writes.
   [[LLM: The Kestra flow is the source of truth for ordering and side-effects. Agents
   are *called from* the flow (or run async and return); they do not own control flow.
   Keep all TC writes and branching deterministic and in Kestra.]]

2. **Design / select ADK agents.** For each node the decomposition assigned to "ADK
   agent," FIRST try to reuse a shared-library agent (~6–10 agents serve all 60
   playbooks). Reference `adk-patterns-kb.md`. For each agent specify: role, model tier,
   instruction/prompt outline, input contract, output schema, and which **FunctionTools**
   it may call. Only define a NEW agent if no shared agent fits — justify the gap.

3. **Design the ADK tools.** For each node assigned "ADK FunctionTool," define the tool
   signature (typed inputs/outputs), the deterministic implementation it wraps, and
   error semantics. Enrichment tools should come from the shared enrichment tool lib.

4. **Define shared data models and TC client calls.** Specify the typed data models that
   cross the Kestra↔ADK boundary (replacing TcEx playbook variables). Enumerate every TC
   read/write as an explicit **shared TC v3 REST/Batch client** call — no TcEx.

5. **Specify guardrails.** Define: ADK callbacks (before/after model & tool callbacks
   for validation, redaction, cost caps); approval gates where the agent's action is
   high-impact; least-privilege scoping of the TC client token per playbook; and output
   validation against the agent's declared schema.
   [[LLM: Treat every agent action that writes to TC or touches a case as requiring an
   explicit guardrail — either a deterministic validation callback or a human approval
   gate. Default to deny; agents propose, deterministic code disposes.]]

6. **Choose the integration pattern.** Decide per agent how Kestra invokes ADK:
   **ADK endpoint** (deployed service Kestra calls via HTTP — preferred for reuse and
   scaling) vs **inline** (agent runs within the task — only for trivial, low-volume
   cases). Justify the choice with the cost tier and reuse from the analysis.

7. **Fill the design template.** Populate `conversion-design-tmpl.yaml`: Kestra flow,
   agents, tools, data models, TC client calls, guardrails, and integration pattern.

8. **Run the quality checklist.** Execute every item in `conversion-quality-checklist.md`
   against the design. Record pass/fail per item; resolve any failures before marking
   the design complete.

## Elicitation

[[LLM: Stop only for genuine design forks the user owns.]]
**elicit: true (conditional)** — If the design requires a NEW shared agent (a gap in the
library) or a high-impact guardrail/approval-gate policy decision, STOP and present the
options to the user for sign-off before finalizing.

## Output
- Filled `conversion-design-tmpl.yaml` (Kestra flow + ADK agents/tools + data models +
  guardrails + integration pattern).
- Completed `conversion-quality-checklist.md` results.

## Done When
- Every decomposition target has a concrete design element.
- Reuse of the shared agent library is maximized and any new agent is justified.
- The quality checklist passes.
