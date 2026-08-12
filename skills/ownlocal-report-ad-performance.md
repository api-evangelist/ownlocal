---
name: ownlocal-report-ad-performance
description: >-
  Pull OwnLocal performance reporting — a publisher-wide ads report (impressions, interaction breakdown by
  click target, leads, digital lift) or a per-business report that adds print reach, directory and Origami
  breakdowns and search keyword rank history. Use for advertiser-facing campaign reporting or publisher
  revenue review.
api: OwnLocal API v1
base_url: https://admin.austin.ownlocal.com
operations:
  - getAdsReport
  - getBusinessReport
  - listAds
  - listBusinesses
generated: '2026-08-12'
method: generated
source: >-
  openapi/ownlocal-reports-data-api-openapi.yml (derived from OwnLocal's published Swagger 2.0 at
  https://admin.austin.ownlocal.com/api-docs/v1/swagger.json) + the published reference at
  https://api.docs.ownlocal.com/
---

# Read OwnLocal ad and business performance

Two report operations, both scoped to a publisher, both synchronous ("builds and returns").

## Before you start

- **`publisher_id` is required on both reports** and cannot be discovered through the API. Supply it out of band.
- **These operations use snake_case**, unlike the rest of the API. `publisher_id`, `business_id`,
  `start_date`, `end_date` — not the camelCase (`publisherUuids`, `dateCreatedMin`) used on the Ads and
  Businesses resources. Do not carry a naming convention across from one to the other.
- **Date defaults are implicit**: `start_date` defaults to *one month ago* and `end_date` to *today*. If you
  need a deterministic window, always pass both.

## Ads report (`getAdsReport`)

`GET /api/v1/reports/ads?publisher_id=<uuid>&start_date=<date>&end_date=<date>`

The reference also documents an optional `ad_id` filter that narrows the report to a single ad and accepts
**either** an ad id **or** an origami id. Note that `ad_id` is documented in the prose but is **not**
declared in OwnLocal's machine contract — treat it as unverified and fall back to filtering client-side if
it does not take.

Response shape: window dates, `publisher_name`, `publisher_id`, then `ads[]`. Each entry carries
`report_url` (a human-readable ad report page on `reports.ownlocal.com`) and a `data` object with:

- `total_views` — impressions
- `interaction_breakdown.breakdowns.overall` — one counter per click target: `total_clicks`,
  `expand_ad_unit_clicks`, `website_clicks`, `call_clicks`, `directions_clicks`, `contact_form_submits`,
  `about_us_clicks`, `contact_clicks`, `print_ad_clicks`, `share`, plus per-network social clicks
  (`facebook_clicks`, `twitter_clicks`, `instagram_clicks`, `yelp_clicks`)
- `leads`
- `digital_lift` — a float, OwnLocal's headline metric for incremental reach over print alone

**Schema caveat:** `ads[]` items `$ref` a schema (`ad_reach`) that OwnLocal's own Swagger never defines, so
the entry shape is only knowable from the response example. Parse defensively — do not assume a field is
present because the example has it.

## Business report (`getBusinessReport`)

`GET /api/v1/reports/business?publisher_id=<uuid>&business_id=<uuid>&start_date=<date>&end_date=<date>`

`business_id` is **optional**: omit it and you get a report for *every* business under the publisher. That is
the expensive call — bound it with a narrow date window before running it across a large publisher.

Adds two blocks the ads report does not have:

- **`reach_report`** — `print_reach`, `total_views`, `leads`, `digital_lift`, `total_ads`, plus separate
  `directory` (days, impressions, article_impressions) and `origami` (ads, days, impressions) breakdowns, and
  the same `interaction_breakdown`. Business-level interactions include two extra counters not present at ad
  level: `coupons_printed` and `question_intents`.
- **`search_report.keywords[]`** — per keyword: `current_rank`, `initial_rank`, `initial_date`, a
  `rank_history[]` time series, and `search_results[]` showing the surrounding SERP with
  `is_current_business` marking the advertiser's own row. Ranks can be the **string** `"100+"` rather than an
  integer — type-check before doing arithmetic on a rank.

`reach_report` has the same undefined-`$ref` problem (`business_reach`), and in the source spec applies an
`items` keyword to a `type: object`. Parse defensively.

## Practical notes

- **No pagination.** Neither report takes `page` or `size`. A wide window over a large publisher returns one
  large document — control size with the date range, which is the only lever you have.
- **Reports are computed on request.** Expect latency to grow with the window. Cache results rather than
  re-requesting the same window.
- **`429` is enforced and unquantified** — no `Retry-After`, no `RateLimit-*` headers. Serialize report
  calls rather than fanning out, and back off exponentially on a 429.
- **Neither operation declares any error response** in the spec — only `200`. Handle `401`, `403`, `429` and
  `5xx` from the reference's error table even though a generated client will not know they exist.

## Related

- `ownlocal-submit-print-ad`, `ownlocal-onboard-local-business` — produce the ads and businesses reported on here.
- `errors/ownlocal-problem-types.yml`, `rate-limits/ownlocal-rate-limits.yml`.
