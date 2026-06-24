# Conversion Quality Checklist

[[LLM: This checklist GATES a target design (or a converted playbook) before it is promoted.
Execute it as follows:

1. Work through every item in order. Mark `[x]` (satisfied) or `[ ]` (not satisfied).
2. For each checked item, cite concrete evidence: the Kestra task id, ADK agent name, tool
   definition, callback, secret reference, model id, eval result, or shadow-run metric that
   proves it. Design assertions without evidence do NOT pass.
3. The governing principle is DETERMINISTIC-FIRST: anything that can run deterministically in
   the Kestra control plane MUST; ADK agents exist only where judgment / NL / ambiguity is
   unavoidable; reuse the ~6-10 shared agents before adding a new one; no TcEx at runtime.
4. For each unchecked item, record the defect and the required fix.
5. At the END, produce a Gaps Summary and an overall verdict. Any unchecked item in
   Deterministic-First, Agent Justification, No TcEx Runtime, or Guardrails is a HARD BLOCK.]]

## Deterministic-First
- [ ] Every node that can be deterministic IS implemented deterministically (Kestra task or TC client)
- [ ] No LLM/agent is doing regex, string parsing, or schema validation
- [ ] No LLM/agent is doing CRUD against TC (writes go through the deterministic TC client)
- [ ] No LLM/agent is doing iteration / fan-out (Kestra EachParallel handles loops)
- [ ] Control flow (if / switch / join) lives in Kestra, not in an agent prompt

## Agent Justification
- [ ] Each agent is justified by a genuine judgment / NL / ambiguity need
- [ ] Reuse-first honored: shared agents used before any new agent is introduced
- [ ] Any new agent has a recorded justification for why a shared agent could not serve
- [ ] Archetype assigned (A/B/C/D) and consistent with the actual agent usage
- [ ] No "agent for agent's sake" — bounded scope and clear stop conditions

## No TcEx Runtime
- [ ] No TcEx package imported anywhere in the converted code
- [ ] Clean TC v3 REST/Batch client used for all TC reads and writes
- [ ] TC typing preserved through the client (TCEntity / Binary not flattened to strings)
- [ ] No leftover TcEx playbook-variable read/write idioms

## Guardrails
- [ ] Approval / Pause gate present before every destructive TC write
- [ ] Approval / Pause gate present before every outbound action (mail / ticket / block)
- [ ] Agents hold NO credentials (secrets injected only into deterministic tasks/tools)
- [ ] Callbacks enforce policy (input/output validation, allow-lists, kill-switches)
- [ ] Secrets externalized (no hardcoded keys; referenced from secret store)

## Data Models
- [ ] Typed shared models used end to end
- [ ] TC typing preserved, including TCEntity and Binary structures
- [ ] Agent output_schema defined for structured returns
- [ ] No lossy String coercion of typed objects

## Error Handling & Retries
- [ ] Kestra retry policy defined with backoff for transient failures
- [ ] Rate-limit handling for external integrations
- [ ] Error tasks / failure paths defined (no silent drops)
- [ ] Idempotency considered for TC writes on retry

## Observability
- [ ] Tracing spans across Kestra → ADK → tools are correlated (single trace id)
- [ ] Agent decisions and tool calls are logged
- [ ] TC writes are auditable (who/what/when)
- [ ] Metrics emitted for parity and eval monitoring

## Eval
- [ ] Golden set defined for every agent in the design
- [ ] Eval thresholds defined and recorded
- [ ] Deterministic parity checks defined against the legacy playbook
- [ ] Promotion is eval-gated (cannot go live below threshold)

## Shadow-run Parity
- [ ] Deterministic outputs match the legacy playbook exactly over the shadow window
- [ ] Agent outputs stay within defined eval thresholds over the shadow window
- [ ] No unexpected or extra TC writes observed
- [ ] Discrepancies triaged and resolved before cutover

## Cost
- [ ] Archetype-A playbooks incur zero token cost (no agent calls)
- [ ] Model tiering applied (cheapest capable model per task; LiteLlm routing where used)
- [ ] Enrichment results cached to avoid redundant agent/tool calls
- [ ] Per-playbook cost estimated and within budget

---

[[LLM: Output the Gaps Summary here — unchecked items, defects, required fixes, and owners.
State the overall verdict: PASS or BLOCK. Remember: any gap in Deterministic-First, Agent
Justification, No TcEx Runtime, or Guardrails is a HARD BLOCK regardless of other passes.]]
