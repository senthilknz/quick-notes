# SPCBKS-5032 — Story (Jira wiki markup)

Each block below is ready to paste into the named Jira field (Data Center renders wiki markup, not Markdown).

---

## STORY — SPCBKS-5032

**Summary**

```
xAPI: View Account Detail
```

**Description**

```
h3. User Story
*As a* personal loan customer starting a Top-Up
*I want* to see my existing loan's current setup (balance, repayment and rate)
*So that* I understand my current position before choosing how much to top up

h3. Summary
Implement {{POST /v2/customer-offer/personal-lending/view-account-detail}} to return the "Current setup" figures for the *Top-up details* screen, proxying the Power Lending host API. Top-Up journey only. Read only: the FE replays the figures into {{create-application}}.

h3. Scope
* Endpoint, request validation, downstream call and response mapping
* Error handling and response envelopes as per the contract

h3. Out of Scope
* New loan (origination) journey
* Downstream fields beyond those in the contract's response section
* Creating or updating the application draft

h3. Open Questions
Answer each in the Confluence contract page, record the decision here, then update the linked AC. The story can't close while any row is Open.
||#||Question||Owner||Status||Affects||Answer / Decision||
|Q1|*Ownership check* - proposed: allow only if the JWT {{crsNumber}} is in the downstream {{customers[]}} list (joint loans list several). If not owned: 403 FORBIDDEN or 404 NOT_FOUND_ERROR? (404 avoids account enumeration)|[Solution Designer]|Open|AC7| |
|Q2|*AccountDetails array* - downstream returns {{Data.AccountDetails[]}}. Behaviour when it is empty (404?) or has more than one entry?|[Solution Designer]|Open|AC8| |
|Q3|*Account type* - which {{accountType}} values are Personal Loans? Needed to reject non-loan accounts|[Power Lending team]|Open|AC8| |
|Q4|*Downstream failure status* - the error table maps downstream failures to 500, but the example heading says 502/503. Which applies?|[Solution Designer]|Open|AC9, AC13| |
|Q5|*Unknown frequency codes* - downstream code other than FN / WK / MN: null, or error?|[Solution Designer]|Open|AC3| |
|Q6|*Weekly* - this endpoint can return Weekly, but the Context repayment-frequency picker (SPCBKS-4990) offers only Fortnightly and Monthly. How does Top-Up handle a weekly loan?|[BA / Product Owner]|Open|SPCBKS-4990| |
|Q7|*Balance sign* - SYST shows {{onlineBalance}} positive with {{onlineBalanceCredit: false}} for a balance owing. Confirm the xAPI returns it as a positive amount|[BA]|Open|AC3| |
|Q8|*Non-loan account* - a valid account that isn't a Personal Loan: 404 or 400?|[Solution Designer]|Open|AC8| |
|Q9|*Downstream timeout* - agree the value and whether any retry applies|[Dev Lead]|Open|AC9| |
|Q10|*XSRF token lifetime* - is the token static per environment, or does it expire / need refreshing? If it expires, how does the xAPI obtain a new one?|[Power Lending team]|Open|AC13| |
|Q11|*Credential rotation* - who owns the API key and token, and how are rotations communicated?|[Power Lending team]|Open|AC13| |

h3. Dependencies
* Downstream: Power Lending host API {{POST /powerlending/host/v2/account-detail}} (CIF Data API v1.0.1)
** SYST: {{https://ingress-lending.syst.k8stest.cloud.westpac.co.nz}} - verified returning 200
** Contract and Bruno collection ({{power-lending.zip}}): see View Account Detail page, External References
** Required headers: {{x-api-key}}, {{x-xsrf-token}}, {{x-correlation-id}}, {{content-type}} / {{accept: application/json}}
* Credentials: API key and XSRF token to be provided by the Power Lending team for each environment (SYST, UAT, Prod), and configured as secrets - not in code or config files
* Test data: Personal Loan accounts on each frequency (FN, WK, MN), plus a closed loan, a non-loan account, a joint loan and another customer's loan

h3. Reference
* [View Account Detail|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1132545778/View+Account+Detail] - contract of record, version [vX]
* [Swagger - Customer Offer Personal Lending xAPI V2|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1119525294/Swagger+Customer+Offer+-+Personal+Lending+xAPI+V2]
* Bruno collection: power-lending.zip (attached to the View Account Detail page)

h3. Definition of Done
* All acceptance criteria met and demoed to the tester / BA
* Open questions resolved on the Confluence contract page; affected ACs updated
* Code reviewed and merged to main; pipeline green
* Unit tests written for new logic; coverage meets team threshold
* Integration tests cover all ACs, with the downstream stubbed (e.g. WireMock)
* Contract tests pass against the published OpenAPI spec
* Confluence contract page / Swagger updated if the contract changed
* Security scans pass (SAST / dependency scan); no PII or credentials in logs
* Logging checked in Splunk: Correlation-Id present, no sensitive values
* Deployed to SYST and smoke tested
* Test cases linked to this story and passed by QA
* No open Critical / High defects
```

**Acceptance Criteria**

```
*AC1 - Happy path*
*Given* an authenticated customer with a Top-Up-eligible personal loan
*When* the endpoint is called with a valid {{accountNumber}}
*Then* 200 is returned with {{data}} populated as per the contract's response example

*AC2 - Downstream request*
*Then* the downstream request is built as per the contract's Request section (dashes stripped, wrapped in {{Data}})
*And* {{crsNumber}} is taken from the JWT only; any client-supplied value is ignored

*AC3 - Response mapping*
*Then* each field is mapped as per the contract's {{AccountDetailResult}} table, including frequency code translation and balance sign
*And* no other downstream fields appear in the response

*AC4 - Partial data*
*Given* downstream omits any of the figures
*Then* 200 is returned with that field null and the others populated

*AC5 - Invalid request*
*Given* {{accountNumber}} is missing or not in NN-NNNN-NNNNNNN-NNN format
*Then* 400 BAD_REQUEST is returned and no downstream call is made

*AC6 - Authentication*
*Given* the JWT is missing or invalid
*Then* 401 UNAUTHORIZED is returned

*AC7 - Account not owned by caller* _(pending Q1)_
*Given* the {{accountNumber}} does not belong to the caller's {{crsNumber}}
*Then* [403 FORBIDDEN | 404 NOT_FOUND_ERROR] is returned and no loan data is exposed

*AC8 - Account not found* _(empty-array and non-loan cases pending Q2, Q3, Q8)_
*Given* no account exists for the supplied {{accountNumber}}
*Then* 404 NOT_FOUND_ERROR is returned

*AC9 - Downstream failure* _(pending Q4, Q9)_
*Given* Power Lending times out or returns 5xx
*Then* [500 | 502/503] DOWNSTREAM_ERROR is returned
*Given* an unexpected internal error
*Then* 500 UNEXPECTED_ERROR is returned

*AC10 - Error envelope*
*Then* every error is returned as {{\{ errors[], isSuccessful: false, requestId \}}} with no {{data}} key, and {{requestId}} equals the inbound Correlation-Id

*AC11 - Read only / idempotent*
*When* the same request is repeated
*Then* the same data is returned and no application draft is created or changed

*AC12 - Security & logging*
*Then* the account number and monetary values are not written to logs
*And* Correlation-Id is logged and propagated to the downstream call as {{x-correlation-id}}

*AC13 - Downstream authentication*
*Then* every downstream call sends {{x-api-key}} and {{x-xsrf-token}}, read from the environment's secret configuration
*And* the key and token never appear in logs, error responses or source control
*Given* the downstream rejects the credentials (401/403)
*Then* the xAPI returns 500 DOWNSTREAM_ERROR (not 401), since the failure is not the caller's
```