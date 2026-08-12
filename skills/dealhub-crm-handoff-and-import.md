---
name: Hand a CRM user into DealHub and import CRM accounts
description: Authenticate a CRM or PRM user into DealHub with a 60-second one-time
  token, redirect them into a quote, and run a tracked asynchronous import of buyer
  accounts and contacts from the connected CRM.
api: openapi/dealhub-crm-api-openapi.yml
operations:
- authenticateUser
- createQuote
- openQuote
- viewQuotesForOpportunity
- viewMyOpportunities
- authenticatePartnerUser
- createPartnerQuote
- openPartnerQuote
- openPartnerOpportunity
- startCrmImport
- getCrmImportStatus
- getCrmImportDetail
- lookupCrmImportId
- retryCrmImport
generated: '2026-08-12'
method: generated
source: openapi/dealhub-crm-api-openapi.yml, openapi/dealhub-partner-api-openapi.yml,
  openapi/dealhub-crm-import-api-openapi.yml,
  https://developers.dealhub.io/docs/authentication-overview
---

# Hand a CRM user into DealHub and import CRM accounts

DealHub is CRM-governed. Two distinct jobs live here: getting a *person* from the CRM
into a DealHub quote, and getting *data* from the CRM into DealHub.

## Part 1 — the two-step user handoff

This flow is deliberately split across server and client, and getting the split wrong
is the usual failure.

### Step 1 (server-to-server)

`authenticateUser` — POST `/api/v1/authenticate/user` with your long-lived DealHub
authentication key and the user's identity in the body. DealHub returns a
**one-time access token valid for 60 seconds**.

For a partner portal or PRM, use `authenticatePartnerUser` instead — same shape.

### Step 2 (must run client-side)

Pass the one-time token to the browser and call `createQuote` or `openQuote` **from
the client**, presenting the one-time token as the bearer.

This call has to originate client-side because DealHub returns a redirect URL *and*
sets a session cookie in the browser, which is what logs the user into the DealHub
portal. Call it from your server and the redirect will land on a login screen.

Partner equivalents: `createPartnerQuote`, `openPartnerQuote`,
`openPartnerOpportunity`.

### Read-only redirects

`viewQuotesForOpportunity` returns a URL to the **My Proposals** page for one
opportunity; `viewMyOpportunities` returns the user's **My Opportunities** dashboard.
Both are URL-producing endpoints, not data endpoints.

## Part 2 — importing buyer accounts and contacts

`startCrmImport` starts — or **joins** — an asynchronous import of buyer accounts and
their contacts from the tenant's connected CRM. It returns a `request_id`.

Because it joins an in-flight import rather than starting a duplicate one, this is the
closest thing DealHub has to a safe re-entrant write. It is still not an idempotency
key: it deduplicates by import, not by request.

Track it:

- `getCrmImportStatus` — aggregate counts: total, processed, succeeded, failed,
  skipped.
- `getCrmImportDetail` — the same summary plus every buyer account in the import.
- `lookupCrmImportId` — resolve a single CRM id within an import: is it a buyer
  account, a contact, or not part of the import at all.
- `retryCrmImport` — re-runs **only** the failed ids plus any ids left unfinished by
  an interruption, under the **same `request_id`**. Already-succeeded ids are not
  reprocessed, so this is safe to call more than once.

The `crmImport` webhook fires on completion with total/succeeded/failed/skipped counts
and the originating `request_id` in `event_info` — prefer it over polling.

## Error handling

- `403 Unauthenticated` — token missing, invalid, or not matching the user. This is
  also what an **expired one-time token** looks like; the 60-second window is short,
  so do not stage the token through a queue.
- `400 Entity (ID = <entity_id>) not found` — DealHub returns 400, not 404.
- `500` responses are declared on every CRM Import operation; treat import
  orchestration as failure-prone and always reconcile with
  `getCrmImportDetail` rather than trusting a single response.

See `conventions/dealhub-conventions.yml` for auth, pagination and async semantics,
and `errors/dealhub-problem-types.yml` for the full error catalog.
