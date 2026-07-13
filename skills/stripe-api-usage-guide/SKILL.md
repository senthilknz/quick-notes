---
name: api-usage-guide
description: Generates a Stripe-style API usage guide in Markdown from
  Spring Boot REST controllers, for frontend engineers. Use when the
  user asks to document APIs, create an API guide, or generate
  endpoint documentation.
---

# API Usage Guide Generator (Stripe-style)

Generate documentation in the style of the Stripe API Reference
(https://docs.stripe.com/api): example-first, one resource per
section, every endpoint paired with a runnable request and a real
response, errors documented once and referenced everywhere.

## Process

1. Find all @RestController classes and their request mappings
2. Read the DTO classes behind each endpoint, including validation
   annotations (@NotNull, @NotBlank, @Size, @Min, @Max, @Pattern)
3. Detect error handling (@ControllerAdvice, ProblemDetail) and
   Pageable usage
4. Group endpoints by resource (Stripe style: "The Customer Offer
   object" then its endpoints), not by controller class name
5. Generate the guide using the output format below
6. Save as Markdown to docs/api-usage-guide.md

## Output format — follow this Stripe anatomy exactly

### Top of document
- Title, audience line, last-updated date
- "Authentication" section: required headers table + ONE runnable
  curl example a developer can copy first
- "Errors" section: document the error object ONCE (RFC 7807
  ProblemDetail schema with example JSON), then an HTTP status
  code table: code | meaning | how the frontend should handle it.
  Never repeat the error schema per endpoint — link back here.
- "Pagination" section: page/size/sort params and the response
  envelope, documented once

### Per resource (Stripe's core pattern)
For each resource, in this order:

1. **"The {Resource} object"** — a complete example JSON of the
   resource, followed by an attribute table:

   | Attribute | Type | Description |

   Mark child objects and expand them inline (like Stripe's
   expandable attributes). Derive types and constraints from the
   actual DTO fields.

2. **Endpoint list** — a compact index of the resource's endpoints:

   POST   /v1/customer-offers
   GET    /v1/customer-offers/{id}
   GET    /v1/customer-offers
   POST   /v1/customer-offers/{id}/acceptance

3. **One section per endpoint**, each containing:
    - One-sentence description of what it does
    - **Parameters** — each parameter with name, type, a
      required/optional badge, and constraint notes from validation
      annotations (e.g. "required, string, 3-letter ISO currency
      code"). Stripe documents constraints in prose next to the
      param, not in a separate column.
    - **Returns** — one sentence stating what comes back on success
      and which errors are possible (link to Errors section),
      e.g. "Returns the offer object if the call succeeded. Raises
      409 if an active offer already exists."
    - **Request example** — runnable curl with realistic values
    - **Response example** — the actual JSON returned, using real
      field names from the DTOs

### End of document
- "Versioning" section: current version, deprecation policy
- "Inconsistencies to fix" section: endpoints that deviate from
  the conventions (verbs in paths, non-ProblemDetail errors,
  missing pagination), each with a suggested fix

## Style rules

- Example-first: show the JSON before explaining it
- Realistic values everywhere: real field names from DTOs, plausible
  data (NZD amounts, ISO dates), never "string" or "foo"
- Every endpoint's curl must be copy-paste runnable given a $TOKEN
- Keep prose minimal — Stripe pages are mostly examples and tables
- Monetary values as {"amount": "15000.00", "currency": "NZD"}
- camelCase JSON, ISO 8601 UTC timestamps