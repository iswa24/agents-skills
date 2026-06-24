# Inventory TC Playbook

## Purpose
Ingest and inventory a single ThreatConnect (TC) playbook export (XML/JSON) plus any
custom-app source so downstream tasks have a complete, structured picture of what the
playbook does. Everything in this task runs **internal/local** — exports, custom-app
code, and any extracted secrets never leave the organization boundary. The end product is a
filled `playbook-analysis-tmpl.yaml` for exactly one playbook.

## Elicitation

[[LLM: If the user has NOT given you the export path/location, STOP and ask. Do not
guess or fabricate a path.]]

**elicit: true** — Ask the user:
1. The absolute path (or internal artifact location) of the playbook export (XML/JSON).
2. Whether any custom TcEx app source is checked out locally, and where.
3. Confirmation that you may read these files locally (they must stay internal).

Wait for the response before proceeding.

## SEQUENTIAL Task Execution

1. **Locate & parse the export.** Open the export at the path provided. Detect format
   (TC exports as XML or JSON). Parse into a working structure. Record the playbook
   name, ID, and TC environment/owner it came from.

2. **Identify the trigger type.** Determine how the playbook is invoked: UserAction,
   WebHook/HTTP Link, Timer/Scheduled, TC Data (indicator/group create or update),
   Mailbox, or a Component invocation. Record trigger config (filters, schedules,
   webhook auth) verbatim.
   [[LLM: The trigger drives the Kestra trigger choice later (archetype + design). Be
   precise about TC Data triggers — capture the object filter and the change event.]]

3. **Enumerate every app/node.** Walk the playbook graph. For each app/node record:
   name, app type/category (e.g., Iterator, Set Variable, HTTP Client, TC-write app,
   third-party integration app, Logic/Filter, custom app), and its configured inputs.
   Reference `tc-playbook-kb.md` for the standard app catalog and category mapping.

4. **Map all playbook variables and their types.** List every variable (TC types:
   String, StringArray, Binary, BinaryArray, TCEntity, TCEntityArray, KeyValue,
   KeyValueArray, plus org/system variables). Record where each is produced and
   consumed.

5. **List Components used.** Identify every TC Component the playbook calls. For each,
   note it must be inventoried separately (a Component is itself a sub-playbook).

6. **List external integrations.** Enumerate all outbound integrations (enrichment
   APIs, ticketing/case systems, mail, SIEM, sandboxes, etc.) with endpoints and the
   auth mechanism each uses.

7. **Flag custom TcEx apps for code review.** Any node that is a custom/private app
   (not a stock TC app) MUST be flagged. Record app name, language, and whether source
   is available locally. [[LLM: Custom apps hide business logic and TcEx coupling — do
   NOT attempt to infer their behavior from the node config alone. Flag them with a
   "needs code review" marker so `decompose-logic.md` extracts the real logic.]]

8. **Build a node→edge dependency graph.** Capture every directed edge (node → node),
   including conditional/branch edges and Iterator fan-out edges. Represent as an
   adjacency list. Note any cycles, parallel branches, and join points.

9. **Record TC writes.** Enumerate every write-back to TC: indicators created/updated,
   groups, tags, attributes, associations, security labels, victim assets, and
   task/case writes. These are the deterministic side-effects that shadow-run parity
   will later diff exactly.

10. **Fill the analysis template.** Populate `playbook-analysis-tmpl.yaml` with all of
    the above (metadata, trigger, nodes, variables, components, integrations, custom-app
    flags, dependency graph, TC writes). Leave archetype/decomposition/design sections
    for later tasks. Save the filled doc next to the export, internal.

## Output
- A completed `playbook-analysis-tmpl.yaml` (inventory sections) for this one playbook.
- An explicit list of custom apps flagged for code review.
- The node→edge dependency graph and the full TC-write inventory.

## Done When
- Every node, variable, component, integration, and TC write is recorded.
- All custom apps are flagged.
- The dependency graph has no unresolved/dangling nodes.
