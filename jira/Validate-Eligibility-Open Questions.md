# Validate Eligibility — Open Questions

Related Jira: SPCBKS-4637. Record each answer here, then update the matching acceptance criteria in the Jira.

## Decisions

| # | Decision |
|---|---|
| D1 | If the customer is eligible but Power Lending account detail fails, the xAPI returns an error (not partial data). The Top-Up journey cannot proceed without the account summary, so this is a blocking failure under xAPI Specification §7.3. |

## Flow and mapping

| # | Question | Owner | Status | Answer |
|---|---|---|---|---|
| Q1 | **Downstream mapping.** TopUp maps to `Increase`; what does New map to? Is `partyReference` the JWT `crsNumber`? | [Solution Designer] | Open | |
| Q2 | **Loan purpose and residency.** The downstream request carries neither. How do they reach the interest-rate and residency checks? | [Solution Designer] | Open | |
| Q3 | **Term units.** Downstream returns terms in Months (1–60); the xAPI examples show Years. Does the xAPI pass the unit through or convert it? | [Solution Designer] | Open | |

## Conformance with the xAPI Specification

| # | Question | Owner | Status | Answer |
|---|---|---|---|---|
| Q4 | **Error format.** 4xx/5xx errors MUST use RFC 9457 Problem Details (`application/problem+json`), an approved decision. This spec uses `{ errors[], isSuccessful, requestId }`. Align, or document an exception? | [Solution Designer] | Open | |
| Q5 | **Backend codes.** `decision`, reason `code` and `severity` are relayed as-is from downstream. §2.3 says backend codes must not be exposed. Map them to xAPI-owned values? | [Solution Designer] | Open | |
| Q6 | **Error object.** `priority` is not in the guide, and errors SHOULD include a `recovery` object. Update the error schema? | [Solution Designer] | Open | |
| Q7 | **Downstream unavailable.** The guide specifies 503; this spec's error table says 500. Confirm 503. | [Solution Designer] | Open | |
| Q8 | **Auth header.** This spec uses `API-Token`; the guide specifies `Authorization: Bearer`. Which applies? | [Platform team] | Open | |

## Spec tidy-up

| # | Item | Owner | Status |
|---|---|---|---|
| T1 | Rename the endpoint to `validate-eligibility` on this page and in the Swagger. | [Solution Designer] | Open |
| T2 | Add `calculationStatus` to the `EligibilityOption` table (it appears in the examples only). | [Solution Designer] | Open |
| T3 | Add the cAPI Eligibility Bruno collection (currently TODO). | [Dev] | Open |
| T4 | Document decision D1 in the Errors section. | [Solution Designer] | Open |

> Q4–Q8 likely apply to the Context page too, as the specs share the same error envelope.