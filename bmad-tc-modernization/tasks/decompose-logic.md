# Decompose Logic (Deterministic vs Agentic)

## Purpose
Decompose each node of an inventoried, classified playbook into **deterministic** vs
**agentic** work, and extract the real business logic from any custom TcEx apps while
stripping TcEx coupling. The output is a per-node decomposition table that tells
`design-target.md` exactly what to build and where.

## Inputs
- Filled `playbook-analysis-tmpl.yaml` (inventory + archetype).
- Custom-app source for any node flagged "needs code review" by `inventory-playbook.md`.

## SEQUENTIAL Task Execution

1. **Confirm prerequisites.** Ensure inventory and archetype are present, and that all
   flagged custom-app sources are available locally. If custom source is missing, STOP
   (see Elicitation).

2. **Apply the scoring rubric per node.** For every node, answer:
   - Does it **need judgment**? (decision not reducible to fixed rules)
   - Does it require **natural language** processing/generation?
   - Is its input **ambiguous/unstructured**?
   - Is there a **deterministic alternative** that yields the same result reliably?
   Reference `agentic-decision-kb.md` for the rubric and tie-breakers.
   [[LLM: The deterministic-alternative question is the decisive one. If a reliable
   deterministic path exists, take it — even if the original TC app or a human did it
   "by feel." Agents cost tokens, add latency, and introduce nondeterminism; spend that
   budget ONLY where judgment is irreducible.]]

3. **Assign a target to each node.** Map every node to exactly one target:
   - **Kestra task** — deterministic control flow, transforms, fan-out, scheduling.
   - **ADK FunctionTool** — a deterministic capability an agent calls (lookup, fetch,
     compute) — still deterministic code, just callable by an agent.
   - **ADK agent** — irreducible judgment/NL/ambiguity (use a shared library agent).
   - **shared TC client** — any read/write to TC (indicators, groups, tags, batch).
   [[LLM: Routine TC reads/writes go through the shared TC v3 REST/Batch client as
   deterministic calls — NOT through an agent. An agent may *decide* to write, but the
   write itself is a deterministic TC-client call or FunctionTool.]]

4. **Extract custom-app business logic.** For each flagged custom TcEx app, read the
   source and write down what it actually does in plain terms: inputs, transforms,
   decisions, outputs, and TC side-effects. Separate genuine business logic from
   framework plumbing.

5. **Identify TcEx coupling to strip.** Mark every place the app depends on TcEx
   (e.g., `tcex.playbook` variable I/O, `tcex.batch`, `tcex.results_tc`, app
   args/install.json wiring, TcEx logging/session/token handling). Reference
   `tcex-coupling-kb.md` for the coupling catalog and clean-replacement mapping. Note
   that ALL of this is removed — we fully exit TcEx and re-implement only a clean TC v3
   REST/Batch client.
   [[LLM: Preserve the business logic; discard the TcEx scaffolding. Every TcEx
   playbook-variable read/write becomes an explicit input/output in the new
   Kestra/ADK data model; every `tcex.batch` becomes a shared-TC-client batch call.]]

6. **Produce the per-node decomposition table.** Emit a table with columns:
   `node | category | judgment? | NL? | ambiguous? | det-alternative? | target | reused
   agent/tool | TcEx coupling to strip | notes`. Include one row per node, plus rows for
   each extracted custom-app logic unit.

7. **Record into the analysis doc.** Write the decomposition table and the custom-app
   logic extractions back into `playbook-analysis-tmpl.yaml`.

## Elicitation

[[LLM: Stop if you cannot determine a node's true behavior — never invent business
logic.]]
**elicit: true (conditional)** — If a custom app's source is missing, obfuscated, or its
intent is unclear, STOP and ask the user for the source or a behavioral description
before assigning a target.

## Output
- Per-node decomposition table (deterministic vs agentic, target assigned).
- Custom-app business-logic extractions with TcEx coupling explicitly marked for removal.

## Done When
- Every node has a single target and a rubric score.
- All flagged custom apps have logic extracted and coupling cataloged.
