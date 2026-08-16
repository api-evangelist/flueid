---
name: Build and report on a Flueid Pro farm
description: Define a geographic or filter-based farm (a prospecting territory) in Flueid Pro, populate it by polygon search or spreadsheet upload, save reusable filters, and pull the farm overview and Excel report.
api: openapi/flueid-pro-openapi.yml
base_url: https://api.pro.flueid.com
operations:
  - GET /api/Farms/GetAllFarms
  - GET /api/Farms/FarmFilters
  - POST /api/Farms/SaveFarmFilter
  - POST /api/Farms/SearchFarmGeoPoints
  - POST /api/Farms/SearchFarmProps
  - POST /api/Farms/SaveFarm
  - POST /api/Farms/AddPropertyToFarm
  - POST /api/Farms/UploadExcelFarm
  - POST /api/Farms/FixNotFoundPropertyForFarm
  - POST /api/Farms/FarmOverview
  - GET /api/Farms/FarmReport
generated: '2026-08-16'
method: generated
source: openapi/flueid-pro-openapi.yml
---

# Build and report on a Flueid Pro farm

A "farm" in Flueid Pro is a saved prospecting territory — a set of properties defined by geography
and/or filters that a real estate or title professional works. No `operationId` exists on any
operation; bind by method + path against `openapi/flueid-pro-openapi.yml`.

## Before you start

- Base URL `https://api.pro.flueid.com`, `Authorization: Bearer <JWT>`, `api-version=1.0`.
- Farms are per-user. `GET /api/Farms/GetAllFarms` — "Gets all farms for the user."

## Path A — build a farm from a map area

1. **Load existing filters.** `GET /api/Farms/FarmFilters` — "Gets any global filters or filters saved
   by the user." Reuse one rather than rebuilding criteria the user already saved.
2. **Search the polygon.**
   `POST /api/Farms/SearchFarmGeoPoints` and `POST /api/Farms/SearchFarmProps` both "find properties
   within a defined polygon" — the first returns geo points for map rendering, the second the property
   records. Filter shapes available include min/max ranges, occupancy status, property status and
   property labels (see `FarmFilterMinMax`, `FarmFilterOccupancyStatus`, `FarmFilterPropertyStatus`,
   `FarmFilterPropertyLabel` in `data-model/flueid-data-model.yml`).
3. **Save the criteria for reuse.** `POST /api/Farms/SaveFarmFilter`.
4. **Create the farm.** `POST /api/Farms/SaveFarm` — "Saves the farm." Confirm the name with the user
   first; `POST /api/Farms/RenameFarm` changes it later.
5. **Adjust membership.** `POST /api/Farms/AddPropertyToFarm` and
   `POST /api/Farms/DeletePropertyFromFarm`. `POST /api/Farms/SaveFarmProperty` writes per-property
   farm attributes.

## Path B — build a farm from a spreadsheet

1. `POST /api/Farms/UploadExcelFarm` — "Accepts a spreadsheet of properties and processes it to create
   a new farm." Processing is asynchronous in effect: expect a run to complete before the farm is whole.
2. **Reconcile the misses.** Not every row will match a Flueid property.
   `POST /api/Farms/FixNotFoundPropertyForFarm` — "Fixes unfound property from the farm upload." Walk
   the unmatched rows with the user; do not silently drop them.
3. `POST /api/Farms/MarkFarmUploadViewed` once the user has reviewed the result.

## Reporting

- `POST /api/Farms/FarmOverview` — "Gets the farm overview information." Use this for the summary you
  show in conversation.
- `GET /api/Farms/FarmReport` — "Gets an Excel report for the farm." This is the deliverable; it
  returns a spreadsheet, not JSON.
- `GET /api/Farms/GetFarm` for a single farm's definition.

## Destructive operations — confirm first

`POST /api/Farms/DeleteFarm` and `POST /api/Farms/DeleteFarmFilter` are unrecoverable from the API.
Always confirm with the user by farm name before calling either, and never call them as part of an
automated cleanup.

## Error handling

- The contract declares **only 200** on every operation. Any other status is undocumented — report the
  raw status and body and stop.
- No idempotency key exists (`conventions/flueid-conventions.yml`). A retried `SaveFarm` or
  `UploadExcelFarm` can create a duplicate farm. On a failed write, re-run `GET /api/Farms/GetAllFarms`
  to see what actually landed before doing anything else.
- Paging on farm searches is per-request (`page`, `pageSize`, `sortBy` on the filter body); there is no
  shared paged envelope and no cursor.
