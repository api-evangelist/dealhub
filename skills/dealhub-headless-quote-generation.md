---
name: Generate and publish a DealHub quote headlessly
description: Simulate pricing, generate a quote from an API Playbook, submit it for
  approval and publish it to a DealRoom entirely from a backend system, with no user
  interaction in the DealHub UI.
api: openapi/dealhub-headless-api-openapi.yml
operations:
- getVersions
- getGenerateQuoteTemplate
- simulateQuote
- generateQuote
- submitQuote
- publishQuote
- signQuoteExternally
- getQuoteById
- getQuoteDocument
generated: '2026-08-12'
method: generated
source: openapi/dealhub-headless-api-openapi.yml, openapi/dealhub-version-api-openapi.yml,
  openapi/dealhub-quote-api-openapi.yml, https://developers.dealhub.io/docs/headless-api-overview
---

# Generate and publish a DealHub quote headlessly

Use this when a backend system — a self-service checkout, a renewal job, an agent —
must produce a real DealHub quote without a salesperson opening the UI.

## Before you start

- Authenticate every call with `Authorization: Bearer <DealHub Authentication Token>`.
  A CPQ administrator generates the token in Control Panel > System Settings > API
  Settings; it is displayed once and cannot be retrieved again.
- Base URL is `https://api.dealhub.io`.
- A missing or invalid token returns **HTTP 403** with the message `Unauthenticated`.
  DealHub does not use 401.
- **There is no idempotency key.** None of DealHub's 616 published operations accepts
  one. Never blind-retry a `generateQuote`, `submitQuote` or `publishQuote` call — a
  retry can create a second quote. If a call times out, resolve state by reading
  before you write again.

## Step 1 — resolve the Version you are quoting against

Everything in DealHub is scoped to a **Version**: products, pricing rules, playbooks
and API configuration. Call `getVersions` and pick the `ACTIVE` version, or pass a
specific `version_id` when you must pin behaviour.

If you omit `version_id`, DealHub resolves against the ACTIVE version — which means
the same request can return different results after an administrator activates a new
Version. Pin it if the caller needs reproducibility.

## Step 2 — get the request shape for the playbook

Call `getGenerateQuoteTemplate` (Version API) to retrieve a preformatted JSON template
for the playbook you are quoting. Build your request body from that template rather
than hand-rolling it — the playbook structure differs per tenant.

## Step 3 — price it without committing

Call `simulateQuote` to calculate line items and a financial summary **without**
creating a persistent quote. This is synchronous.

The request must carry `version_id` plus **exactly one** of:

- `external_opportunity_id` — to simulate against an existing CRM opportunity, or
- both `geo_code` and `currency` — for a simulation with no CRM dependency.

Sending both combinations is an error. `quote_data` is **required and must always be
present**, even when there are no line items; send `"quote_data": []`. Omitting the
field entirely fails, and the auto-generated code sample on the docs page may not
show it.

## Step 4 — create the quote

Call `generateQuote` to persist a Draft quote. Keep the returned identifiers.

If you only hold the `dealhub_proposal_id` — the id a human sees in the UI — resolve
it to the API-side id with `getQuoteId`.

## Step 5 — move it through the lifecycle

- `submitQuote` — **asynchronous**. Submits a `Draft` quote for approval.
- `publishQuote` — **asynchronous**. Publishes a `Ready to be sent` quote to a DealRoom.
- `signQuoteExternally` — **asynchronous**. Marks a quote `Won` when it was signed
  outside the DealRoom. The quote must be in `Ready to be sent` or `Published`.

## Step 6 — track the asynchronous work

Each asynchronous call returns a `request_id` immediately, not a finished result.
Poll `getAsyncRequestStatus` (`GET /api/v1/request/{request_id}/status`), which
returns one of `queued`, `in-progress`, `done`, `failed`.

Poll every 5–10 seconds at first, then widen the interval. Better: subscribe to the
webhook instead of polling — `quotePendingApproval`, `quoteReady`, `quoteRejected`,
`quotePublished` and `quoteWon` all fire on this path.

**Failures do not come back on the HTTP response.** A failed async job surfaces as
`status: failed` with `error_description` and `error_code` on the status endpoint, and
is separately delivered as the `Failure WebHook v1` event.

## Step 7 — read the result

- `getQuoteById` returns quote detail. By default it returns **only**
  `dealhub_quote_id`, `status` and `quote_upgrade_required` — you must opt in to
  detail by repeating the `feature` query parameter.
- If `quote_upgrade_required` is `true`, the draft was built on a now-inactive Version
  and DealHub will withhold **all** requested `feature` data. Handle this case
  explicitly rather than treating the empty response as "no data".
- `getQuoteDocument` returns the output document. It defaults to the format and
  template the quote was generated with; `document_type` (PDF, WORD) works only if
  that format is enabled for the playbook. EXCEL requires the quote to have been
  configured for Excel output.

## Error handling

- `403` `Unauthenticated` — token missing, invalid, or not matching the user.
- `400` `Entity (ID = <entity_id>) not found` — DealHub returns 400, not 404, for a
  missing entity.
- `400` `Request payload missing mandatory field(s): <field1>, <field2>`.
- `400` `Unrecognized parameter: <PARAM_NAME>` — DealHub rejects unknown fields
  rather than ignoring them.

Full catalog: `errors/dealhub-problem-types.yml`.
