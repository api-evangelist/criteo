---
name: Run an async Retail Media report and collect the output
description: Request a Criteo Retail Media analytics report, poll it to completion, and download the output — respecting the stricter reporting rate limit and the attribution data-freshness windows.
api: openapi/criteo-retail-media-api-openapi.yml
api_version: '2026-07'
base_url: https://api.criteo.com/2026-07/retail-media
scopes:
  - RetailMedia_Analytics_Read
  - RetailMedia_Billing_Read
operations:
  - GenerateAsyncPerformanceReport
  - GenerateAsyncRevenueReport
  - GenerateAsyncFillRateReport
  - GenerateAsyncMissedOpportunitiesReport
  - GenerateAsyncAttributedTransactionsReport
  - GenerateAsyncUnfilledPlacementsReport
  - GetAsyncExportStatus
  - GetAsyncExportOutput
  - GenerateSyncCampaignsReport
  - GenerateSyncLineItemsReport
  - GenerateSyncRealTimePerformanceReport
  - CreatePartnerBillingReportRequestV1
  - GetPartnerBillingReportStatusV1
  - GetPartnerBillingReportOutputV1
generated: '2026-08-13'
method: generated
source: openapi/criteo-retail-media-api-openapi.yml (2026-07) + https://developers.criteo.com/criteo-apis/docs/rate-limits
---

# Run an async Retail Media report and collect the output

Criteo reporting is request/poll/fetch. It is also the most rate-limited surface on the
platform, so the way you shape the request matters more than the way you poll it.

## Before you start

- `RetailMedia_Analytics_Read` for analytics; `RetailMedia_Billing_Read` for partner billing.
  Both are **Read-level only** — there is no Manage level on either domain.
- Reporting endpoints are capped at **40 calls/min**, not the default 250. Budget accordingly.

## Steps

1. **Choose the right generator.** All are POST, all under `/reports`:

   | Report | operationId |
   |---|---|
   | Campaign / line-item performance | `GenerateAsyncPerformanceReport` |
   | Revenue | `GenerateAsyncRevenueReport` |
   | Fill rate | `GenerateAsyncFillRateReport` |
   | Missed opportunities | `GenerateAsyncMissedOpportunitiesReport` |
   | Attributed transactions | `GenerateAsyncAttributedTransactionsReport` |
   | Unfilled placements | `GenerateAsyncUnfilledPlacementsReport` |

   Dimensions and metrics in the request body select the output schema; filters constrain the
   rows.

2. **Prefer a sync report when the window is small.** `GenerateSyncCampaignsReport`,
   `GenerateSyncLineItemsReport` and `GenerateSyncRealTimePerformanceReport` return inline and
   save you two calls out of a 40/min budget.

3. **Poll status.** `GetAsyncExportStatus` — `GET /reports/{reportId}/status`. Poll with
   backoff, not on a tight loop; every poll spends reporting quota.

4. **Fetch output.** `GetAsyncExportOutput` — `GET /reports/{reportId}/output`. Output media
   type depends on the requested format: `application/json`, `text/csv`, `application/csv` or
   an XLSX sheet.

5. **Partner billing follows the same shape** with its own operations:
   `CreatePartnerBillingReportRequestV1` → `GetPartnerBillingReportStatusV1` →
   `GetPartnerBillingReportOutputV1`.

## Rules you must follow

- **Do not query for data that does not exist yet.** Criteo publishes the freshness windows:
  onsite activity lands in **6–8h**, offsite in **~24h**, initial attribution in **7–9h**, and
  final attribution within **74h**, with minor updates possible up to **120h**. Polling for
  yesterday's final attribution this morning burns quota and returns numbers that will move.
- **Cap reporting windows at four consecutive days.** Criteo's own guidance: spend and
  attribution stabilise within 72–74 hours, so longer pulls are usually wasted calls.
- **Batch, do not loop.** Pass up to **50** campaign or line-item IDs per call. Looping one id
  at a time is the fastest way to hit 429.
- **Do not enumerate every attribution setting.** Pull the settings your users actually look
  at; the combinatorial pull is the classic quota killer.
- **Poll politely.** On 429 back off exponentially — 1s, 2s, 4s. Read `x-ratelimit-reset`
  (Unix epoch seconds). There is no `Retry-After` header.
- **`dataCompleteThrough`.** The real-time performance report returns a `metadata` object with
  `dataCompleteThrough` — the latest time present in the streaming table, in the request
  timezone. Treat anything after it as incomplete rather than as zero.
- **Breaking change to watch:** in 2026-07, real-time performance renamed
  `billableImpressions` → `impressions` and `billableClicks` → `clicks`.

## Cross-references

- `rate-limits/criteo-rate-limits.yml` — the 40/min reporting cap and the bulk-call ceiling
- `changelog/criteo-changelog.yml` — the 2026-07 metric renames
- `errors/criteo-problem-types.yml` — `invalid-timespan` and the validation category
