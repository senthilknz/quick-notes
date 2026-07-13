# Customer Offer API — Usage Guide

> Audience: frontend engineers consuming the Customer Offer API.
> Generated from `xapi-wone-customer-offer` controllers. Last updated: 2026-07-14.

---

## 1. Overview

| Environment | Base URL |
|---|---|
| DEV | `https://api-dev.example.co.nz/customer-offer/v1` |
| SIT | `https://api-sit.example.co.nz/customer-offer/v1` |
| PROD | `https://api.example.co.nz/customer-offer/v1` |

### Authentication

All requests require a Bearer token obtained from the identity provider.

Required headers on every request:

| Header | Value | Notes |
|---|---|---|
| `Authorization` | `Bearer <token>` | OAuth2 access token |
| `X-Correlation-Id` | UUID | Generate per request; echoed in responses and logs |
| `Content-Type` | `application/json` | For POST/PUT/PATCH |

### Quick start

```bash
curl -X GET \
  "https://api-dev.example.co.nz/customer-offer/v1/customer-offers?customerId=C-10042" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Correlation-Id: $(uuidgen)"
```

---

## 2. Conventions

These follow the Zalando RESTful API Guidelines (MUST / SHOULD / MAY).

- **MUST** — Resource paths are plural, kebab-case nouns: `/customer-offers`, `/offer-documents`. No verbs in paths.
- **MUST** — JSON property names are camelCase.
- **MUST** — All timestamps are ISO 8601 UTC: `2026-07-14T02:15:00Z`.
- **MUST** — Monetary amounts are objects: `{ "amount": "1500.00", "currency": "NZD" }` — amount is a string to avoid float precision issues.
- **SHOULD** — Clients send `X-Correlation-Id`; if absent the server generates one.
- **MAY** — Clients use `fields` query parameter for sparse responses where supported.

### HTTP methods

| Method | Usage | Success code |
|---|---|---|
| GET | Read a resource or collection | 200 |
| POST | Create a resource | 201 + `Location` header |
| PUT | Full replace | 200 |
| PATCH | Partial update (JSON Merge Patch) | 200 |
| DELETE | Remove | 204 |

---

## 3. Error contract

All errors return **RFC 7807 Problem Details** (`Content-Type: application/problem+json`). The shape is always:

```json
{
  "type": "https://api.example.co.nz/problems/validation-error",
  "title": "Validation failed",
  "status": 400,
  "detail": "offerAmount must be greater than 0",
  "instance": "/customer-offers",
  "correlationId": "7f3c2a10-9b1e-4c5d-8a2f-1e6d4b9c0a33",
  "errors": [
    { "field": "offerAmount", "message": "must be greater than 0" }
  ]
}
```

| Status | When | Frontend handling |
|---|---|---|
| 400 | Validation failure — see `errors[]` | Map `errors[].field` to form fields |
| 401 | Missing/expired token | Trigger token refresh, retry once |
| 403 | Authenticated but not permitted | Show access-denied state, do not retry |
| 404 | Resource does not exist | Show not-found state |
| 409 | Conflict (e.g. offer already accepted) | Refetch resource, show current state |
| 500 | Server error | Show generic error with `correlationId` for support |

**Always surface `correlationId` in error reporting** — it is the key support uses to find the request in logs.

---

## 4. Pagination, filtering, sorting

Collection endpoints use standard Spring pagination parameters:

| Param | Example | Default |
|---|---|---|
| `page` | `page=0` | 0 (zero-based) |
| `size` | `size=20` | 20, max 100 |
| `sort` | `sort=createdDate,desc` | endpoint-specific |

Paged responses have this envelope:

```json
{
  "content": [ ... ],
  "page": {
    "number": 0,
    "size": 20,
    "totalElements": 134,
    "totalPages": 7
  }
}
```

---

## 5. Endpoint reference

### 5.1 List customer offers

`GET /customer-offers`

Query parameters:

| Param | Type | Required | Description |
|---|---|---|---|
| `customerId` | string | yes | Customer identifier |
| `status` | enum | no | `DRAFT`, `PRESENTED`, `ACCEPTED`, `EXPIRED` |
| `page`, `size`, `sort` | — | no | See section 4 |

Response `200`:

```json
{
  "content": [
    {
      "offerId": "OFF-2026-000123",
      "customerId": "C-10042",
      "productCode": "PERSONAL_LOAN",
      "status": "PRESENTED",
      "offerAmount": { "amount": "15000.00", "currency": "NZD" },
      "interestRate": "12.95",
      "expiryDate": "2026-08-14",
      "createdDate": "2026-07-14T02:15:00Z"
    }
  ],
  "page": { "number": 0, "size": 20, "totalElements": 1, "totalPages": 1 }
}
```

---

### 5.2 Get a single offer

`GET /customer-offers/{offerId}`

| Path param | Type | Description |
|---|---|---|
| `offerId` | string | Offer identifier, format `OFF-YYYY-NNNNNN` |

Responses: `200` (body as above, single object), `404` if unknown.

---

### 5.3 Create an offer

`POST /customer-offers`

Request body (validation shown from DTO constraints):

```json
{
  "customerId": "C-10042",          // @NotBlank
  "productCode": "PERSONAL_LOAN",   // @NotNull, enum
  "offerAmount": {                   // @NotNull @Valid
    "amount": "15000.00",            // @Positive
    "currency": "NZD"                // @Pattern("[A-Z]{3}")
  },
  "termMonths": 36                   // @Min(6) @Max(84)
}
```

Responses:

- `201` — created; `Location: /customer-offers/OFF-2026-000124`, body is the full offer
- `400` — validation error (see section 3)
- `409` — an active offer already exists for this customer/product

---

### 5.4 Accept an offer

`POST /customer-offers/{offerId}/acceptance`

Request body:

```json
{ "acceptedBy": "C-10042", "channel": "MOBILE" }
```

Responses: `200` offer with `status: ACCEPTED`; `409` if offer expired or already accepted.

---

## 6. Versioning and deprecation

- Current version: **v1** (path prefix `/v1/`)
- Breaking changes ship as a new version; v(n-1) is supported for 6 months after v(n) GA
- Deprecated endpoints return a `Deprecation` header with the sunset date
- Additive changes (new optional fields, new endpoints) do **not** bump the version — clients must ignore unknown fields

---

## 7. Inconsistencies to fix

Flagged during generation — endpoints that deviate from section 2 conventions:

| Endpoint | Issue | Suggested fix |
|---|---|---|
| `GET /getOfferHistory` | Verb in path | Rename to `GET /customer-offers/{offerId}/history` |
| `POST /customer-offers/{id}/expire` | Verb-style action, returns 200 with empty body | Model as `PATCH` status change or `POST .../expiration`, return updated resource |
| `OfferDocumentController` | Errors return `{ "errorMsg": ... }` not ProblemDetail | Route through global `@ControllerAdvice` |