---
name: ownlocal-onboard-local-business
description: >-
  Onboard a local business (advertiser) into OwnLocal — look up the category vocabulary, create the business
  under one or more publishers with its locations and opening hours, then attach a logo and gallery imagery.
  Use when a publisher signs a new advertiser that needs a directory listing before ads can run.
api: OwnLocal API v1
base_url: https://admin.austin.ownlocal.com
operations:
  - listCategories
  - createBusiness
  - getBusiness
  - listBusinesses
  - uploadBusinessLogo
  - uploadBusinessImage
generated: '2026-08-12'
method: generated
source: >-
  openapi/ownlocal-businesses-openapi.yml + openapi/ownlocal-categories-openapi.yml (derived from OwnLocal's
  published Swagger 2.0 at https://admin.austin.ownlocal.com/api-docs/v1/swagger.json) + the published
  reference at https://api.docs.ownlocal.com/
---

# Onboard a local business into OwnLocal

A business is the advertiser an ad belongs to, and it is also the directory listing OwnLocal publishes on
that advertiser's behalf. Create it **before** submitting ads for it.

## Before you start

- **You must already know your `publisherUuid`(s).** `publishers[]` is required on create and there is no
  operation that lists publishers. It must be supplied out of band.
- **Send the API key raw** in `Authorization` — no `Bearer` prefix.
- **Check for an existing record first.** `createBusiness` has no idempotency key and there is no DELETE, so
  a duplicate is permanent as far as the API is concerned. Call `listBusinesses` and match on your own
  `custom_id` before you create.

## Step 1 — read the category vocabulary (`listCategories`)

`GET /api/v1/categories`

Returns top-level categories, each with a `subCategories` map of **integer id → name** (e.g.
`{"104": "Newspapers and Magazines"}`). Paged, default `size` 25 — page through it; there is no total count
in the response, so keep requesting until you get back fewer than `size` items.

**Watch the asymmetry:** you write a sub-category as its **integer id** (`primary_sub_cat_id`), but you read
it back as a **name string** in `subCategories[]`. Cache the id↔name map from this call; you will need it in
both directions and the API will not do the translation for you.

## Step 2 — create the business (`createBusiness`)

`POST /api/v1/businesses`, with everything nested under a `business` object.

The reference states `publishers[]` (at least one) and `name` are **required** — note that OwnLocal's own
machine contract declares **no** required fields on the business schema, so the spec is weaker than the
documentation here. Follow the documentation.

Fields worth getting right on the first call, because there is no update-business operation in the machine
contract (the prose reference documents a `POST` to the item URL, but the Swagger definition does not
declare it — treat business updates as unverified):

- `name`, `custom_id`, `long_description`
- Address: `address`, `address2`, `city`, `state`, `country`, `zip`, `zip_suffix` — `state` and `country`
  must be **codes**, not names. The reference publishes the full country/state code table (US, CA, AU, GB,
  IE, DE, NL, BE, DK, NZ, MX, PA, HN).
- `phone`, `email`, `fax`, `website`, `contact`
- `primary_sub_cat_id` — the integer from step 1.
- `hours_of_operation` — a flat object with five keys **per day**: `<day>_open`, `<day>_close`,
  `<day>_open_24_hours`, `<day>_closed_all_day`, `<day>_by_appointment_only`. Times are **12-hour format
  only** (`"08:00AM"`), and the three boolean-ish flags are the **strings** `"1"` or `"0"`, not booleans.

As with ads, the "Invalid input" response on this operation is declared as **`405`**, not `400`.

## Step 3 — attach imagery

- `POST /api/v1/businesses/{businessUuid}/logo` — `uploadBusinessLogo`
- `POST /api/v1/businesses/{businessUuid}/images` — `uploadBusinessImage` (call once per gallery image)

Both are `multipart/form-data` with a required `file` part and a required `fileName` string part, and both
return **`202 Accepted`** — processing is asynchronous with no completion callback. A rejected file returns
`400` with the `unacceptableContent` body.

## Step 4 — verify (`getBusiness`)

`GET /api/v1/businesses/{businessUuid}`

Poll until `logo_url` and `images[]` populate — that is the only signal that step 3 completed. Confirm
`opening_hours` rendered the way you intended: the API transforms your flat `hours_of_operation` write into
a per-day **display string** on read (`"08:00AM - 10:00PM"`, `"Open All Day"`, `"Close All Day"`,
`"Appointments Only"`). The write shape and the read shape are different structures for the same data.

## Failure handling

| Status | Do |
|---|---|
| `401` | Key wrong or prefixed. Send it raw. |
| `400` | Upload rejected — parse `unacceptableContent.message`, re-upload against the same `businessUuid`. |
| `404` | Bad `businessUuid`. Do not create a replacement without checking `listBusinesses` first. |
| `405` | Create body invalid — confirm `publishers[]` and `name`. |
| `429` | Back off exponentially; no `Retry-After` is sent. |
| `5xx` | Retry reads freely. Do **not** blindly retry `createBusiness`. |

## Related

- `ownlocal-submit-print-ad` — submit ads once the business exists.
- `ownlocal-report-ad-performance` — read performance for this business.
- `data-model/ownlocal-data-model.yml` for the full entity graph.
