# Produce Program-Level Conversion Plan

## Purpose
Roll the individual per-playbook analyses and designs up into a single **program-level
migration plan** spanning all ~60 playbooks. This task sequences the work, identifies
the shared foundation to build first, defines the reusable agent library, and sets
shadow-run/cutover criteria. Output is a filled `migration-plan-tmpl.yaml`.

## Inputs
- All completed `playbook-analysis-tmpl.yaml` docs (inventory + archetype +
  decomposition).
- Any completed `conversion-design-tmpl.yaml` docs available so far.

## SEQUENTIAL Task Execution

1. **Aggregate archetypes.** Tally how many playbooks fall in A/B/C/D. Record the agent
   cost-tier distribution and the most-used reusable agents across the portfolio.

2. **Batch by archetype / shared dependency.** Group playbooks into conversion batches
   that share an archetype, an integration, or a custom-app logic family — so a batch
   reuses the same agents, tools, and TC-client calls. Smaller, deterministic
   archetype-A playbooks make good early batches; archetype-C the latest.

3. **Sequence the 6 phases.** Lay out the program timeline across:
   1. **Discovery** — inventory/classify/decompose all 60.
   2. **Foundation** — build the shared platform (see step 4).
   3. **Pilot** — convert 1–2 representative playbooks end-to-end to prove the pattern.
   4. **Factory** — convert in batches using the proven pattern and shared library.
   5. **Parallel-run / validate** — shadow-run each conversion (see step 8).
   6. **Cutover / decommission** — switch traffic, retire TC playbooks after sign-off.

4. **Identify the shared foundation to build first.** Specify the platform components
   that every conversion depends on, built in the Foundation phase:
   - **TC data-model library** (typed models replacing TcEx playbook variables).
   - **TC API/Batch client** (clean TC v3 REST/Batch — the only TcEx replacement).
   - **Enrichment tool library** (shared ADK FunctionTools).
   - **Agent scaffolds** (base classes, callbacks, prompt/version conventions).
   - **Eval harness** (golden sets + disposition-quality scoring for agents).
   - **CI/CD + secrets** (deploy Kestra flows + ADK services; internal secret handling).
   - **Observability** (tracing, cost metering, run logs across both planes).
   [[LLM: The Foundation is the highest-leverage work — get the TC v3 client, data
   models, agent scaffolds, and eval harness right BEFORE the Factory phase, or every
   batch re-litigates them. Do not let pilot velocity tempt you into skipping it.]]

5. **Define the reusable agent library.** From the aggregated archetype/agent data,
   name the ~6–10 agents that serve all 60 playbooks (e.g., triage/disposition,
   enrichment-summarizer, investigation-planner, evidence-correlator, case-assist).
   For each: responsibility, the playbooks/archetypes it serves, and its tool surface.
   [[LLM: Hold the line at ~6–10 agents. If analyses imply a 30th bespoke agent,
   reconsider — most "new agent" needs are a new prompt/tool on an existing agent, not a
   new agent. Sixty agents for sixty playbooks is the anti-pattern this program exists
   to avoid.]]

6. **Estimate effort and risk.** Per batch, estimate effort (size/complexity) and rank
   risk (custom-app complexity, integration fragility, agent-judgment criticality,
   trigger volume). Flag high-risk playbooks for extra validation and earlier scheduling
   of their foundation dependencies.

7. **Define shadow-run / cutover criteria.** Set the program-wide gates that each
   playbook must pass to cut over: deterministic-output parity, agent eval thresholds
   against the golden set, and a passing `conversion-quality-checklist.md`. Reference
   `shadow-run-validation.md` for the procedure.

8. **Fill the plan template.** Populate `migration-plan-tmpl.yaml` with the archetype
   aggregate, batches, 6-phase sequence, shared-foundation backlog, reusable agent
   library, effort/risk estimates, and cutover criteria.

## Elicitation

**elicit: true** — Before finalizing, STOP and present the user: (a) the proposed phase
sequence and batch order, and (b) the reusable agent library roster. Get confirmation on
sequencing priorities and any organization-specific constraints (capacity, change windows,
internal hosting) before writing the final plan.

## Output
- Filled `migration-plan-tmpl.yaml`: batches, 6-phase plan, shared foundation, reusable
  agent library, effort/risk, and cutover gates.

## Done When
- All 60 playbooks are batched and sequenced.
- The shared foundation backlog and agent library are defined.
- Cutover criteria are explicit and tied to the shadow-run procedure.
