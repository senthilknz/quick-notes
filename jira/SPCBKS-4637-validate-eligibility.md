# SPCBKS-4637 — Story (Jira wiki markup)

Paste each block into the named Jira field. Open questions live on the Evaluate Lending Eligibility Confluence page.

---

**Summary**

```
xAPI: Validate Eligibility
```

**Description**

```
h3. User Story
*As a* customer applying for a Personal Loan (Top-Up or New)
*I want* my eligibility checked as I move through the pre-application screens
*So that* I know early whether I can proceed, and see the options available to me

h3. Summary
Implement {{POST /v2/customer-offer/personal-lending/validate-eligibility}} as the digital eligibility gate before a draft application exists. One endpoint serves both journeys, selected by {{processType}} (TopUp / New).
The xAPI calls the Eligibility API first. For an eligible Top-Up, it then calls Power Lending account detail to return the "Current setup" account summary. If the customer is not eligible, account detail is *not* called.

h3. Scope
* Endpoint, request validation and error handling as per the API contract
* Eligibility check via the cAPI Eligibility API
* Account summary via the Power Lending host API (eligible Top-Up only)
* Business outcomes (ELIGIBLE, NOT_ELIGIBLE, REFER, UNAVAILABLE) returned as 200

*Dependencies*
* cAPI Eligibility API and Power Lending host API availability
* Environment credentials and connectivity for both APIs
* Test customers covering eligible, not eligible and refer outcomes, for Top-Up and New

h3. Out of Scope
* Creating the draft application
* Customer-facing wording for reasons (owned by Westpac One)

h3. References
* [Evaluate Lending Eligibility|<confluence page link>] - API contract and open questions
* [Swagger - Customer Offer Personal Lending xAPI V2|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1119525294/Swagger+Customer+Offer+-+Personal+Lending+xAPI+V2]
* Customer, Product and Service Eligibility - Lending API (see Evaluate Lending Eligibility page, External References)
* [xAPI Specification|<confluence page link>]
* Related: SPCBKS-5032 View Account Detail (same Power Lending call)
* Definition of Done: [team DoD page]
```

**Acceptance Criteria**

```
*AC1 - Eligible Top-Up*
*Given* an authenticated customer eligible to top up their personal loan
*When* the endpoint is called with {{processType}} TopUp and a valid account number
*Then* 200 is returned with {{eligible: true}}, decision ELIGIBLE, the eligible options and the account summary, as defined by the API contract

*AC2 - Not eligible*
*Given* the Eligibility API returns a blocking outcome
*When* the endpoint is called
*Then* 200 is returned with {{eligible: false}}, decision NOT_ELIGIBLE, the reasons and no options
*And* Power Lending account detail is not called

*AC3 - Refer and unavailable*
*Given* the Eligibility API returns REFER or UNAVAILABLE
*When* the endpoint is called
*Then* 200 is returned with that decision and its reasons, as defined by the API contract

*AC4 - New loan*
*Given* an eligible customer applying for a new loan ({{processType}} New)
*When* the endpoint is called
*Then* 200 is returned with the eligibility result and no account summary
*And* Power Lending account detail is not called

*AC5 - Invalid request*
*Given* a partial residency object, an invalid loan purpose code, or a missing or invalid account number for Top-Up
*When* the endpoint is called
*Then* 400 BAD_REQUEST is returned and no downstream call is made

*AC6 - Authentication*
*Given* the JWT is missing or invalid
*When* the endpoint is called
*Then* 401 UNAUTHORIZED is returned

*AC7 - Downstream failure*
*Given* the Eligibility API fails or times out
*Or* the customer is eligible but Power Lending account detail fails or times out
*When* the endpoint is called
*Then* a service error is returned as defined by the API contract, with no eligibility result or account summary

*AC8 - Contract compliance, security & logging*
*Then* response and error payloads conform to the API contract and the xAPI Specification
*And* the Correlation-Id is returned as a response header
*And* {{crsNumber}} is taken from the JWT only
*And* account numbers, customer numbers and downstream credentials are not written to logs
*And* the Correlation-Id is logged and propagated to both downstream calls
```