# SPCBKS-4990 — Story + Sub-tasks (Jira wiki markup)

Paste each block into the named Jira field. Open questions and design notes live on the Context Confluence page.

---

## STORY — SPCBKS-4990

**Summary**

```
xAPI: Personal Lending Context - Dynamic Form Configuration
```

**Description**

```
h3. User Story
*As a* Personal Lending channel (Westpac One)
*I want* to fetch the journey configuration at the start of the Personal Lending journey
*So that* pickers such as loan purpose and repayment frequency are driven by configuration, not hard-coded in the FE

h3. Summary
Implement {{POST /v2/customer-offer/personal-lending/context}} to return the configuration groups a channel requests, in *Selected* ({{configurationTypes}}) or *All supported* ({{includeAllConfiguration: true}}) mode.
Phase 1 serves configuration from *local config*. Switching to the downstream configuration API is a separate follow-up story, with no contract change.

h3. Scope
* Endpoint, request validation and error handling as per the API contract
* Initial configuration groups, delivered as sub-tasks:
** LOAN_PURPOSES
** REPAYMENT_FREQUENCIES
* New configuration types are added later as additive sub-tasks

*Dependencies*
* Local config values signed off by the business owner

h3. Out of Scope
* Calling the downstream configuration API (follow-up story)
* Configuration groups beyond the two above

h3. References
* [Context - PL experience Bootstrap And Configuration|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1129880395/Context+-+PL+experience+Bootstrap+And+Configuration] - API contract, design notes and open questions
* [Swagger - Customer Offer Personal Lending xAPI V2|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1119525294/Swagger+Customer+Offer+-+Personal+Lending+xAPI+V2]
* Definition of Done: [team DoD page]
```

**Acceptance Criteria**

```
*AC1 - Selected mode*
*Given* a valid JWT
*When* the endpoint is called with {{configurationTypes}} listing one or more supported groups
*Then* 200 is returned with exactly the requested groups

*AC2 - All-supported mode*
*When* the endpoint is called with {{includeAllConfiguration: true}}
*Then* 200 is returned with every group supported for the product and process

*AC3 - Unrequested groups omitted*
*Then* groups that were not requested are absent from the response (not returned as null)

*AC4 - Invalid request*
*Given* both modes, neither mode, an empty / duplicate / unknown configuration type, or an unsupported product + process combination
*When* the endpoint is called
*Then* 400 BAD_REQUEST is returned

*AC5 - Authentication*
*Given* the JWT is missing or invalid
*When* the endpoint is called
*Then* 401 UNAUTHORIZED is returned

*AC6 - Internal error*
*Given* an unexpected internal failure
*When* the endpoint is called
*Then* 500 UNEXPECTED_ERROR is returned

*AC7 - Contract compliance*
*Then* response and error payloads conform to the agreed xAPI contract, including {{productType}} and {{processType}} echoed from the request, and requestId on errors equal to the inbound Correlation-Id

*AC8 - Security & logging*
*Then* no customer identifiers are accepted in the URL or body ({{crsNumber}} comes from the JWT only)
*And* the Correlation-Id is logged
```

---

## SUB-TASK 1 — Loan Purposes

**Summary**

```
Context config group: LOAN_PURPOSES (local config)
```

**Description**

```
h3. Goal
Add the {{LOAN_PURPOSES}} configuration group to the Context endpoint, sourced from local config.

h3. Acceptance Criteria
* Requesting {{LOAN_PURPOSES}} returns {{configuration.loanPurposes}} as defined by the API contract, and it is included in "All supported" mode
* Categories and purposes are returned in the configured display order
* Every purpose category matches a listed category
* Each purpose has {{value}}, {{label}} and {{code}}
* An empty configured group returns empty {{categories}} and {{purposesByCategory}}
```

---

## SUB-TASK 2 — Repayment Frequencies

**Summary**

```
Context config group: REPAYMENT_FREQUENCIES (local config)
```

**Description**

```
h3. Goal
Add the {{REPAYMENT_FREQUENCIES}} configuration group to the Context endpoint, sourced from local config.

h3. Acceptance Criteria
* Requesting {{REPAYMENT_FREQUENCIES}} returns {{configuration.repaymentFrequencies}} as defined by the API contract, and it is included in "All supported" mode
* Values are returned in the configured display order, each with {{value}} and {{label}}
```

---

## TEMPLATE — future config type (sub-task)

```
Summary: Context config group: <CONFIG_TYPE> (local config)

h3. Goal
Add the {{<CONFIG_TYPE>}} configuration group to the Context endpoint.

h3. Acceptance Criteria
* Requesting {{<CONFIG_TYPE>}} returns {{configuration.<fieldName>}} as defined by the API contract, and it is included in "All supported" mode
* Values are returned in the configured display order
* Existing groups and consumers are unaffected
```

---

## FOLLOW-UP STORY — switch to downstream API (create when the API is ready)

```
Summary: xAPI: Personal Lending Context - source configuration from downstream API

h3. Summary
Replace the local config source for the Context endpoint with the downstream configuration API. The xAPI contract does not change.

h3. Acceptance Criteria
* All configuration groups are sourced from the downstream API, and responses match the contract as before
* A downstream failure or timeout returns the error defined by the API contract
* Partial-success and caching behaviour implemented as agreed on the design page
* Existing contract tests pass unchanged
```