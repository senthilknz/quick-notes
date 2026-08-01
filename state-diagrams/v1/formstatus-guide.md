# formStatus — the frontend guide

**Audience:** Frontend (FE) engineers integrating with the Personal Lending xAPI (`xapi-wone-customer-offer`).
**Scope:** How the `formStatus` value changes across the income and expenses journey, and what each side is responsible for.
**Status:** Draft. Pending a test-coverage check against the implementation — see §8.

---

## 1. Read this first — the six rules

If nothing else on this page sticks, these do.

| # | Rule | Why it matters |
|---|------|----------------|
| 1 | **Always echo back the `formStatus` returned by the last response.** Never rebuild it from scratch after the initial POST. | The blob carries keys the xAPI wrote and keys other screens wrote. Rebuilding it silently drops them. |
| 2 | **The frontend must never set `lendingApplyState.status`.** | The xAPI owns that transition. A value sent by the frontend is overwritten or ignored. |
| 3 | **The frontend does not need to create `likely-changes`.** The xAPI creates and stamps it. | Supplying `isIncomeLikelyToChange` on PUT sets `likely-changes` to `"IsComplete"`, whether or not the key already exists. |
| 4 | **Absent is not the same as null** on optional PUT fields. Omit a field entirely if it should not change. | Sending `null` may clear the value downstream. |
| 5 | **The response always carries the persisted `formStatus`.** Use it as the source of truth. | No follow-up GET is needed after any mutating call. |
| 6 | **One request can cascade.** Supplying `isIncomeLikelyToChange` on PUT can stamp `likely-changes` *and* complete the whole journey in a single call. | Do not assume one call equals one state change. |

---

## 2. What `formStatus` actually is

`formStatus` is an **opaque, double-encoded JSON string** that the frontend round-trips on every request. The xAPI reads it, mutates a small set of known keys, and writes it back. Every other key the frontend puts in there is preserved untouched.

Think of it as a bag the frontend carries through the journey. The xAPI reaches into the bag and changes six things. Everything else in the bag belongs to the frontend.

### Key ownership

```mermaid
flowchart TB
    FS["formStatus<br/><i>JSON string — always echo back</i>"]

    FS --> X["<b>xAPI writes these</b><br/>FE must not set them"]
    FS --> FE["<b>FE-owned</b><br/>xAPI never reads or touches these"]

    X --> X1["lendingApplyState.status"]
    X --> X2["lendingApplyState.otherFinancialProviders"]
    X --> X3["eligibility.feature<br/>eligibility.isEligible<br/>eligibility.reason"]
    X --> X4["your-income<br/>fixed-commitments<br/>essential-living-costs<br/>recurring-expenses<br/><i>stamped by finance-summary</i>"]
    X --> X5["likely-changes<br/><i>created and stamped on PUT</i>"]

    FE --> F1["any other custom keys<br/><i>preserved unchanged</i>"]

    classDef xapi fill:#1D4E89,stroke:#0F2E52,stroke-width:2px,color:#FFFFFF
    classDef fe fill:#2E7D5B,stroke:#1B4D37,stroke-width:2px,color:#FFFFFF
    classDef root fill:#3C3F44,stroke:#1F2124,stroke-width:2px,color:#FFFFFF

    class FS root
    class X,X1,X2,X3,X4,X5 xapi
    class FE,F1 fe
```

| Owner | Keys | Rule |
|-------|------|------|
| **xAPI** | `lendingApplyState.*`, `eligibility`, and all five section markers (`your-income`, `fixed-commitments`, `essential-living-costs`, `recurring-expenses`, `likely-changes`) | The xAPI writes these. The frontend must not set them manually. |
| **Frontend** | Any other custom keys | The frontend creates and sets the values. The xAPI preserves them and never reads them. |

### Full shape

A `formStatus` blob mid-journey, decoded and pretty-printed. On the wire this is a single escaped string inside the request or response body — see §5 for the escaped form.

```json
{
  "lendingApplyState": {
    "status": "InProgress",
    "otherFinancialProviders": true
  },
  "eligibility": {
    "feature": "INCOME_AND_EXPENSES",
    "isEligible": true
  },
  "your-income": "IsComplete",
  "fixed-commitments": "IsComplete",
  "essential-living-costs": "IsComplete",
  "recurring-expenses": "IsComplete",
  "likely-changes": "NotStarted",
  "anyKeyOfYourOwn": "preserved untouched"
}
```

### Key reference

| Key | Type / values | Written by | When |
|---|---|---|---|
| `lendingApplyState.status` | `"NotStarted"` \| `"InProgress"` \| `"IsComplete"` | xAPI | Create, first document upload, and the PUT completion cascade |
| `lendingApplyState.otherFinancialProviders` | `true` \| `false` | xAPI | PUT, folded in from the `otherFinancialProviders` request field |
| `eligibility.feature` | `"INCOME_AND_EXPENSES"` | xAPI | Create, always |
| `eligibility.isEligible` | `true` \| `false` | xAPI | Create, derived from `isJointApplication` |
| `eligibility.reason` | `"JOINT_APPLICANT"` | xAPI | Create, only when `isEligible` is `false` |
| `your-income` | `"IsComplete"` | xAPI | `finance-summary`, on a successful categorised report |
| `fixed-commitments` | `"IsComplete"` | xAPI | `finance-summary`, on a successful categorised report |
| `essential-living-costs` | `"IsComplete"` | xAPI | `finance-summary`, on a successful categorised report |
| `recurring-expenses` | `"IsComplete"` | xAPI | `finance-summary`, on a successful categorised report |
| `likely-changes` | `"IsComplete"` | xAPI | PUT, when `isIncomeLikelyToChange` is supplied and the key is not already `"IsComplete"`. Created if absent |
| *any other key* | Anything | Frontend | Never read or modified by the xAPI |

> `lendingApplyState` may gain further xAPI-owned fields later. Treat the whole object as xAPI territory rather than enumerating what is in it today.

---

## 3. The journey lifecycle

Three states, and the xAPI decides all three.

```mermaid
stateDiagram-v2
    direction TB

    [*] --> NotStarted : POST /personal-lending<br/>(always forced on create)
    NotStarted --> InProgress : POST .../documents<br/>(first successful upload)
    InProgress --> InProgress : POST .../documents<br/>(already InProgress → PATCH skipped)
    InProgress --> IsComplete : PUT /personal-lending/{id}<br/>(all 5 sections IsComplete)
    IsComplete --> IsComplete : PUT /personal-lending/{id}<br/>(already IsComplete → no-op)
    IsComplete --> [*]

    classDef notStarted fill:#8C8C8C,stroke:#5A5A5A,stroke-width:2px,color:#FFFFFF
    classDef inProgress fill:#C77700,stroke:#8A5300,stroke-width:2px,color:#FFFFFF
    classDef complete fill:#2E7D5B,stroke:#1B4D37,stroke-width:2px,color:#FFFFFF

    class NotStarted notStarted
    class InProgress inProgress
    class IsComplete complete
```

### Wire values

Exact strings in the JSON. Case sensitive.

| State | Wire code |
|-------|-----------|
| Not started | `"NotStarted"` |
| In progress | `"InProgress"` |
| Complete | `"IsComplete"` |

---

## 4. The completion gate — five markers

`lendingApplyState.status` can only become `"IsComplete"` when **all five** of these top-level keys exist **and** equal `"IsComplete"`.

```mermaid
flowchart LR
    A["your-income"] --> G{"All five<br/>= IsComplete?"}
    B["fixed-commitments"] --> G
    C["essential-living-costs"] --> G
    D["recurring-expenses"] --> G
    E["likely-changes"] --> G

    G -->|Yes| Y["lendingApplyState.status<br/>= IsComplete"]
    G -->|No| N["No status change"]

    classDef auto fill:#1D4E89,stroke:#0F2E52,stroke-width:2px,color:#FFFFFF
    classDef manual fill:#B8860B,stroke:#7A5A07,stroke-width:2px,color:#FFFFFF
    classDef gate fill:#3C3F44,stroke:#1F2124,stroke-width:2px,color:#FFFFFF
    classDef yes fill:#2E7D5B,stroke:#1B4D37,stroke-width:2px,color:#FFFFFF
    classDef no fill:#8C8C8C,stroke:#5A5A5A,stroke-width:2px,color:#FFFFFF

    class A,B,C,D auto
    class E manual
    class G gate
    class Y yes
    class N no
```

| Section marker key | Stamped by | When |
|---|---|---|
| `your-income` | xAPI (finance-summary) | Successful categorised report |
| `fixed-commitments` | xAPI (finance-summary) | Successful categorised report |
| `essential-living-costs` | xAPI (finance-summary) | Successful categorised report |
| `recurring-expenses` | xAPI (finance-summary) | Successful categorised report |
| `likely-changes` | xAPI (PUT) | `isIncomeLikelyToChange` is supplied and the key is not already `"IsComplete"` |

> **`likely-changes` is the odd one out.** The other four are stamped together by a successful `finance-summary`. This one is only stamped on PUT, when the customer answers the income-change question. That is why an application can have a successful categorised report and still sit at `"InProgress"` — four markers are set, the fifth is waiting on that answer.

---

## 5. Walkthrough — one journey, call by call

The happy path. Watch the `formStatus` grow.

```mermaid
flowchart TB
    S1["<b>1. POST /personal-lending</b><br/>Send: {} or FE-owned keys<br/>Get back: lendingApplyState + eligibility<br/>status = NotStarted"]
    S2["<b>2. POST .../documents</b><br/>Send: formStatus + base64 PDFs<br/>Get back: status = InProgress"]
    S3["<b>3. POST .../finance-summary</b><br/>Send: formStatus (poll until 200)<br/>Get back: 4 section markers stamped"]
    S4["<b>4. PUT /personal-lending/{id}</b><br/>Send: formStatus + isIncomeLikelyToChange<br/>Get back: likely-changes stamped<br/>status = IsComplete"]

    S1 --> S2 --> S3 --> S4

    classDef ns fill:#8C8C8C,stroke:#5A5A5A,stroke-width:2px,color:#FFFFFF
    classDef ip fill:#C77700,stroke:#8A5300,stroke-width:2px,color:#FFFFFF
    classDef done fill:#2E7D5B,stroke:#1B4D37,stroke-width:2px,color:#FFFFFF

    class S1 ns
    class S2,S3 ip
    class S4 done
```

### Step 1 — Create the application

**Before** (what the frontend sends)

```json
{ "formStatus": "{}" }
```

**After** (what comes back)

<!-- VERIFY: paste the real enriched formStatus string from a POST response -->

```json
{
  "formStatus": "{\"lendingApplyState\":{\"status\":\"NotStarted\"},\"eligibility\":{\"feature\":\"INCOME_AND_EXPENSES\",\"isEligible\":true}}"
}
```

**What changed:** `lendingApplyState.status` forced to `"NotStarted"`, `eligibility` block added.

### Step 2 — Upload documents

**Before:** the blob from step 1, echoed back unchanged.

**After**

<!-- VERIFY: paste the real formStatus string from a documents POST response -->

**What changed:** `lendingApplyState.status` moved `"NotStarted"` → `"InProgress"`. If the status was already `"InProgress"`, nothing changes and no downstream PATCH is made.

### Step 3 — Poll the categorised report

**After a 200**

<!-- VERIFY: paste the real formStatus string from a finance-summary 200 response -->

**What changed:** four section markers stamped to `"IsComplete"` — `your-income`, `fixed-commitments`, `essential-living-costs`, `recurring-expenses`. Status is still `"InProgress"` because `likely-changes` is not stamped until the PUT in step 4.

### Step 4 — Answer "is your income likely to change?"

This is the cascade. One request, three mutations.

**Request**

```json
{
  "data": {
    "formStatus": "{\"lendingApplyState\":{\"status\":\"InProgress\"},\"your-income\":\"IsComplete\",\"fixed-commitments\":\"IsComplete\",\"essential-living-costs\":\"IsComplete\",\"recurring-expenses\":\"IsComplete\"}",
    "isIncomeLikelyToChange": true,
    "income": ["..."],
    "fixedCommitments": ["..."],
    "recurringExpenses": ["..."],
    "essentialLivingCosts": ["..."]
  }
}
```

**What the xAPI does, in order**

1. `otherFinancialProviders` not supplied → skip.
2. `isIncomeLikelyToChange` supplied **and** `likely-changes` is not already `"IsComplete"` → create it if needed and stamp it `"IsComplete"`.
3. All five sections are now `"IsComplete"` → set `lendingApplyState.status` = `"IsComplete"`.
4. PATCH downstream with the final `formStatus`, all catalogues, and `isIncomeLikelyToChange: true`.

**Response**

```json
{
  "data": {
    "formStatus": "{\"lendingApplyState\":{\"status\":\"IsComplete\"},\"your-income\":\"IsComplete\",\"fixed-commitments\":\"IsComplete\",\"essential-living-costs\":\"IsComplete\",\"recurring-expenses\":\"IsComplete\",\"likely-changes\":\"IsComplete\"}"
  }
}
```

**Result:** journey completed in a single request.

---

## 6. Endpoint reference

### 6.1 Quick reference — what needs to be sent?

| Endpoint | `formStatus` required? | Other required fields | What comes back |
|---|---|---|---|
| `POST /personal-lending` | Optional (can be `{}`) | `isJointApplication` | Enriched `formStatus` with `lendingApplyState` + `eligibility` |
| `POST .../documents` | Mandatory | `files[]` (base64 PDFs) | Updated `formStatus` (status may change to `InProgress`) |
| `POST .../finance-summary` | Mandatory | — | Updated `formStatus` (4 sections stamped) + report data |
| `PUT /personal-lending/{id}` | Mandatory | — (all else optional) | Updated `formStatus` (possibly cascaded to `IsComplete`) |

### 6.2 `POST /v1/customer-offer/personal-lending` — start application

| What the xAPI does to `formStatus` | Always / Conditional |
|---|---|
| Forces `lendingApplyState.status = "NotStarted"` | Always, even if the frontend sends a different value |
| Sets `eligibility.feature = "INCOME_AND_EXPENSES"` | Always |
| Sets `eligibility.isEligible` based on `isJointApplication` | Always |
| Preserves all other frontend-supplied keys | Always |

**Frontend action:** send the initial `formStatus` blob. It can be `{}`. The response echoes the enriched version.

### 6.3 `POST /v1/customer-offer/personal-lending/{id}/documents` — upload documents

| Condition | What happens to `formStatus` |
|---|---|
| `lendingApplyState.status` is **not** `"InProgress"` | Sets status to `"InProgress"`, then PATCHes downstream |
| `lendingApplyState.status` is **already** `"InProgress"` | No PATCH. Echoes the incoming `formStatus` unchanged |

**Frontend action:** always send `formStatus` in the request body. The response echoes the possibly updated value.

### 6.4 `POST /v1/customer-offer/personal-lending/{id}/finance-summary` — poll categorised report

| Condition | What happens to `formStatus` |
|---|---|
| Downstream report succeeds | Stamps four section markers to `"IsComplete"`: `your-income`, `fixed-commitments`, `essential-living-costs`, `recurring-expenses`. PATCHes if any changed |
| Downstream report not ready (503) | No mutation. Returns 503 with a `Retry-After` header |
| Downstream report fails | No mutation. Error propagated |

**Frontend action:** poll until a 200 is returned. The response carries the updated `formStatus` with the four sections stamped.

### 6.5 `PUT /v1/customer-offer/personal-lending/{id}` — update application

The most complex endpoint. Several independent mutations can cascade in a single request.

**Mutation order, executed top to bottom in one call:**

```text
Step 1: Fold otherFinancialProviders (if supplied)
        → sets lendingApplyState.otherFinancialProviders

Step 2: Create or stamp likely-changes
        (if isIncomeLikelyToChange supplied
         and likely-changes is not already "IsComplete")
        → sets likely-changes = "IsComplete"

Step 3: Check all-five-sections-complete
        → sets lendingApplyState.status = "IsComplete"

Step 4: Build downstream PATCH with formStatus + supplied fields
        (income, recurringExpenses, fixedCommitments,
         essentialLivingCosts, isIncomeLikelyToChange)
```

**Detailed decision table**

| # | Condition (what the frontend sends) | `formStatus` mutation | Downstream field |
|---|---|---|---|
| 1 | `otherFinancialProviders` is present | `lendingApplyState.otherFinancialProviders` = the supplied value | Not sent as its own field. Lives inside `formStatus` only |
| 2 | `isIncomeLikelyToChange` present **AND** `likely-changes` is absent, null, or ≠ `"IsComplete"` | `likely-changes` = `"IsComplete"`, created if it does not exist | `isIncomeLikelyToChange` = supplied value |
| 3 | `isIncomeLikelyToChange` present **AND** `likely-changes` already = `"IsComplete"` | No mutation (idempotent) | `isIncomeLikelyToChange` = supplied value |
| 4 | `isIncomeLikelyToChange` **absent** from request | No mutation to `likely-changes` | Not sent downstream |
| 5 | After steps 1–2: all five section markers = `"IsComplete"` | `lendingApplyState.status` = `"IsComplete"` | — |
| 6 | After steps 1–2: **not** all five sections complete | No status change | — |
| 7 | `lendingApplyState.status` already = `"IsComplete"` when row 5 triggers | No-op (idempotent) | — |

> **Rows 2 and 3 are the whole of step 2.** The only case where `likely-changes` is left alone is when it is already `"IsComplete"`. Whether the key previously existed makes no difference — an absent key is treated the same as one holding any other value.
>
> **`isIncomeLikelyToChange` is checked for presence, not for truth.** Sending `false` stamps the marker exactly as `true` does. The customer has answered the question either way, so the section is complete either way. Only omitting the field entirely leaves `likely-changes` untouched (row 4).

---

## 7. Troubleshooting — why is the status stuck?

| Symptom | Most likely cause | Fix |
|---|---|---|
| All screens done, status still `"InProgress"` | `isIncomeLikelyToChange` has not been sent on a PUT yet, so `likely-changes` is unstamped | Send `isIncomeLikelyToChange` on PUT once the customer answers the income-change question |
| Status never left `"NotStarted"` | No successful document upload yet | The transition happens on the first successful `POST .../documents` |
| Section markers never appear | `finance-summary` is returning 503 | Keep polling. Honour the `Retry-After` header |
| A custom key has vanished | `formStatus` was rebuilt instead of echoed | Always send back the exact string from the last response |
| `otherFinancialProviders` not visible downstream | It is not a downstream field | It lives inside `formStatus` only. Read it from `lendingApplyState` |
| A field that should not have changed came back empty | `null` was sent instead of omitting the field | Absent is not null. Omit the key entirely |
| Second document upload produced no change | Status was already `"InProgress"` | Expected. The PATCH is skipped by design |
| Repeated PUT after completion changes nothing | Status already `"IsComplete"` | Expected. Rows 4 and 8 are idempotent |

---

## 8. Verification checklist

<!-- Internal. Remove before publishing to Confluence, or keep in the repo copy only. -->

Points to confirm against `FormStatusProcessor`, `FormStatusProcessorTest`, `PersonalLendingApplicationService` and `schemas.yaml` before this page is treated as authoritative:

1. **`likely-changes` creation — confirmed, but check test coverage.** The xAPI creates and stamps the key whenever `isIncomeLikelyToChange` is supplied and the key is not already `"IsComplete"`; an absent key is treated the same as any non-`"IsComplete"` value. Two cases worth a dedicated test, since both take the same branch as an already-covered case and a gap would go unnoticed: (a) the absent-key case as distinct from present-but-different, and (b) `isIncomeLikelyToChange: false`, which should stamp the marker exactly as `true` does.
2. **Endpoint path spelling.** This guide uses `finance-summary`. The sequence diagram uses `finances-summary` in at least one place. Confirm which is live.
3. **Case sensitivity of wire values.** The sequence diagram shows `inProgress` and `isComplete` in places, against `"InProgress"` and `"IsComplete"` here. Confirm the exact casing the processor writes and compares.
4. **Decision table coverage.** Each of the seven rows in §6.5 should map to a test in `FormStatusProcessorTest`. List any row with no test, and any test with no row.
5. **Payload placeholders.** Steps 1 to 3 in §5 contain `<!-- VERIFY -->` markers. Replace the illustrative JSON with real strings from responses or test fixtures.
6. **`eligibility.reason`.** Confirm it is written only when `isEligible` is false, and that `"JOINT_APPLICANT"` is the only value.
7. **DELETE documents.** `DELETE .../documents/{documentId}` appears in the sequence diagram with no `formStatus` mutation shown. Confirm it leaves `formStatus` untouched, and add a row to §6.1 if it does not.
8. **`otherFinancialProviders` type.** Boolean per the key reference. Check the schema agrees.
9. **Does `finance-summary` re-evaluate the completion gate?** This guide assumes the gate is only evaluated on PUT, so `finance-summary` leaves `status` at `"InProgress"`. Confirm.

---

## Publishing notes

- **Diagrams.** Editable draw.io sources sit alongside this file: `formstatus-key-ownership`, `formstatus-lifecycle`, `formstatus-completion-gate`, `formstatus-journey-flow`, `formstatus-put-mutation-order`. The Mermaid blocks above render the same content inline if the Confluence instance has a Mermaid macro; if not, embed the draw.io files or attach PNG exports.
- **Source of truth.** Keep this file in the repo alongside `FormStatusProcessor`. Treat the Confluence page as a published copy and re-publish from here rather than editing in place. The decision table in §6.5 is the part most likely to drift.
- **Suggested Confluence structure.** Section 1 as an info panel at the top, sections 6 and 7 inside expand macros, everything else on the page. Use status lozenges for `NotStarted` (grey), `InProgress` (yellow) and `IsComplete` (green) wherever the states appear in prose.