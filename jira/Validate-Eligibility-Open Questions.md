# Validate Eligibility — Open Questions

Related Jira: SPCBKS-4637. Record each answer here, then update the matching acceptance criteria in the Jira.

## Flow and mapping

| # | Question | Owner | Status | Answer |
|---|---|---|---|---|
| Q1 | **REFER outcome.** REFER returns `eligible: true`. Is Power Lending account detail called for REFER, as it is for ELIGIBLE? | [Solution Designer] | Open | |
| Q2 | **Downstream mapping.** TopUp maps to `Increase`; what does New map to? Is `partyReference` the JWT `crsNumber`? | [Solution Designer] | Open | |
| Q3 | **Loan purpose and residency.** The downstream request carries neither. How do they reach the interest-rate and residency checks? | [Solution Designer] | Open | |
| Q4 | **Ownership check.** Confirm the Eligibility API checks account ownership (reason `ACCOUNT_PERSONAL_LOAN_OWNERSHIP`). This also answers the ownership question on SPCBKS-5032. | [Eligibility API team] | Open | |
| Q5 | **Overlap with View Account Detail (SPCBKS-5032).** This endpoint already returns the account summary. Is a separate `view-account-detail` still needed? xAPI Specification §2.2 prefers the xAPI to orchestrate rather than the frontend calling two xAPIs. | [BA / Product Owner] | Open | |
| Q6 | **Term units.** Downstream returns terms in Months (1–60); the xAPI examples show Years. Does the xAPI pass the unit through or convert it? | [Solution Designer] | Open | |
| Q7 | **Partial-success warning.** When the customer is eligible but account detail fails, the xAPI returns data plus a warning with recovery `USE_PARTIAL_DATA` (xAPI Specification §7.5). Agree the warning code, e.g. `ACCOUNT_SUMMARY_UNAVAILABLE`. | [Solution Designer] | Open | |

## Conformance with the xAPI Specification

| # | Question | Owner | Status | Answer |
|---|---|---|---|---|
| Q8 | **Error format.** 4xx/5xx errors MUST use RFC 9457 Problem Details (`application/problem+json`), an approved decision. This spec uses `{ errors[], isSuccessful, requestId }`. Align, or document an exception? | [Solution Designer] | Open | |
| Q9 | **Backend codes.** `decision`, reason `code` and `severity` are relayed as-is from downstream. §2.3 says backend codes must not be exposed. Map them to xAPI-owned values? | [Solution Designer] | Open | |
| Q10 | **Error object.** `priority` is not in the guide, and errors SHOULD include a `recovery` object. Update the error schema? | [Solution Designer] | Open | |
| Q11 | **Downstream unavailable.** The guide specifies 503; this spec's error table says 500. Confirm 503. | [Solution Designer] | Open | |
| Q12 | **Auth header.** This spec uses `API-Token`; the guide specifies `Authorization: Bearer`. Which applies? | [Platform team] | Open | |

## Spec tidy-up

| # | Item | Owner | Status |
|---|---|---|---|
| T1 | Rename the endpoint to `validate-eligibility` on this page and in the Swagger. | [Solution Designer] | Open |
| T2 | Add `calculationStatus` to the `EligibilityOption` table (it appears in the examples only). | [Solution Designer] | Open |
| T3 | Add the cAPI Eligibility Bruno collection (currently TODO). | [Dev] | Open |

> Q8–Q12 likely apply to the Context and View Account Detail pages too, as all three specs share the same error envelope.