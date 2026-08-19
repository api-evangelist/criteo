---
name: Fund a balance and launch a Retail Media campaign
description: Create a funded Balance on a Criteo Retail Media account, create a campaign against it, add an auction line item with keywords and promoted products, and verify it can spend.
api: openapi/criteo-retail-media-api-openapi.yml
api_version: '2026-07'
base_url: https://api.criteo.com/2026-07/retail-media
scopes:
  - RetailMedia_Accounts_Read
  - RetailMedia_Balance_Read
  - RetailMedia_Balance_Manage
  - RetailMedia_Campaign_Read
  - RetailMedia_Campaign_Manage
operations:
  - GetAccounts
  - GetPageOfBalancesV1
  - CreateBalanceByAccountId
  - AddFundsByAccountAndBalanceId
  - AppendCampaignsToBalanceV1
  - CreateCampaignsByAccountId
  - GetCampaignsByAccountId
  - CreateAuctionLineItem
  - GetAuctionLineItemsByCampaign
  - AddRemoveKeywords
  - SetKeywordBids
  - GetCampaignsByBalanceId
  - GetBalanceHistoryV1
generated: '2026-08-13'
method: generated
source: openapi/criteo-retail-media-api-openapi.yml (2026-07) + https://developers.criteo.com/criteo-apis/docs/api-error-codes
---

# Fund a balance and launch a Retail Media campaign

Criteo Retail Media will not spend money that is not first placed in a **Balance**. A campaign
with no balance attached is inert. Do these in order.

## Before you start

- Get a token: `POST https://api.criteo.com/oauth2/token` with `grant_type=client_credentials`,
  `client_id`, `client_secret`, form-encoded. **This endpoint is not in the OpenAPI** — it is
  documented only in prose. The token lives **900 seconds**; refresh before expiry rather than
  reacting to `authorization-token-expired`.
- Send `Authorization: Bearer <token>` on every call below.
- Your application must have the Balances and Campaign domains set to **Manage** in the
  developer portal. Permissions are fixed at app-configuration time; you cannot widen them at
  runtime.

## Steps

1. **Find the account.** `GetAccounts` — `GET /accounts`. Take the `accountId` you are
   entitled to operate. Requires `RetailMedia_Accounts_Read`.

2. **Check for an existing usable balance before creating one.** `GetPageOfBalancesV1` —
   `GET /accounts/{accountId}/balances`. Creating a duplicate balance is the most common
   avoidable mistake, and **Criteo has no idempotency key**, so a retried create is a second
   balance, not the same one.

3. **Create the balance if none fits.** `CreateBalanceByAccountId` —
   `POST /accounts/{accountId}/balances`. Requires `RetailMedia_Balance_Manage`.
   Record the returned balance id immediately; you cannot recover it by replaying the request.

4. **Add funds.** `AddFundsByAccountAndBalanceId` —
   `POST /accounts/{account-id}/balances/{balance-id}/add-funds`.
   **This is the money operation.** It is not idempotent and there is no request key. If it
   times out, do **not** resend — call `GetBalanceHistoryV1`
   (`GET /balances/{balanceId}/history`) and confirm whether the deposit landed.

5. **Create the campaign.** `CreateCampaignsByAccountId` —
   `POST /accounts/{account-id}/campaigns`. Requires `RetailMedia_Campaign_Manage`.

6. **Attach the campaign to the balance.** `AppendCampaignsToBalanceV1` —
   `POST /balances/{balanceId}/campaigns/append`. Until this succeeds the campaign cannot
   spend. Verify with `GetCampaignsByBalanceId`.

7. **Create an auction line item.** `CreateAuctionLineItem` —
   `POST /campaigns/{campaignId}/auction-line-items`. Confirm with
   `GetAuctionLineItemsByCampaign`.

8. **Add keywords and bids.** `AddRemoveKeywords` —
   `POST /line-items/{line-item-id}/keywords/add-remove`, then `SetKeywordBids`.
   Use `GetRecommendedKeywords` first if you want Criteo's suggestions.

9. **Verify.** `GetCampaignsByAccountId` and `GetCampaignsByBalanceId` should both show the
   campaign, and the balance should show the funds.

## Rules you must follow

- **No idempotency.** Criteo publishes no `Idempotency-Key` header. Every write in this flow
  is at-most-once. On a timeout, read back before retrying — never resend a create or a
  fund.
- **A 200 is not proof of success on bulk endpoints.** Bulk requests always return 200
  regardless of individual failures; failed entities appear in `warnings[]` and are silently
  excluded from processing. Parse `warnings[]` and `errors[]` and reconcile counts against
  what you submitted.
- **An empty search is 200, not 404.** You get
  `{"meta":{"totalItems":0,...},"data":[],"warnings":[],"errors":[]}`.
- **403 means consent, not credentials.** A 403 here usually means the advertiser revoked
  consent or the app's domain level is Read. Relaunch the OAuth2 flow; do not retry.
- **Rate limits.** 250 calls/min per application on client_credentials. Back off
  exponentially on 429 — 1s, then 2s, doubling. Read `x-ratelimit-reset`; there is no
  `Retry-After`.
- **Errors are RFC 7807-shaped inside a Criteo envelope**: `{"errors":[{traceId, type, code,
  instance, title, detail}]}`. Always log `traceId` — Criteo support needs it.

## Cross-references

- `conventions/criteo-conventions.yml` — pagination, async patterns, the bulk 200 trap
- `errors/criteo-problem-types.yml` — full error category and code list
- `scopes/criteo-scopes.yml` — the 22 real scope strings
- `rate-limits/criteo-rate-limits.yml` — limits and headers
