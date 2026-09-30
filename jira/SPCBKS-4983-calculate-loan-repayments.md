# Calculate Loan Repayments — Open Questions

Related Jira: SPCBKS-5032. Record each answer here, then update the matching acceptance criteria in the Jira.

## Scope and inputs

| # | Question | Owner | Status | Answer |
|---|---|---|---|---|
| Q1 | **Journeys.** The overview says Top-Up only, but the field notes also describe New Personal Loans. Is the endpoint used for both? | [BA / Product Owner] | Open | |
| Q2 | **Interest rate source.** The spec says the rate is "sourced by the xAPI from Pricing", but it is also a request field sent by the FE. Which is it? If the FE sends it, it comes from calculate-pricing, which returns a percentage (`13.40`), while this endpoint expects a decimal (`0.1340`). Who converts? | [Solution Designer] | Open | |
| Q3 | **FE-sent or xAPI-set.** `lendingType` and `repaymentType` are listed as request fields, but the notes say the xAPI sets them. Should the xAPI reject, ignore or accept them if the FE sends them? | [Solution Designer] | Open | |
| Q4 | **Top-Up loan amount.** For a Top-Up, `loanAmount` is the current balance plus the top-up amount. Does the FE calculate this, or does the xAPI? Where do `outstandingLoanBalance` and `accruedInterest` come from for a Top-Up? | [Solution Designer] | Open | |
| Q5 | **Dates.** `startDate` defaults to the xAPI current date. Who sets `nextPaymentDate`, and what is the default if it is omitted? | [Solution Designer] | Open | |
| Q6 | **Validation limits.** Should the xAPI validate amount and term against the min/max returned by validate-eligibility, or only check they are positive numbers? What decimal precision is allowed for amounts, term and rate? | [Solution Designer] | Open | |
| Q7 | **Ideal repayment below minimum.** Does the xAPI check this, or does LCS reject it? If LCS rejects it, how does the xAPI tell it apart from a genuine LCS failure so it returns 400, not 500? (The spec refers to OQ-4.) | [Solution Designer] | Open | |

## Behaviour and performance

| # | Question | Owner | Status | Answer |
|---|---|---|---|---|
| Q8 | **Call volume.** The FE calls this on every slider move. Will the FE debounce calls? Has LCS confirmed it can handle the expected volume? Do we need a timeout below the usual value? | [Dev Lead] | Open | |
| Q9 | **Rounding.** LCS returned `repaymentAmount: "449"` (no cents) in Bruno. What rounding rule applies, and should the xAPI return 2 decimal places? | [Loan Calculator team] | Open | |
| Q10 | **Number of payments.** In Bruno, a 1-year monthly loan returned `numberOfPayments: 11`. Confirm this is expected (first payment one month after start) rather than a defect. | [Loan Calculator team] | Open | |

## Conformance with the xAPI Specification

| # | Question | Owner | Status | Answer |
|---|---|---|---|---|
| Q11 | **Error format.** 4xx/5xx errors MUST use RFC 9457 Problem Details (`application/problem+json`), an approved decision. This spec uses `{ errors[], isSuccessful, requestId }`. Align, or document an exception? | [Solution Designer] | Open | |
| Q12 | **Error object.** `priority` is not in the guide, and errors SHOULD include a `recovery` object. Update the error schema? | [Solution Designer] | Open | |
| Q13 | **Downstream unavailable.** The error example heading says 502/503; the error table says 500; the guide specifies 503. Which status applies? | [Solution Designer] | Open | |
| Q14 | **Auth header.** This spec uses `API-Token`; the guide specifies `Authorization: Bearer`. Which applies? | [Platform team] | Open | |

## Spec tidy-up

| # | Item | Owner | Status |
|---|---|---|---|
| T1 | The request table has an "HLN-only field" legend, but no field is marked. Mark the fields or remove the legend. | [Solution Designer] | Open |
| T2 | `idealRepaymentAmount` note reads "must be the minimum". Make it "must be at or above the minimum". | [Solution Designer] | Open |
| T3 | The error `code` list includes `NOT_FOUND_ERROR` and `CONFLICT`, which this endpoint never returns. Remove them. | [Solution Designer] | Open |
| T4 | Update the overview if the endpoint also serves New loans (Q1). | [Solution Designer] | Open |