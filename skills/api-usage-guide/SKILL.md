---
name: api-usage-guide
description: Generates a frontend-facing API usage guide in
  Confluence wiki markup from Spring Boot REST controllers.
  Use when the user asks to document APIs, create an API
  guide, or generate endpoint documentation.
---

# API Usage Guide Generator

## Process
1. Find all @RestController classes and their mappings
2. Read the DTO classes for each endpoint, including
   validation annotations
3. Detect the error handling (@ControllerAdvice,
   ProblemDetail) and Pageable usage
4. Generate the guide with these sections:
    - Overview: base URLs, auth, headers, one curl example
    - Conventions (Zalando MUST/SHOULD/MAY structure)
    - Error contract with example JSON per status code
    - Pagination/filtering/sorting
    - Endpoint reference: method, path, params, request/
      response JSON derived from real DTO fields
    - Versioning and deprecation policy
5. End with an "Inconsistencies to fix" section listing
   endpoints that deviate from the conventions
6. Output in Confluence wiki markup, save to
   docs/api-usage-guide.confluence