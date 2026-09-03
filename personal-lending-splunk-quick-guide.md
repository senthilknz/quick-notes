# Personal Lending xAPI — Splunk Quick Guide

**Service:** `xapi-wone-customer-offer` · **Splunk app:** Apps → **585 - Wone Team** (no access? raise a ServiceNow request) · **Owner:** _<name>_ · **Last verified:** _<date>_

**Before you run anything**
- Replace `<env>` with `syst`, `uat` or `prod`.
- `` `xw1co_all(<env>)` `` = all levels · `` `xw1co_info(<env>)` `` = INFO only. Anything about failures needs `xw1co_all`.
- **Get the `correlationId` first.** It is on every log line of a request. If you only have a `crsNo` or `applicationId`, use Q1/Q2 to find it.
- Placeholders `<correlation-id>`, `<application-id>`, `<crsNo>` — never paste real ones into this page.

---

## Customer says … → run

| Symptom | Run |
|---|---|
| Anything — start here | **Q1** find the request → **Q3** trace it → **Q4** what did we send back |
| "My application is stuck" | **Q3** journey → **Q5** did state persist |
| "Upload failed / only some files" | **Q6** upload result |
| "Finance summary never loads" | **Q7** polling status |
| "Unexpected error" | **Q4** what we returned → **Q8** downstream failures |
| "It's slow" | **Q10** latency |
| **Many customers at once** | **Q9** service health → **Q8** downstream → **Q10** latency |

---

## Q1 — Find a customer's requests (from crsNo)

```spl
`xw1co_info(<env>)`
| search message="*crsNo=<crsNo>*"
| rex field=message "^(?<http_method>\w+)\s(?<path>\S+)"
| table _time http_method path correlationId
```
Take the `correlationId` into Q3.

## Q2 — Follow one application's whole journey (from applicationId)

```spl
`xw1co_all(<env>)`
| search message="*<application-id>*"
| sort 0 _time
| rex field=message "op=(?<op>[\w/\-]+)"
| table _time level op logger message
```
Look for the sequence `op=start → documents → finance-summary → update` and the `formStatus OUT … "status":"IsComplete"` at the end. Missing steps = where the customer stopped or where we failed.

## Q3 — Trace one request (the primary support query)

```spl
`xw1co_all(<env>)`
| search correlationId="<correlation-id>"
| sort 0 _time
| table _time level logger message
```
Read top to bottom: `RequestLoggingFilter` line = status + `durationMs` · `PATCH=` line = did we save state · `<sys> ->` / `<- success|failed` = downstream calls · `GlobalExceptionHandler` line = **what the customer got back**.

## Q4 — What did we return to customers? (all errors)

```spl
`xw1co_all(<env>)`
| search logger="*GlobalExceptionHandler"
| table _time level message correlationId
```
WARN = client mistake (4xx), usually not our fault. ERROR = 5xx, ours or downstream. `Unexpected error` = investigate immediately. Add `correlationId="<correlation-id>"` for one request.

## Q5 — Did state persist? (stuck applications)

```spl
`xw1co_all(<env>)`
| search message="*PATCH=*"
| rex field=message "op=(?<op>[\w/\-]+)"
| rex field=message "applicationId=(?<applicationId>[0-9a-f\-]+)"
| rex field=message "PATCH=(?<decision>CALLED|SKIPPED|FAILED)"
| rex field=message "reason=(?<reason>\S+)"
| table _time op applicationId decision reason correlationId
```
`CALLED` = saved. `FAILED` = **not saved** — the app shows stale state; run Q8 on that correlationId, the lending-application PATCH failed. `SKIPPED reason=already-inProgress` on `op=documents` is normal (re-upload), not a fault.

## Q6 — Upload result (accepted vs rejected)

```spl
`xw1co_info(<env>)`
| search message="*<- success: POST*outcome success=*"
| rex field=message "outcome success=(?<ok>\w+) accepted=(?<accepted>\d+) rejected=(?<rejected>\d+)"
| rex field=message "applicationId=(?<applicationId>[0-9a-f\-]+)"
| table _time applicationId ok accepted rejected correlationId
```
`rejected>0` = partial upload, customer got 200 but not all files stored. **No row at all** for the correlationId → check Q3: an `Upload files [...]` line with no `upload decoded` line = our validation rejected it (bad type/size, WARN `Bad input`); a `<- failed` line = lending-documents was down (see Q8).

## Q7 — Finance summary "not ready" (503 polling)

```spl
`xw1co_all(<env>)`
| search message="Finance summary not ready*"
| rex field=message "reason='(?<reason>[^']*)'\sdocumentsPending=(?<pending>\d+)/(?<total>\d+)"
| table _time reason pending total correlationId
```
A steady trickle is **normal** — the app polls while documents are categorised. One customer stuck at the same `documentsPending` for many minutes = documents never finished downstream → escalate to transaction-categorisation.

## Q8 — Downstream failures, with HTTP status

```spl
`xw1co_all(<env>)`
| search message="*API error*status=*"
| rex field=message "^(?<system>.+?) API error"
| rex field=message "status=(?<httpStatus>\d+)"
| table _time system httpStatus message correlationId
```
Tells you *which* system (lending-application / lending-documents / transaction-categorisation) failed and with what. `429` = they load-shed us; `5xx` = their fault → escalate to that team with the correlationId.

## Q9 — Is the service healthy? (incident view)

```spl
`xw1co_info(<env>)`
| search logger="*RequestLoggingFilter" endpoint="*personal-lending*" message="* -> status=*"
| rex field=message "->\sstatus=(?<status>\d+)"
| eval class=substr(status,1,1)."xx"
| timechart span=5m count by class
```
Traffic gone to zero = ingress/front-end/pods. `5xx` climbing = run Q4 and Q8. `4xx` spike = front-end release sending bad `formStatus`.
Also worth a glance: `` `xw1co_all(<env>)` | search message="*RESILIENCE*CIRCUITBREAKER*" | table _time message `` — a breaker opening is the early warning that a downstream is out (customers see 503).

## Q10 — Latency: where is the time going?

```spl
`xw1co_all(<env>)`
| search (message="*<- success:*" OR message="*<- failed:*")
| rex field=message "^(?<system>[\w\-]+) <- (?<outcome>success|failed):"
| rex field=message "durationMs=(?<durationMs>\d+)"
| stats p50(durationMs) p95(durationMs) max(durationMs) count by system outcome
```
One system's p95 spiking = that dependency is slow. For our own end-to-end numbers, swap the search for `logger="*RequestLoggingFilter" endpoint="*personal-lending*"` and `stats p95(durationMs) by endpoint`.

---

## Cheat sheet — customer saw … → it means

| Customer saw | Meaning | Log line to grep |
|---|---|---|
| **400** | Front-end sent bad `formStatus` or a bad request | `formStatus is not valid JSON` / `Bad input provided` / `Validation failed` |
| **415** | Wrong content type | `Unsupported media type` |
| **429** | We load-shed (too much traffic) | `Load shed by Resilience4j` |
| **500** | Unhandled error — investigate now | `Unexpected error` |
| **502 / 503 / 504** | A downstream failed | `<sys> <- failed:` + `<sys> API error – status=` |
| **503 + Retry-After** | Finance summary still processing (normal) — or circuit breaker open (not normal) | `Finance summary not ready` / `Circuit breaker open` |
| **200 but state is stale** | State not saved, or partial upload | `PATCH=FAILED` / `rejected=<n>` |
| Downstream **401 / 403** | Our outbound token failed | `Failed to add … Bearer token` |

**Escalating?** Send: `correlationId`, `applicationId`, `<env>`, timestamp, the `GlobalExceptionHandler` line, and the `API error – status=` line.
