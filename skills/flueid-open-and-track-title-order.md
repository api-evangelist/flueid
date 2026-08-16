---
name: Open and track a Flueid Pro title order
description: Collect the order options and contacts a Flueid Pro partner requires, check for a duplicate in-process order, create a closing-services order, then follow its status, history and documents through to close.
api: openapi/flueid-pro-openapi.yml
base_url: https://api.pro.flueid.com
operations:
  - POST /api/NewOrders/OrderTypeOptions
  - POST /api/NewOrders/OrderDetailsOptions
  - POST /api/NewOrders/OrderContactOptions
  - POST /api/NewOrders/DefaultOrderContacts
  - POST /api/NewOrders/CheckDuplicateOrder
  - POST /api/NewOrders/CalculateCloseDate
  - POST /api/NewOrders/CreateOrder
  - POST /api/Orders/SearchOrders
  - GET /api/Orders/OrderDetail
  - GET /api/OrderEvents/OrderHistory
  - POST /api/Orders/UpdateOrderStatus
  - POST /api/Orders/GetDocument
generated: '2026-08-16'
method: generated
source: openapi/flueid-pro-openapi.yml
---

# Open and track a Flueid Pro title order

Creating an order commits real work at a title operation. Confirm with the user before step 6.
No `operationId` exists on any operation — bind by method + path, verified against
`openapi/flueid-pro-openapi.yml`.

## Before you start

- Base URL `https://api.pro.flueid.com`, `Authorization: Bearer <JWT>`, `api-version=1.0`.
- Order shape is **partner-specific**. Which sections are collected, which order types exist and which
  contact roles are required all come from the partner's configuration — never hardcode them.

## Steps

1. **Establish the partner context.**
   `GET /api/AccountPartner/GetActivePartners`, then `POST /api/AccountPartner/SetCurrentPartner` if
   the user belongs to more than one. `GET /api/Partners/PartnerOrderOptions` returns that partner's
   order options.
   Note: `GET /api/Partners/PartnerOrderSettings` and `POST /api/Partners/SavePartnerOrderSettings`
   are flagged `deprecated: true` in the contract — prefer the `PartnerOrderSettings`-tagged
   operations and do not build new work on the deprecated pair.

2. **Ask the API what this order needs.**
   - `POST /api/NewOrders/OrderTypeOptions` — "Gets the order type options for the new order."
   - `POST /api/NewOrders/OrderDetailsOptions` — "Gets the settings to indicate which sections of data
     should be collected for an order."
   - `POST /api/NewOrders/OrderContactOptions` — "Gets the order contact options for the new order."
   Drive your prompts from these responses. If a section is not returned, do not collect it.

3. **Resolve people to real records.** Do not free-text a name into an order.
   - `POST /api/NewOrders/DefaultOrderContacts` — the business source user's defaults.
   - `GET /api/NewOrders/SearchOrderContact` — "Searches for order contact based on role."
   - `POST /api/NewOrders/CompanyTitleOfficers`, `POST /api/NewOrders/CompanyAccountManagers`,
     `POST /api/NewOrders/RelatedOfficers`, `GET /api/NewOrders/RelatedAssistants` for the internal side.
   - `GET /api/NewOrders/BusinessSourceRoles` — "Gets the roles that this business source can play on
     an order."

4. **Attach the property.** Use the property-research skill
   (`skills/flueid-property-research.md`) to resolve an address to a Flueid property id first.

5. **Guard rails before the write.**
   - `POST /api/NewOrders/CheckDuplicateOrder` — "Checks if the property is already involved in a
     current in-process order." **Always run this.** If it reports a duplicate, stop and report back;
     do not create a second order.
   - `POST /api/NewOrders/CalculateCloseDate` — "Gets a default estimated closing date."

6. **Create the order.** `POST /api/NewOrders/CreateOrder` — "Creates a new order for closing
   services."
   This is a non-idempotent write. There is no `Idempotency-Key` on this API
   (`conventions/flueid-conventions.yml`). **If the call times out or returns a non-200, do not retry.**
   Re-run `POST /api/NewOrders/CheckDuplicateOrder` and `POST /api/Orders/SearchOrders` to find out
   whether the order landed, and report what you found.

7. **Track it.**
   - `POST /api/Orders/SearchOrders` — "Gets a list of orders that are viewable by the user."
   - `GET /api/Orders/OrderDetail` — full detail, by `orderId` query parameter.
   - `GET /api/OrderEvents/OrderHistory` — the event trail.
   - `GET /api/Orders/GetDecisionOrderData` returns the Flueid Decision (VOT) payload attached to the
     order, where the partner is licensed for it.

8. **Keep it current.**
   - `POST /api/Orders/RunDateDownUpdate` — "Performs a datedown update on the order."
   - `POST /api/Orders/RunFullUpdate` — "Performs a full update on the order."
   - `POST /api/Orders/UpdateOrderStatus` — "Updates an existing orders status." Status values come
     from the partner's configured set; read them rather than assuming.
   - `POST /api/Orders/GetDocument` and `POST /api/Orders/DownloadTrailingDocument` for documents.

## Error handling

- The contract declares **only 200** on every operation. Any other status is undocumented: report the
  raw status and body verbatim and stop the flow.
- No idempotency, no rate-limit headers, no `Retry-After`. Never auto-retry steps 6 or 8.
- Three deprecated operations exist with no replacement named and no sunset date
  (`lifecycle/flueid-lifecycle.yml`). Avoid them.
