# TcEx Coupling → Clean Replacement KB

Audience: a code-forensics agent reading custom ThreatConnect app source (Python; some older Java).
Goal: extract pure business logic and **fully exit the TcEx library/runtime**, re-expressing logic
as ADK `FunctionTool`s (reasoning plane) or Kestra tasks (control plane), backed by a clean
**TC v3 REST + Batch** client. Deterministic stays deterministic — most TcEx apps are plumbing and
become non-LLM tools, not agents.

---

## 1. What TcEx actually provides

| TcEx facility | What it does | Why it's coupling |
|---|---|---|
| `tcex.inputs` / args | Parses `install.json` params + runtime args into typed inputs. | Ties logic to TcEx arg model. |
| `tcex.playbook.read(var)` | Reads upstream playbook variable by `#App:ID:name!Type`. | Couples to playbook variable wire format. |
| `tcex.playbook.create(var, val)` | Writes a typed output variable for downstream apps. | Same. |
| `tcex.api` / session client | Authenticated TC API session (HMAC/token), request signing. | Hides auth; couples to TcEx HTTP stack. |
| `tcex.batch` / `tcex.v3.batch` | Bulk indicator/group create with dedup + job polling. | Hidden batch semantics. |
| Token management | Acquires/renews API tokens from TC. | Embeds credential lifecycle. |
| Logging | `tcex.log`, ships logs to TC. | TC-bound logging. |
| Outputs declaration | `install.json` `playbookDataType`/`outputVariables`. | Output contract lives in JSON, not code. |

---

## 2. Clean replacement map

| TcEx pattern | Clean replacement |
|---|---|
| `tcex.inputs` / `install.json` args | **Typed function params** on an ADK `FunctionTool` (or Kestra task `inputs`). Pydantic for validation. |
| `tcex.playbook.read(...)` | **Function arguments** (pass upstream values in). Kestra wires them via `{{ outputs.x.vars.y }}` / ADK `session.state`. |
| `tcex.playbook.create(...)` | **Function return values** → Kestra task `outputs` / ADK `output_key` into `session.state`. |
| `tcex.api` session client | **Your own thin TC v3 REST client** (`requests`/`httpx`, explicit auth). |
| `tcex.batch` | **Re-implemented batch helper** in that client: `batch_create(items)` → submit `/v3/batch` + poll job. |
| Token management | **Secret Manager / Kestra secrets** (`{{ secret('TC_TOKEN') }}`); client reads creds, agents never do. |
| Logging | Standard `logging` → Kestra/Cloud Logging. |
| `install.json` outputs declaration | **Function return type / Pydantic model** (the schema lives in code). |

---

## 3. Extracting business logic from a TcEx app

1. **Find the entry point.** Locate `run()` (often `App.run()` in `app.py`/`run.py`), or the
   `main()` that builds `App(tcex)`. This is the I/O harness wrapper.
2. **Strip the harness.** Delete `tcex.inputs.*`, `tcex.playbook.read/create`, `tcex.exit`,
   arg parsing, and log shipping. They are not business logic.
3. **Classify each remaining statement** into three buckets:
   - **Pure logic** — transforms, scoring, parsing, branching (keep as-is; becomes the tool body).
   - **TC I/O** — reads/writes to TC (replace with your TC v3 client calls).
   - **External API calls** — VT/Shodan/etc. (keep; swap any TcEx HTTP helpers for `httpx`).
4. **Map `install.json` → signature.** Each `params[]` entry → a typed function parameter
   (respect `required`, `default`, `type`). Each `outputVariables[]` → a field on the return model.
5. **Result:** one clean function with typed params and a typed return. Wrap with
   `FunctionTool(fn)` if it's a tool an agent may call, or register directly as a Kestra Python task
   if it's deterministic plumbing.

> **Decision check:** does the extracted logic *decide* or *write prose*? If yes → it may belong to
> an ADK agent. If it transforms/calls/CRUDs → it stays a deterministic tool. See
> `agentic-decision-kb.md`.

---

## 4. Worked before/after

**Before — TcEx app `run()`** (`install.json` declares `score_threshold:String`,
`indicator:TCEntity`; output `disposition:String`):

```python
class App:
    def __init__(self, _tcex):
        self.tcex = _tcex

    def run(self):
        args = self.tcex.inputs.model           # I/O harness
        indicator = self.tcex.playbook.read(args.indicator)   # TC var read
        threshold = int(self.tcex.playbook.read(args.score_threshold))

        score = self._vt_score(indicator["value"])            # pure-ish logic + ext API
        disposition = "malicious" if score >= threshold else "benign"

        self.tcex.playbook.create("disposition", disposition) # TC var write
        self.tcex.exit(0)

    def _vt_score(self, ioc):
        r = self.tcex.session_external.get(f"https://vt/api/{ioc}")
        return r.json()["positives"]
```

**After — extracted clean ADK `FunctionTool`** (no TcEx, typed I/O, deterministic — note this is a
*tool*, not an agent; "malicious vs benign" via a fixed threshold is deterministic):

```python
import httpx
from pydantic import BaseModel
from google.adk.tools import FunctionTool

class Disposition(BaseModel):
    disposition: str
    score: int

def score_indicator(indicator_value: str, score_threshold: int = 5) -> Disposition:
    """Score an indicator against VirusTotal and apply a fixed threshold.

    Args:
        indicator_value: the IOC value (e.g. an IP, domain, or hash).
        score_threshold: positives at/above which the IOC is 'malicious'.
    """
    r = httpx.get(f"https://vt/api/{indicator_value}", timeout=15)
    score = r.json()["positives"]
    return Disposition(
        disposition="malicious" if score >= score_threshold else "benign",
        score=score,
    )

score_tool = FunctionTool(score_indicator)
```

TC variable reads → params; the write → a return value Kestra maps to a task output / ADK
`session.state`. Secrets (the VT key) come from Secret Manager, not the app.

---

## 5. TcEx-isms to strip — checklist

- [ ] `tcex.inputs`, `tcex.args`, `Arg`/`@configure` decorators → typed params.
- [ ] `tcex.playbook.read/create/check_variable` → params / returns.
- [ ] `tcex.exit(code, msg)` / `tcex.playbook.exit` → normal `return` / raise.
- [ ] `tcex.log` → `logging`.
- [ ] `tcex.api` / `tcex.session` → TC v3 client.
- [ ] `tcex.batch` / `tcex.v3.batch` → `batch_create` helper.
- [ ] `tcex.ti` / TI module entity builders → shared Pydantic `TCEntity`.
- [ ] `install.json` arg/output schema → Pydantic models.
- [ ] Token acquisition/renew → Secret Manager.

## 6. Gotchas

| Gotcha | Risk | Mitigation |
|---|---|---|
| Apps relying on **batch dedup** | Re-platform may double-write. | Re-implement dedup on `(type, value, owner)` in `batch_create`. |
| **Association helpers** (`indicator.associate_group`) | Hidden second API calls. | Make associations explicit calls in the TC client. |
| **TCEntity coercion** | TcEx silently normalizes/casts entity dicts. | Normalize explicitly via the shared `TCEntity` model + validators. |
| **Implicit owner context** | TcEx infers owner from runtime. | Pass `owner` explicitly to every client call. |
| **Output typing** (`!Binary`) | File content mishandled as text. | Map Binary→artifacts, never inline into text/LLM context. |
| **Stateful `tcex` object** | Logic reads from `self.tcex.*` mid-method. | Hoist all reads to params at function boundary. |

---

## 7. Java apps (older)

Some legacy apps are Java (TcEx Java SDK). **Same approach:** locate the `run()`/`execute()` entry,
strip the SDK harness (input parsing, playbook read/write, batch helpers), isolate the pure logic
and external/TC calls, then **re-express in Python** as a `FunctionTool` or Kestra task. Treat the
Java source as a spec for the logic, not as code to port line-for-line; the target runtime is Python
+ ADK + Kestra regardless of the original language.
