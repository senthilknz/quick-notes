# SPCBKS-4983 — Story (Jira wiki markup)

Paste each block into the named Jira field. Open questions live on the Calculate Pricing Confluence page.

---

**Summary**

```
xAPI: Calculate Pricing
```

**Description**

```
h3. User Story
*As a* customer applying for a Personal Loan
*I want* to see my indicative interest rate as soon as I choose my loan purpose
*So that* I know the rate before I choose my loan amount and repayments

h3. Summary
Implement {{POST /v2/customer-offer/personal-lending/calculate-pricing}} to return the indicative pricing offer (offer rate and margin) for the customer's selected loan purpose. It is called on the Loan Purpose step, immediately after the customer selects a purpose.
The FE sends {{lendingType}} and {{loanPurpose}}. The xAPI enriches the request and calls the Pricing Orchestrator Service. This is a pure pricing lookup: no draft is created and no application state changes.

h3. Scope
* Endpoint, request validation, enrichment, downstream call and response mapping as per the API contract
* Error handling as per the API contract

*Dependencies*
* Pricing Orchestrator Service availability
* Environment credentials ({{X-API-KEY}}) and connectivity
* Test data for each supported loan purpose, including a special-rate purpose (e.g. Electric Vehicle)

h3. Out of Scope
* Home Loan pricing (the contract allows it later without a change)
* Repayment and quote calculation
* Creating or updating the application draft

h3. References
* [Calculate Pricing|<confluence page link>] - API contract and open questions
* [Swagger - Customer Offer Personal Lending xAPI V2|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1119525294/Swagger+Customer+Offer+-+Personal+Lending+xAPI+V2]
* [xAPI Specification|<confluence page link>]
* Pricing Orchestrator Service and Bruno collection (see Calculate Pricing page, External References)
* Definition of Done: [team DoD page]
```

**Acceptance Criteria**

```
*AC1 - Happy path*
*Given* an authenticated customer
*When* the endpoint is called with {{lendingType}} PersonalLoan and a supported {{loanPurpose}}
*Then* 200 is returned with the offer rate and margin as numbers, and the maximum rate when the downstream returns it, as defined by the API contract

*AC2 - Downstream request*
*When* the endpoint is called
*Then* the downstream request is built with the enriched and mapped values defined by the API contract
*And* only {{lendingType}} and {{loanPurpose}} are accepted from the client

*AC3 - Invalid request*
*Given* {{lendingType}} or {{loanPurpose}} is missing or not supported (including HomeLoan in phase 1)
*When* the endpoint is called
*Then* 400 BAD_REQUEST is returned and no downstream call is made

*AC4 - Authentication*
*Given* the JWT is missing or invalid
*When* the endpoint is called
*Then* 401 UNAUTHORIZED is returned

*AC5 - Downstream failure*
*Given* the Pricing Orchestrator Service fails, times out, or does not return a successful result
*When* the endpoint is called
*Then* a service error is returned as defined by the API contract

*AC6 - Read only*
*When* the same request is repeated
*Then* the same pricing is returned and no application state is created or changed

*AC7 - Contract compliance, security & logging*
*Then* response and error payloads conform to the API contract and the xAPI Specification
*And* the Correlation-Id is returned as a response header, logged, and propagated downstream
*And* {{crsNumber}} is taken from the JWT only
*And* the downstream {{X-API-KEY}} is never logged or returned
```