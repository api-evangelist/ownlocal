---
name: ownlocal-submit-print-ad
description: >-
  Submit a newspaper print advertisement to OwnLocal for automatic conversion into a digital ad unit —
  create the ad record under a publisher, upload the source PDF, then poll until OwnLocal's conversion
  pipeline has populated the derived creative. Use when a publisher has a print ad PDF that needs to run
  online.
api: OwnLocal API v1
base_url: https://admin.austin.ownlocal.com
operations:
  - createAd
  - uploadAdContent
  - getAd
generated: '2026-08-12'
method: generated
source: >-
  openapi/ownlocal-ads-openapi.yml (derived from OwnLocal's published Swagger 2.0 at
  https://admin.austin.ownlocal.com/api-docs/v1/swagger.json) + the published reference at
  https://api.docs.ownlocal.com/
---

# Submit a print ad to OwnLocal

This is OwnLocal's marquee flow: a print ad PDF goes in, a responsive digital ad unit comes out. It is a
**two-call create followed by a poll**, because the conversion is asynchronous and OwnLocal publishes no
completion callback.

## Before you start

- **You must already know your `publisherUuid`.** It is required on every ad. The API exposes **no**
  operation that lists publishers, so this value has to be supplied out of band by OwnLocal. If you do not
  have it, stop — you cannot discover it.
- **Authenticate with the raw key.** Send `Authorization: <token>` with **no** `Bearer` prefix. A `Bearer`
  prefix will 401. The key is organization-scoped and is issued by OwnLocal support.
- **These calls are not idempotent and not reversible.** There is no `Idempotency-Key`, and there is no
  DELETE on ads anywhere in the API. A retry after a timeout creates a *second* ad that you cannot remove
  through the API. Set `adCustomId` to your own external id on every create so you can detect a duplicate,
  and check `listAds` before retrying rather than blindly re-POSTing.

## Step 1 — create the ad record (`createAd`)

`POST /api/v1/ads`

| Field | Required | Notes |
|---|---|---|
| `publisherUuid` | yes | The publisher the ad runs under. The only genuinely required field. |
| `businessUuid` | no | The advertiser. Create it first with `ownlocal-onboard-local-business` if it does not exist. |
| `adCustomId` | no | Your external id. Not enforced unique, but it is the only reconciliation handle you get. |
| `startDate` / `endDate` | no | Run dates. The reference states that supplying one requires the other. |

The `201` response returns `adUuid`, a `status`, a `message`, and a **`content_url`** — the only hypermedia
link in the entire API. It points at the upload endpoint for step 2.

Note the error semantics here: the spec declares **`405`** — not `400` — as the "Invalid input" response on
this operation. If you get a 405 from `createAd`, treat it as a body validation failure, not as a wrong HTTP
method.

## Step 2 — upload the print PDF (`uploadAdContent`)

`POST /api/v1/ads/{adUuid}/content` with `Content-Type: multipart/form-data`.

Two parts, both required: `file` (the PDF binary) and `fileName` (the name as a string — yes, it is
redundant with the multipart filename, and yes, the API wants both).

**PDF only.** A rejected file returns `400` with the API's one and only typed error body,
`unacceptableContent`, which carries a single `message` string. That is the only error payload you can parse
programmatically anywhere in this API — read it and surface it verbatim.

A success is **`202 Accepted`**, not `200`. The file has been queued, not processed.

## Step 3 — poll for the converted creative (`getAd`)

`GET /api/v1/ads/{adUuid}`

There is **no webhook, no callback, and no job-status resource** — polling is the only option OwnLocal
offers. Watch for these derived fields to populate, which is the signal that conversion finished:

- `image` — the rendered digital ad unit
- `adText` — text extracted from the PDF
- `offers[]` / `offerCount` — coupons and promotions machine-extracted from the creative

Poll with backoff. `429` is enforced but carries **no** `Retry-After` and no `RateLimit-*` headers, so use
blind exponential backoff (start around 5s, cap at a minute) and do not tighten the loop when nothing has
changed. There is no published limit to pace against.

## Failure handling

| Status | On this flow | Do |
|---|---|---|
| `401` | Key wrong, or you sent `Bearer` | Send the key raw. Do not retry with the same header shape. |
| `400` | Upload rejected | Parse `unacceptableContent.message`. Re-upload a valid PDF against the **same** `adUuid` — do not create a new ad. |
| `404` | Bad `adUuid` | The id is a UUID scoped to your organization. Do not create a replacement on a 404 without checking `listAds` first. |
| `405` | Create body invalid | Fix the body. Confirm `publisherUuid` is present. |
| `429` | Rate limited | Back off exponentially. No server hint is available. |
| `5xx` | Server side | Safe to retry `getAd`. **Not** safe to retry `createAd` — check `listAds?businessUuids=...` for your `adCustomId` first. |

## Related

- `ownlocal-onboard-local-business` — create the advertiser this ad belongs to.
- `ownlocal-report-ad-performance` — read how the ad performed once it is running.
- `conventions/ownlocal-conventions.yml`, `errors/ownlocal-problem-types.yml`.
