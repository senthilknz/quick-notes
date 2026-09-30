# SPCBKS-5032 — Story (Jira wiki markup)

This ticket is repurposed from View Account Detail to Calculate Loan Repayments, so replace the Summary, Description and Acceptance Criteria in full. Paste each block into the named Jira field. Open questions live on the Calculate Loan Repayments Confluence page.

---

**Summary**

```
xAPI: Calculate Loan Repayments
```

**Description**

```
h3. User Story
*As a* customer applying for a Personal Loan
*I want* my repayment amount to update as I adjust the loan amount and term
*So that* I can choose a loan I can afford before I submit my application

h3. Summary
Implement {{POST /v2/customer-offer/personal-lending/calculate-repayments}} to return an indicative repayment (repayment amount, total interest, total payable and number of payments) for a requested loan amount and term. The FE calls it as the customer moves the amount and term sliders.
The xAPI maps the FE-friendly values to the Loan Calculator Service (LCS) format and calls LCS. This is a pure calculation: no draft is created and no application state changes.

h3. Scope
* Endpoint, request validation, value mapping, downstream call and response mapping as per the API contract
* Optional ideal repayment amount (priced to it when supplied; minimum repayment returned when omitted)
* Error handling as per the API contract

*Dependencies*
* Loan Calculator Service availability
* Environment credentials ({{X-API-KEY}}) and connectivity
* Test scenarios covering Fortnightly and Monthly repayments, whole and part-year terms, and an ideal repayment amount

h3. Out of Scope
* Home Loan repayments (the contract allows it later without a change)
* Weekly repayments (not supported by LCS)
* Creating or updating the application draft

h3. References
* [Calculate Loan Repayments|<confluence page link>] - API contract and open questions
* [Swagger - Customer Offer Personal Lending xAPI V2|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1119525294/Swagger+Customer+Offer+-+Personal+Lending+xAPI+V2]
* [xAPI Specification|<confluence page link>]
* Loan Calculator Service and Bruno collection (see Calculate Loan Repayments page, External References)
* Definition of Done: [team DoD page]
```

**Acceptance Criteria**

```
*AC1 - Happy path*
*Given* an authenticated customer
*When* the endpoint is called with a valid loan amount, term, interest rate and repayment frequency
*Then* 200 is returned with the repayment amount, total interest, number of payments, total payable and term (years and months), with monetary values as numbers, as defined by the API contract

*AC2 - Ideal repayment amount*
*Given* an ideal repayment amount at or above the minimum repayment
*When* the endpoint is called
*Then* the loan is priced to that amount
*Given* no ideal repayment amount
*When* the endpoint is called
*Then* the minimum repayment is returned

*AC3 - Downstream request*
*When* the endpoint is called
*Then* the downstream request is built as defined by the API contract: FE-friendly values mapped to LCS codes, numbers sent in the format LCS expects, and xAPI-set defaults applied

*AC4 - Invalid request*
*Given* a missing or invalid amount, term or rate, an unsupported repayment frequency (e.g. Weekly), or an ideal repayment amount below the minimum
*When* the endpoint is called
*Then* 400 BAD_REQUEST is returned

*AC5 - Authentication*
*Given* the JWT is missing or invalid
*When* the endpoint is called
*Then* 401 UNAUTHORIZED is returned

*AC6 - Downstream failure*
*Given* the Loan Calculator Service fails or times out
*When* the endpoint is called
*Then* a service error is returned as defined by the API contract

*AC7 - Read only*
*When* the same request is repeated
*Then* the same result is returned and no application state is created or changed

*AC8 - Contract compliance, security & logging*
*Then* response and error payloads conform to the API contract and the xAPI Specification
*And* the Correlation-Id is returned as a response header, logged, and propagated downstream
*And* {{crsNumber}} is taken from the JWT only
*And* loan amounts, balances and the downstream {{X-API-KEY}} are not written to logs
```