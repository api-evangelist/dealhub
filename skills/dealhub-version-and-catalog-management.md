---
name: Manage DealHub Versions and the product catalog
description: Duplicate and activate a DealHub configuration Version, read and push the
  product catalog, patch individual SKUs, and track every asynchronous job through to
  completion.
api: openapi/dealhub-version-api-openapi.yml
operations:
- getVersions
- getVersionById
- duplicateVersion
- activateVersion
- getProductCatalog
- getVersionProducts
- getProductsBySku
- uploadProductCatalog
- updateProductCatalogItems
- getAsyncRequestStatus
- getAsyncRequestSummary
- exportPlaybookData
generated: '2026-08-12'
method: generated
source: openapi/dealhub-version-api-openapi.yml,
  https://developers.dealhub.io/docs/version-api-overview,
  https://developers.dealhub.io/docs/handling-asynchronous-requests
---

# Manage DealHub Versions and the product catalog

A DealHub **Version** is the unit of configuration: products, pricing, playbooks and
API settings all live inside one. You never edit an ACTIVE version — you duplicate it,
change the copy while it is `DRAFT`, then activate it.

## Before you start

- `Authorization: Bearer <DealHub Authentication Token>`, base URL
  `https://api.dealhub.io`.
- No idempotency key exists on any operation. `duplicateVersion` called twice creates
  two versions.

## The safe change sequence

1. `getVersions` — list versions, optionally filtered by status. `getVersionById`
   returns one by id.
2. `duplicateVersion` — **asynchronous**. Creates a new `DRAFT` from an existing
   version and returns a `request_id`.
   - The source and target version accounts must share a domain name, or the call
     fails with `The domain name of the source version account does not match the
     target version account`.
   - A new version name must be unique.
3. Modify the DRAFT: `uploadProductCatalog` replaces the whole catalog,
   `updateProductCatalogItems` patches specific data elements on existing SKUs. Both
   are **asynchronous** and both target a `DRAFT` version only. Modifying an ACTIVE
   version returns `Specified version cannot be modified: Invalid version status.`
4. `activateVersion` — **asynchronous**. Activating an already-active version returns
   `Version (id= <version_id>) already active.`

## Batch size is the failure mode that will bite you

DealHub documents an open known issue: an asynchronous request carrying more than
roughly **500 SKUs** may stall in `queued` and never advance to `in-progress`.

- Keep each request under 500 SKUs.
- Split a large catalog into batches and submit each as its own request.
- Track every `request_id` individually — do not assume batch N succeeded because
  batch N-1 did.

## Track every asynchronous job

`getAsyncRequestStatus` returns `queued | in-progress | done | failed` plus
`error_description` and `error_code`. `getAsyncRequestSummary` gives the aggregate
picture.

Poll at 5–10 second intervals, widening as the job runs. Or subscribe to the
`productsUpload` webhook (success/error) and `version` webhook (`created`,
`activated`, `reactivated`), and to `Failure WebHook v1` for async failures — those
never appear on the original HTTP response.

## Reading the catalog

- `getProductCatalog` — paginated products and bundles with attributes and assignments.
- `getVersionProducts` — paginated basic product attributes and pricing for a version.
  Requesting `MODIFIED` products against a `DRAFT` version fails with
  `'MODIFIED' products option is not available for version in 'DRAFT' status. Fetch
  'ALL' products instead.`
- `getProductsBySku` — basic detail for a specific SKU list. Unknown SKUs come back as
  `The following SKUs not found: <LIST_OF_SKUS>`, and an oversized request returns
  `The number of requested items exceeds the allowed limit of <MAX_LIMIT>` — DealHub
  does not publish the numeric value of `MAX_LIMIT`, so discover it empirically and
  cache it.
- `exportPlaybookData` — exports the playbook configuration and data for a version.

## Hard limits enforced by the API

| Limit | Value |
|---|---|
| Product hierarchy depth | 10 levels |
| Product hierarchy items | 50,000 products or labels |
| SKU length | 200 characters |
| Label name length | 50 characters |
| SKUs per async request | keep below 500 (advisory, known issue above it) |

## Pagination and errors

List endpoints use `offset` and `limit`, default page size 50. An empty match returns
`200` with an empty array, not a 404.

Errors are proprietary JSON with a human-readable message and no machine-readable
code; `403 Unauthenticated` for auth failure, `400` for a missing entity. Full catalog
in `errors/dealhub-problem-types.yml`; cross-cutting semantics in
`conventions/dealhub-conventions.yml`.
