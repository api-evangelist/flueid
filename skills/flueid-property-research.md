---
name: Research a property with Flueid Pro
description: Find a property in the Flueid Pro property database, pull its full detail, ownership history and tax rolls, and assemble a comparable-sales view for a listing or title conversation.
api: openapi/flueid-pro-openapi.yml
base_url: https://api.pro.flueid.com
operations:
  - POST /api/PropertyData/PropertySearchSuggestions
  - POST /api/PropertyData/FindProperties
  - GET /api/PropertyData/PropertyDetail
  - GET /api/PropertyData/PropertyHistory
  - GET /api/PropertyData/GetPropertyTaxRolls
  - POST /api/PropertyData/SalesComps
  - POST /api/PropertyData/ZillowValuation
  - POST /api/PropertyData/PropertyProfileReportBase64
generated: '2026-08-16'
method: generated
source: openapi/flueid-pro-openapi.yml
---

# Research a property with Flueid Pro

The Flueid Pro API has **no `operationId` on any operation**, so every step below is bound by HTTP
method + path. Verify each path against `openapi/flueid-pro-openapi.yml` before calling.

## Before you start

- Base URL: `https://api.pro.flueid.com`
- Send `Authorization: Bearer <JWT>`. The token comes from the Flueid Pro AWS Cognito user pool.
  The published contract declares **no** `securitySchemes`, so this is not discoverable from the spec —
  see `authentication/flueid-authentication.yml`.
- Every operation accepts an optional `api-version` query parameter; send `api-version=1.0`.
- Request bodies are `application/json`.

## Steps

1. **Turn free text into candidate properties.**
   `POST /api/PropertyData/PropertySearchSuggestions` — "Gets suggestions matching the passed search
   text, within a certain geographic area." Use this for an address the user typed; it is the
   autocomplete surface. For a fuller match set use `POST /api/PropertyData/FindProperties`
   ("Finds properties matching the specified text").
   If you have a bounding box or map viewport instead of text, use
   `POST /api/PropertyData/PropertiesNearby` ("Gets all properties within a defined geographic box").

2. **Resolve to one property id.** Do not guess. If more than one candidate comes back, present them
   and let the user choose. With an id in hand, `GET /api/PropertyData/PropertyById` returns the single
   record; `POST /api/PropertyData/PropertiesByIds` batches several.

3. **Pull the full record.**
   `GET /api/PropertyData/PropertyDetail` — "Fetches the full property details by property ID."
   Pass `propertyId` as a query parameter.

4. **Pull ownership and transaction history.**
   `GET /api/PropertyData/PropertyHistory` — "Fetches the full property history by property ID."
   This is the chain an agent should read before making any statement about ownership or liens.

5. **Pull tax position.**
   `GET /api/PropertyData/GetPropertyTaxRolls` — "Fetches the property tax rolls from BKFS."
   Note the upstream source is a third party (Black Knight/BKFS); treat gaps as a coverage gap, not a
   contradiction.

6. **Build the valuation picture.**
   `POST /api/PropertyData/SalesComps` — "Searches for sales comparables matching the supplied
   criteria." Then `POST /api/PropertyData/ZillowValuation` for a third-party AVM and
   `GET /api/PropertyData/MarketAnalysis` for the zip-code level view.
   Label the Zillow figure as Zillow's, never as Flueid's.

7. **Produce the deliverable.**
   `POST /api/PropertyData/PropertyProfileReportBase64` — "Requests the property profile in PDF
   format." The response is base64; decode before writing to disk.

8. **Optional — recorded documents.** Image availability is county-by-county:
   first `GET /api/PropertyData/CheckDocumentImagesCapability` ("Verify county (fips code) eligibility
   for pulling images based on the partner set"), then `GET /api/PropertyData/RequestDocumentImage`,
   then poll `GET /api/PropertyData/GetPropertyDocumentImages`. Document image requests may bill; ask
   before requesting.

## Error handling

The contract documents **only HTTP 200** on all 132 operations — there is no declared 4xx/5xx, no
error schema, and no published error reference. Therefore:

- Treat any non-200 as opaque. Surface the raw status and body to the user; do not invent a meaning.
- Do not retry a write blindly: the API supports **no idempotency key**
  (`conventions/flueid-conventions.yml`). Reads in this skill are safe to retry; step 8's
  `RequestDocumentImage` is not.
- No rate-limit headers are published. Back off exponentially on repeated failures.
