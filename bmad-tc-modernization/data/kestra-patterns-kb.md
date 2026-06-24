# Kestra Orchestration Patterns KB

Audience: a Kestra engineer agent re-platforming ThreatConnect playbooks onto the **control plane**.
Kestra owns everything deterministic: triggers, scheduling, the DAG, fan-out/iteration,
retries/backoff, rate-limiting, human-approval gates, audit, secrets, backfills. ADK agents are
called only for judgment/NL/ambiguity. Deterministic stays deterministic — do not push plumbing
into agents.

---

## 1. Kestra concepts

| Concept | Notes |
|---|---|
| **Flow** | A YAML workflow: `id`, `namespace`, `inputs`, `tasks`, `triggers`, `errors`. |
| **Namespace** | Hierarchical grouping (e.g. `ti.enrichment`); use one per TC owner/domain. |
| **Tasks** | Typed plugin units (`type: io.kestra.plugin...`); the DAG nodes. |
| **Triggers** | Webhook, Schedule (cron), polling, Flow/Subflow. |
| **Inputs / outputs** | Typed flow inputs; each task exposes `outputs` referenced downstream. |
| **Pebble templating** | `{{ }}` expressions: `{{ inputs.x }}`, `{{ outputs.t.vars.y }}`, `{{ trigger.body }}`. |
| **Retries** | Per-task `retry:` (type `constant`/`exponential`, `maxAttempt`, `interval`). |
| **errors block** | Flow-level `errors:` tasks run on failure (notify/cleanup). |
| **Labels** | Key/value metadata for filtering, audit, multi-tenancy. |
| **Workers / executor** | Executor schedules the DAG; workers run tasks; scale workers for fan-out. |
| **Backfills** | Replay a `Schedule` trigger over a past time range (reprocess history). |

---

## 2. TC → Kestra mapping

| TC construct | Kestra |
|---|---|
| TC Event / Timer trigger | `io.kestra.plugin.core.trigger.Schedule` (poll/cron) |
| HTTP/Webhook trigger | `io.kestra.plugin.core.trigger.Webhook` |
| PlaybookComponent trigger | `io.kestra.plugin.core.flow.Subflow` (callee) |
| Iterator | `io.kestra.plugin.core.flow.EachSequential` / `EachParallel` |
| Logic / Decision | `io.kestra.plugin.core.flow.If` / `Switch` |
| Set Variable | task `outputs` / Pebble vars |
| Merge | join: read multiple `outputs.*` in a following task |
| Sleep / delay | `io.kestra.plugin.core.flow.Pause` (with `delay`) |
| Component | `io.kestra.plugin.core.flow.Subflow` |
| Fail / Stop | `io.kestra.plugin.core.flow.Fail` + `errors` block |
| Transform/enrichment app | `io.kestra.plugin.scripts.python.Script` (deterministic) |
| Judgment node | `io.kestra.plugin.core.http.Request` → ADK endpoint |

---

## 3. Calling an ADK agent from Kestra

Two integration patterns:

**(A) ADK as HTTP endpoint — PRIMARY.** Deploy the agent to Cloud Run or Vertex AI Agent Engine;
call it with `io.kestra.plugin.core.http.Request`. Scales independently, decouples runtimes,
versioned deploys, clean least-privilege boundary (agent service holds its own creds; Kestra holds
none of the agent's).

```yaml
- id: triage
  type: io.kestra.plugin.core.http.Request
  uri: "{{ secret('ADK_TRIAGE_URL') }}/triage"
  method: POST
  contentType: application/json
  body: |
    {"indicator": {{ outputs.normalize.vars.entity | json }}}
  headers:
    Authorization: "Bearer {{ secret('ADK_TOKEN') }}"
```

**(B) ADK Runner inline in a `python.Script` task — low-volume only.** Import the ADK `Runner`,
run the agent in-process. Simpler deploy, but couples runtimes and scales poorly. Reserve for
infrequent or batch flows.

> Recommendation: **A** by default; **B** only for low-volume/back-office flows.

---

## 4. Human-in-the-loop approval gate

Use `io.kestra.plugin.core.flow.Pause` before any destructive or outbound action (TC write, ticket
creation, notification). Agents never write to TC without passing this gate.

```yaml
- id: approval
  type: io.kestra.plugin.core.flow.Pause
  # resumes on manual UI approval or an API/webhook resume call
```

`Pause` can also implement Sleep (with `delay:`). Resume via Kestra API, UI, or a TaskComplete
webhook from TC Cases.

---

## 5. Fan-out enrichment at scale

`EachParallel` over an indicator array; cap concurrency and add backoff retries to respect provider
rate limits.

```yaml
- id: enrich_all
  type: io.kestra.plugin.core.flow.EachParallel
  concurrencyLimit: 10          # throttle / rate-limit
  value: "{{ outputs.normalize.vars.indicators }}"
  tasks:
    - id: enrich_one
      type: io.kestra.plugin.core.http.Request
      uri: "{{ secret('VT_URL') }}/{{ taskrun.value }}"
      headers:
        x-apikey: "{{ secret('VT_API_KEY') }}"
      retry:
        type: exponential
        maxAttempt: 4
        interval: PT2S
        maxInterval: PT1M
```

`concurrencyLimit` throttles outbound calls; `exponential` retry handles 429/5xx.

---

## 6. Full example flow

Webhook → normalize (deterministic) → ADK triage (HTTP) → conditional approval `Pause` if
severity≥4 → write-back to TC → `errors` block notifies Slack.

```yaml
id: indicator_triage_and_writeback
namespace: ti.enrichment

inputs:
  - id: source
    type: STRING
    defaults: webhook

tasks:
  - id: normalize
    type: io.kestra.plugin.scripts.python.Script
    # deterministic: parse trigger body -> shared TCEntity model
    script: |
      import json, sys
      body = json.loads(r'''{{ trigger.body }}''')
      entity = {"value": body["ioc"], "type": body.get("type", "Address")}
      print('::{"outputs":{"vars":{"entity":' + json.dumps(entity) + '}}}::')

  - id: triage
    type: io.kestra.plugin.core.http.Request
    uri: "{{ secret('ADK_TRIAGE_URL') }}/triage"
    method: POST
    contentType: application/json
    body: '{"indicator": {{ outputs.normalize.vars.entity | json }}}'
    headers:
      Authorization: "Bearer {{ secret('ADK_TOKEN') }}"

  - id: gate
    type: io.kestra.plugin.core.flow.If
    condition: "{{ json(outputs.triage.body).severity >= 4 }}"
    then:
      - id: approval
        type: io.kestra.plugin.core.flow.Pause
        # block destructive write-back until analyst approves

  - id: writeback
    type: io.kestra.plugin.core.http.Request
    uri: "{{ secret('TC_TOOLS_URL') }}/indicators"
    method: POST
    contentType: application/json
    headers:
      Authorization: "Bearer {{ secret('TC_TOKEN') }}"
    body: |
      {
        "entity": {{ outputs.normalize.vars.entity | json }},
        "disposition": {{ json(outputs.triage.body) | json }}
      }

triggers:
  - id: inbound
    type: io.kestra.plugin.core.trigger.Webhook
    key: "{{ secret('WEBHOOK_KEY') }}"

errors:
  - id: notify_slack
    type: io.kestra.plugin.core.http.Request
    uri: "{{ secret('SLACK_WEBHOOK_URL') }}"
    method: POST
    contentType: application/json
    body: '{"text": "Flow {{ flow.id }} failed at execution {{ execution.id }}"}'
```

Note: `writeback` targets a deterministic **TC tools** service (your TC v3 REST client behind HTTP),
not the agent — agents never hold TC credentials or write directly.

---

## 7. Secrets handling

Never inline credentials. Reference Kestra secrets:

```yaml
headers:
  x-apikey: "{{ secret('VT_API_KEY') }}"
```

Secrets are injected from the configured backend (env-encoded, Secret Manager, Vault). The TC client
service holds TC creds; provider keys live as Kestra secrets; ADK services hold their own — least
privilege across the board.

---

## 8. Backfills & audit

- **Backfill** a `Schedule` trigger to reprocess a historical window after fixing a bug — replays
  deterministically because the control plane is deterministic.
- **Audit:** every execution, task run, input, and output is recorded; add `labels` (owner,
  playbook id, env) for filtering and compliance. Use a **shadow run** (flow runs, write-backs
  stubbed/logged) before cutover.
