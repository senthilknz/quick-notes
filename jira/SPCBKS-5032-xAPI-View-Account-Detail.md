# SPCBKS-5032 — Story (Jira wiki markup)

Paste each block into the named Jira field. Open questions live on the View Account Detail Confluence page.

---

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
Implement {{POST /v2/customer-offer/personal-lending/view-account-detail}} to return the "Current setup" figures for the *Top-up details* screen, sourced from the Power Lending host API. Top-Up journey only; read only.

h3. Scope
* Endpoint, request validation, downstream call and response mapping
* Error handling as per the API contract

*Dependencies*
* Power Lending host API availability
* Environment credentials and connectivity
* Test accounts available for validation

h3. Out of Scope
* New loan (origination) journey
* Creating or updating the application draft

h3. References
* [View Account Detail|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1132545778/View+Account+Detail] - API contract and open questions
* [Swagger - Customer Offer Personal Lending xAPI V2|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1119525294/Swagger+Customer+Offer+-+Personal+Lending+xAPI+V2]
* Definition of Done: [team DoD page]
```

**Acceptance Criteria**

```
*AC1 - Happy path*
*Given* an authenticated customer with a Top-Up-eligible personal loan
*When* the endpoint is called with a valid account number
*Then* the loan details are returned as defined by the API contract

*AC2 - Partial data*
*Given* some loan attributes are unavailable downstream
*When* the endpoint is called
*Then* the available data is returned and missing values are handled as defined by the API contract

*AC3 - Invalid request*
*Given* the account number is missing or invalid
*When* the endpoint is called
*Then* 400 BAD_REQUEST is returned and no downstream call is made

*AC4 - Authentication*
*Given* the JWT is missing or invalid
*When* the endpoint is called
*Then* 401 UNAUTHORIZED is returned

*AC5 - Account not found or not owned*
*Given* the account does not exist, or does not belong to the authenticated customer
*When* the endpoint is called
*Then* 404 NOT_FOUND_ERROR is returned and no loan data is exposed _(ownership response pending design confirmation)_

*AC6 - Downstream / internal error*
*Given* a downstream or unexpected internal failure
*When* the endpoint is called
*Then* the error response defined by the API contract is returned

*AC7 - Contract compliance*
*Then* response and error payloads conform to the agreed xAPI contract, and requestId on errors equals the inbound Correlation-Id

*AC8 - Security & logging*
*Then* account numbers, monetary values and downstream credentials are not written to logs
*And* the Correlation-Id is logged and propagated to the downstream call
```