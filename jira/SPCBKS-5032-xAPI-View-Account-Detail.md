# SPCBKS-5032 — xAPI: View Account Detail

> The **contract** (request, response, field mapping, error codes) lives on the [View Account Detail design page](https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1132545778/View+Account+Detail) — refined against page version `[vX]`. This story holds the scope, open questions and acceptance criteria only; section numbers below refer to that page.

---

## Description

### User Story
- **As a** personal loan customer starting a Top-Up
- **I want** to see my existing loan's current setup (balance, repayment and rate)
- **So that** I understand my current position before choosing how much to top up

### Summary
Implement `POST /v2/customer-offer/personal-lending/view-account-detail` to return the "Current setup" figures for the **Top-up details** screen, proxying the Power Lending host API. Top-Up journey only; read only (the FE replays the figures into `create-application`).

### Out of Scope
- New loan (origination) journey
- Downstream fields beyond those in the spec's response section
- Creating or updating the application draft

### Open Questions **[OPEN]**
1. **Ownership check** — how is `accountNumber` verified against the JWT `crsNumber`? If not owned: `403 FORBIDDEN` or `404 NOT_FOUND_ERROR`? (404 avoids account enumeration.)
2. **5xx status** — the error table maps downstream failures to `500`, but the example heading says "502/503". Which applies?
3. **Unknown frequency codes** — downstream code other than `FN`/`WK`/`MN`: `null`, or error?
4. **Balance sign** — confirm a balance owing is returned as a positive amount.
5. **Non-loan account** — a valid account that isn't a Personal Loan: `404` or `400`?
6. **Partial data** — should a `null` figure also add a `warnings[]` entry?
7. **Tracing on success** — add `requestId` / `correlationId` to `meta` (currently only on errors)?
8. **Downstream timeout** — agree the value and whether any retry applies.

*Resolve in the Confluence page, then update AC7 / AC9 here.*

### Dependencies
- Power Lending host API available in `[SIT/UAT]`
- Test data: Top-Up-eligible loans on each frequency (FN, WK, MN), plus a closed loan, a non-loan account, and another customer's loan

### Test Notes
- Bruno collection `power-lending.zip` is attached to the design page (External References).
- Use test-environment account numbers only.

### Reference
- [View Account Detail design](https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1132545778/View+Account+Detail) — contract of record
- [Swagger — Customer Offer Personal Lending xAPI V2](https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1119525294/Swagger+Customer+Offer+-+Personal+Lending+xAPI+V2)

---

## Acceptance Criteria

**AC1 – Happy path**
- **Given** an authenticated customer with a Top-Up-eligible personal loan
- **When** the endpoint is called with a valid `accountNumber`
- **Then** `200` is returned with `data` populated as per spec §4.1

**AC2 – Downstream request**
- The downstream request is built as per spec §3 (dashes stripped, wrapped in `Data`)
- `crsNumber` comes from the JWT only; any client-supplied value is ignored

**AC3 – Response mapping**
- Each field is mapped as per spec §4.1.1, including frequency code translation and balance sign
- No downstream fields beyond §4.1.1 appear in the response

**AC4 – Partial data**
- **Given** downstream omits any figure
- **Then** `200` is returned with that field `null` and the others populated

**AC5 – Invalid request**
- **Given** `accountNumber` is missing or in the wrong format
- **Then** `400 BAD_REQUEST` is returned and no downstream call is made

**AC6 – Authentication**
- **Given** the JWT is missing or invalid
- **Then** `401 UNAUTHORIZED` is returned

**AC7 – Account not owned by caller** *(pending Open Question 1)*
- **Then** `[403 | 404]` is returned and no loan data is exposed

**AC8 – Account not found**
- **Then** `404 NOT_FOUND_ERROR` is returned

**AC9 – Downstream failure** *(pending Open Question 2)*
- **Given** Power Lending times out or returns 5xx
- **Then** `[500 | 502/503] DOWNSTREAM_ERROR` is returned

**AC10 – Error envelope**
- All errors follow spec §4.2, and `requestId` equals the inbound `Correlation-Id`

**AC11 – Read only / idempotent**
- Repeating the same request returns the same data and creates or changes no draft

**AC12 – Logging & security**
- Account number and monetary values are not logged
- `Correlation-Id` is logged and propagated downstream

---

## Definition of Done
- [ ] All acceptance criteria met and demoed to the tester / BA
- [ ] Open Questions resolved on the Confluence page; AC7 / AC9 updated
- [ ] Code reviewed and merged to main; pipeline green
- [ ] Unit tests written; coverage meets team threshold
- [ ] Integration tests cover AC1–AC12 with Power Lending stubbed (e.g. WireMock)
- [ ] Contract tests pass against the published OpenAPI spec
- [ ] Confluence / Swagger updated if the contract changed
- [ ] Security checks pass (SAST / dependency scan); no PII in logs
- [ ] Splunk queries verified for the new endpoint
- [ ] Deployed to `[SIT/UAT]` and smoke tested
- [ ] Test cases linked to this story and passed by QA
- [ ] No open Critical / High defects