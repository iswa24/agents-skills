# Playbook Readiness Checklist

[[LLM: This checklist confirms a single ThreatConnect (TC) playbook is FULLY understood
before any conversion design begins. Execute it as follows:

1. Work through every item in order. For each, mark `[x]` if satisfied or `[ ]` if not.
2. For every checked item, cite the concrete evidence (export file, node id, variable name,
   source file path, integration name) that justifies the check — do not check on faith.
3. For every unchecked item, note exactly what is missing and who/what is needed to resolve it.
4. INTERNAL / LOCAL-ONLY: this playbook, its export, IOCs, attributes, and custom-app source
   must never leave the organization boundary. Do not transmit any artifact externally to "verify" it.
5. At the END, produce a Gaps Summary: list every unchecked item, the blocker, and the owner.
   A playbook is NOT ready for conversion design until every item is `[x]` or has an explicit,
   accepted waiver recorded in the summary.]]

## Export Completeness
- [ ] Full playbook export bundle obtained (not a screenshot or partial)
- [ ] All referenced Components resolved and their exports obtained
- [ ] All custom / Run-app source code located internal
- [ ] Export version / TC instance recorded for reproducibility

## Trigger Understood
- [ ] Trigger type identified (UserAction / Webhook / HTTPLink / Timer / TC event)
- [ ] Real-world event semantics documented (what actually fires it)
- [ ] Trigger payload / inputs enumerated
- [ ] Equivalent target Kestra trigger identified

## All Nodes Inventoried
- [ ] Every node listed with its node id and app/type
- [ ] Each node categorized (logic / transform / enrichment / custom / CRUD)
- [ ] Each node's purpose documented
- [ ] No "mystery" nodes remain unexplained

## All Variables & Types Mapped
- [ ] Every playbook variable listed
- [ ] Each variable's TC type recorded (String / StringArray / TCEntity / Binary / KeyValue / etc.)
- [ ] Each variable mapped to a target shared-model field
- [ ] TCEntity / Binary typing fidelity flagged where it must be preserved

## External Integrations & Auth Identified
- [ ] Every external system enumerated (VT / Splunk / ServiceNow / mail / internal APIs)
- [ ] Direction of each integration recorded (inbound / outbound)
- [ ] Auth mechanism documented for each
- [ ] Secret storage location identified for each
- [ ] Rate limits / quotas noted

## Custom Apps Reviewed
- [ ] Language identified for each custom app
- [ ] Source located and readable internal
- [ ] TcEx coupling catalogued (every TcEx import, variable read/write, batch helper)
- [ ] Apps requiring full code review flagged
- [ ] Re-implementation cost (TC v3 REST/Batch client) noted per app

## TC Writes Catalogued
- [ ] Every indicator created/updated listed
- [ ] Every group created/updated listed
- [ ] Every tag applied listed
- [ ] Every attribute created/updated listed
- [ ] Every association created/updated listed
- [ ] Operation type (create / update / delete) and TC owner recorded for each

## Sensitivity / Compliance Flags
- [ ] PII / regulated data touchpoints identified
- [ ] Destructive or outbound actions flagged
- [ ] Privileged credentials touched identified
- [ ] Judgment / NL / ambiguity points (potential agent candidates) flagged

## Internal Handling Confirmed
- [ ] All artifacts confirmed to have stayed inside the organization boundary
- [ ] No export, IOC, attribute, or custom-app source was transmitted externally
- [ ] Handling confirmation recorded with name and date

---

[[LLM: Output the Gaps Summary here — unchecked items, blockers, owners, and any accepted
waivers. State the overall readiness verdict: READY or NOT READY for conversion design.]]
