---
name: remerrill-com-find-pump-line
description: Turn a pumping duty into a deterministic shortlist of R.E. Merrill-represented pump lines — over plain REST or over A2A — and hand everything the API refuses to decide (chemical compatibility, service fit, quotation) to a human engineer through the rfqUrl it returns.
api: openapi/remerrill-com-openapi.yml
operations:
  - findPumpLines
  - sendA2AMessage
method: generated
generated: '2026-09-19'
grounding: >-
  Both operationIds exist in openapi/remerrill-com-openapi.yml, which was itself generated from the provider's
  published descriptions and live responses (see its info.x-apievangelist-provenance). Field names, the enum,
  the exactly-one-of rules and every quoted sentence come from https://www.remerrill.com/llms.txt, the agent
  card, or the agent's own reply. Nothing here was invented.
---

# Find the pump lines that fit a duty (R.E. Merrill & Associates)

Base URL `https://www.remerrill.com`. Everything is `application/json`. No key, no account, no cost.

## 0. Know what this API will and will not do

- It is **deterministic**: "answers come from curated, manufacturer-published data, never generated numbers." Every verdict names the smallest covering model, its family, the source catalog and the printed page.
- It decides **flow and pressure envelopes only**. It never resolves fluid, concentration or temperature — "chemical compatibility is a safety determination confirmed by an REM engineer" — and it does not apply viscosity, particle-size or abrasive deratings. Those come back under `unresolved[]`, not as an error.
- Curated coverage today is **Grundfos dosing** (116 model envelopes, 2012 brochure) and **Continental progressing cavity** (105 envelopes, 2021 catalog). The other eleven represented lines are "consult-REM by design" and appear under `coverage.consultRem`.
- It **quotes nothing**. "No pricing is returned by the agent; quotes come from an engineer." Every response carries an `rfqUrl`.
- It stores nothing and needs no idempotency or reversal handling: repeat freely.

## 1. Build the duty

```json
{ "fluid": "sodium hypochlorite", "concentrationPct": 12.5, "flowGph": 2, "pressurePsi": 40, "service": "feed" }
```

- `fluid` (string) is the only required field.
- Send **exactly one** of `flowGpm` / `flowGph`, and **exactly one** of `pressurePsi` / `headFt`. US units.
- `service` is optional and, if sent, one of `transfer | feed | sump | slurry | boiler`.
- `concentrationPct` and `temperatureF` are optional and are passed through to the engineer, not evaluated.

Do not send natural language to either endpoint — "This agent ... does not interpret natural language." Over A2A a prose message gets a polite schema restatement back, not a verdict.

## 2. Call it — REST (`findPumpLines`)

`POST /api/line-finder` with the duty as the body. Only POST is served (GET → 405; OPTIONS → 204, `Allow: OPTIONS, POST`).

- **200** → `LineFinderResult`: `matches[]` (per line: `families[]` with `matchingModels`, and a `rationale` string citing the catalog page), `unresolved[]`, `coverage{curated[], consultRem[], note}`, `rfqUrl`.
- **400** → `{error: "Invalid request", reason: "<field>: Invalid input ...", rfqUrl: "/contact"}`. Fix the named fields and resend.

## 2b. Or call it — A2A (`sendA2AMessage`)

`POST /api/a2a` with header `A2A-Version: 1.0` and a JSON-RPC 2.0 body, method `SendMessage`, the duty as a **data part**:

```json
{"jsonrpc":"2.0","id":1,"method":"SendMessage","params":{"message":{"messageId":"m1","role":"ROLE_USER","parts":[{"data":{"fluid":"sodium hypochlorite","concentrationPct":12.5,"flowGph":2,"pressurePsi":40,"service":"feed"}}]}}}
```

- A resolvable duty returns `result.task` with `status.state = TASK_STATE_COMPLETED` and one artifact named `line-finder-result` whose first part's `data` is exactly the REST `LineFinderResult` (second part: a text summary).
- **Do not poll.** Tasks are not retained: `GetTask` for any id returns JSON-RPC error `-32001 Task not found` with a `google.rpc.ErrorInfo` detail saying so. No streaming, no push notifications.
- Discovery: `https://www.remerrill.com/.well-known/agent-card.json` (JWS-signed; keys at `/.well-known/jwks.json`).

## 3. Read the verdict honestly

- An **empty `matches[]` is a normal 200**, not a failure: the duty falls outside every curated envelope. Route to consult-REM.
- Treat `rationale` as the citation trail — it names the source document and printed page for the smallest covering model. Catalog performance is stated for water; the provider re-confirms for the actual fluid.
- Surface `unresolved[]` to the user verbatim. It is the provider's list of what a human must still decide.

## 4. Hand off

Resolve `rfqUrl` against `https://www.remerrill.com` (it is site-relative, e.g. `/contact?duty=...`) and send the user there — it prefills the contact form with the duty for a human quotation. Alternatively `info@remerrill.com` or 713-349-0909. There is no API path to a quote, and none should be simulated.
