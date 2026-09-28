# SPCBKS-4990 — Story + Sub-tasks (Jira wiki markup)

Each block below is ready to paste into the named Jira field (Data Center renders wiki markup, not Markdown).

- **Story** — the context endpoint and the extensible framework, sourced from **local config** for now.
- **Sub-task 1** — `LOAN_PURPOSES` group.
- **Sub-task 2** — `REPAYMENT_FREQUENCIES` group.
- Each future config type = one more sub-task (or story) on the same pattern. Switching to the downstream API = a separate follow-up story (template at the end).

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
Implement {{POST /v2/customer-offer/personal-lending/context}} as the Personal Lending experience bootstrap, supporting *Selected* ({{configurationTypes}}) and *All supported* ({{includeAllConfiguration: true}}) modes, scoped by {{productType}} + {{processType}}.

*Phase 1 (this story):* configuration is served from *local config* in the xAPI. Switching to the WNZL Lending API configuration capability is a separate follow-up story once that API is available. The contract does not change when the source is switched.

h3. Scope
* Endpoint, request validation, response envelope and error handling
* Extensible configuration-group framework (see Technical Notes)
* Initial groups, delivered as sub-tasks:
** LOAN_PURPOSES
** REPAYMENT_FREQUENCIES

h3. Out of Scope
* Calling the downstream configuration API (follow-up story)
* Candidate groups under review (living arrangements, employment types, etc.). Each is added later as its own sub-task/story

h3. Technical Notes
* Each configuration type is implemented as its own group provider behind a common interface.
* "All supported" mode returns every *registered* group for the product + process.
* The config *source* sits behind its own abstraction (local config now, downstream API later), so switching source does not touch the endpoint, validation or mapping.
* Adding a new configuration type = new {{PersonalLendingConfigurationType}} value + new optional property on {{configuration}} + new provider. This is an additive, non-breaking change.
* Local config is validated at startup; the service fails fast on invalid config.

h3. Open Questions
Answer each in the Confluence contract page, record the decision here, then update the linked AC. The story can't close while any row is Open.
||#||Question||Owner||Status||Affects||Answer / Decision||
|Q1|Supported {{productType}} + {{processType}} combinations (needed for the "unsupported combination" 400)|[BA / Product Owner]|Open|AC2, AC5| |
|Q2|Does configuration differ between TopUp and New?|[BA / Product Owner]|Open|Sub-tasks 1 & 2| |
|Q3|Who signs off the local config values, and how are changes requested before the downstream API is live? (Each change needs a deploy)|[Product Owner]|Open|DoD| |
|Q4|Is {{code}} returned on repayment frequencies? The mapping table defines LCS codes, but the response examples omit it.|[Solution Designer]|Open|Sub-task 2| |
|Q5|Weekly is not offered, but View Account Detail (SPCBKS-5032) can return Weekly for the existing loan. Is that intended for Top-Up?|[BA / Product Owner]|Open|Sub-task 2, SPCBKS-5032| |

h3. Reference
* [Context - PL experience Bootstrap And Configuration|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1129880395/Context+-+PL+experience+Bootstrap+And+Configuration] - contract of record, version [vX]
* [Swagger - Customer Offer Personal Lending xAPI V2|https://confluence.westpac.co.nz/spaces/BAPCBKS/pages/1119525294/Swagger+Customer+Offer+-+Personal+Lending+xAPI+V2]

h3. Definition of Done
* All acceptance criteria met and demoed to the tester / BA
* All sub-tasks (configuration groups) complete
* Local config values signed off by the business owner
* Open questions resolved on the Confluence contract page; affected ACs updated
* Code reviewed and merged to main; pipeline green
* Unit tests written for new logic; coverage meets team threshold
* Integration tests cover all ACs
* Contract tests pass against the published OpenAPI spec
* Confluence contract page / Swagger updated if the contract changed
* Security scans pass (SAST / dependency scan); no PII or credentials in logs
* Logging checked in Splunk: Correlation-Id present
* Deployed to SYST and smoke tested
* Test cases linked to this story and passed by QA
* No open Critical / High defects
```

**Acceptance Criteria**

```
*AC1 - Selected mode*
*Given* a valid JWT
*When* called with {{configurationTypes}} listing one or more supported groups
*Then* 200 is returned and {{configuration}} contains exactly the requested groups

*AC2 - All-supported mode*
*When* called with {{includeAllConfiguration: true}}
*Then* 200 is returned with every group supported for the product + process

*AC3 - Unrequested groups omitted*
*Then* groups that were not requested are absent from {{configuration}} (not returned as null)

*AC4 - Echo*
*Then* {{data.productType}} and {{data.processType}} echo the request

*AC5 - Invalid request* _(supported combinations pending Q1)_
*Then* 400 BAD_REQUEST is returned when:
* both modes are supplied
* neither mode is supplied (including {{includeAllConfiguration: false}} with no types)
* {{configurationTypes}} is empty, has duplicates, or has an unknown value
* the product + process combination is unsupported

*AC6 - Authentication*
*Given* the JWT is missing or invalid
*Then* 401 UNAUTHORIZED is returned

*AC7 - Internal failure*
*Given* an unexpected internal error
*Then* 500 UNEXPECTED_ERROR is returned

*AC8 - Error envelope*
*Then* every error is returned as {{\{ errors[], isSuccessful: false, requestId \}}} with no {{data}} key, and {{requestId}} equals the inbound Correlation-Id

*AC9 - Security & logging*
*Then* {{crsNumber}} is taken from the JWT only, and Correlation-Id is logged

*AC10 - Groups delivered*
*Then* all sub-tasks (initial configuration groups) are complete and their groups are returned by this endpoint
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

h3. Details
* Returned as {{configuration.loanPurposes}} ({{LoanPurposeConfiguration}}) per the contract: {{effectiveDate}}, ordered {{categories[]}}, {{purposesByCategory}}
* Values, labels and codes as per the Context page (Loan-purposes response example), pending business sign-off

h3. Acceptance Criteria
* Requesting {{LOAN_PURPOSES}} returns {{configuration.loanPurposes}}, and it is included in "All supported" mode
* {{categories[]}} are returned in the configured display order
* Every {{purposesByCategory}} key matches a {{categories[].value}}; the service fails at startup if not
* Each purpose has {{value}}, {{label}} and {{code}}; {{code}} is the value echoed to evaluate-eligibility as {{loanPurposeCode}}
* An empty configured group returns {{categories: []}} and {{purposesByCategory: \{\}}}
* Unit and integration tests cover the above
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

h3. Details
* Returned as {{configuration.repaymentFrequencies}} (ordered {{ReferenceValue[]}}) per the contract
* Initial values: Fortnightly (LCS {{FORTNIGHTLY}}), Monthly (LCS {{MONTHLY}})

h3. Open Questions
See Q4 and Q5 in the parent story (SPCBKS-4990).

h3. Acceptance Criteria
* Requesting {{REPAYMENT_FREQUENCIES}} returns {{configuration.repaymentFrequencies}}, and it is included in "All supported" mode
* Values are returned in the configured display order, each with {{value}} and {{label}} ({{code}} pending parent story Q4)
* Unit and integration tests cover the above
```

---

## TEMPLATE — future config type (sub-task)

```
Summary: Context config group: <CONFIG_TYPE> (local config)

h3. Goal
Add the {{<CONFIG_TYPE>}} configuration group to the Context endpoint.

h3. Contract change (additive)
* New {{PersonalLendingConfigurationType}} value: {{<CONFIG_TYPE>}}
* New optional property: {{configuration.<fieldName>}} ({{<Shape>}})
* Confluence contract page and Swagger updated first

h3. Acceptance Criteria
* Requesting {{<CONFIG_TYPE>}} returns {{configuration.<fieldName>}}, and it is included in "All supported" mode
* Existing groups and consumers are unaffected
* Values returned in configured display order
* Unit and integration tests cover the above
```

---

## FOLLOW-UP STORY — switch to downstream API (create when the API is ready)

```
Summary: xAPI: Personal Lending Context - source configuration from WNZL Lending API

h3. Summary
Replace the local config source for the Context endpoint with the WNZL Lending API configuration / reference-data capability. The xAPI contract does not change.

h3. Acceptance Criteria
* All registered groups are sourced from the downstream API; responses match the contract as before
* Downstream failure or timeout returns 500 DOWNSTREAM_ERROR
* Behaviour agreed and implemented for partial success (one group fails, another succeeds)
* Caching behaviour agreed and implemented (e.g. using effectiveDate)
* Local config removed, or kept as an agreed fallback
* Contract tests pass unchanged against the existing OpenAPI spec
```