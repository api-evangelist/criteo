---
name: Build an audience segment and target a line item or ad set with it
description: Create a Criteo audience segment, load a contact list into it, size it, and attach it as targeting on a Retail Media line item or a Marketing Solutions ad set — across all three services that expose audiences.
api: openapi/criteo-retail-media-api-openapi.yml
also_uses:
  - openapi/criteo-marketing-solutions-api-openapi.yml
  - openapi/criteo-commerce-grid-api-openapi.yml
api_version: '2026-07'
scopes:
  - RetailMedia_Audience_Read
  - RetailMedia_Audience_Manage
  - RetailMedia_Campaign_Manage
  - MarketingSolutions_Audience_Read
  - MarketingSolutions_Audience_Manage
  - MarketingSolutions_Campaign_Manage
  - CommerceGrid_Segment_Read
  - CommerceGrid_Segment_Manage
operations:
  - bulkCreateAudienceSegments
  - bulkUpdateAudienceSegments
  - bulkDeleteAudienceSegments
  - searchAudienceSegments
  - searchAudiences
  - AddRemoveContactListByAudienceSegment
  - ClearContactListByAudienceSegment
  - getAudienceSegmentContactListStatistics
  - GetAudienceTargetsByLineItemId
  - PutAudienceTargetsByLineItemId
  - AppendAudienceTargetsByLineItemId
  - DeleteAudienceTargetsByLineItemId
  - CreateAudienceSegments
  - UpdateAudienceSegments
  - ComputeAudienceSegmentsSizes
  - EstimateAudienceSegmentsSizes
  - SearchAudienceSegments
  - UpdateContactListByAudienceSegment
  - DeleteContactListByAudienceSegment
  - UpdateAdSetAudience
generated: '2026-08-13'
method: generated
source: openapi/criteo-*-api-openapi.yml (2026-07 / 2026-01)
---

# Build an audience segment and target a line item or ad set with it

Audiences exist in **all three** Criteo services — Retail Media, Marketing Solutions and
Commerce Grid — with separate endpoints, separate scopes and different operationId casing.
Pick the service that owns the campaign you are targeting; do not assume the segment crosses
over.

## Which service

| Service | Create segments | Search | Scope |
|---|---|---|---|
| Retail Media | `bulkCreateAudienceSegments` | `searchAudienceSegments` | `RetailMedia_Audience_Manage` |
| Marketing Solutions | `CreateAudienceSegments` | `SearchAudienceSegments` | `MarketingSolutions_Audience_Manage` |
| Commerce Grid | (Cg segment endpoints) | — | `CommerceGrid_Segment_Manage` |

Note the casing split: Retail Media audience operations are camelCase starting lowercase
(`bulkCreateAudienceSegments`), Marketing Solutions are PascalCase (`CreateAudienceSegments`).
They are different operations, not aliases.

## Steps — Retail Media

1. **Look before you create.** `searchAudienceSegments` —
   `POST /accounts/{account-id}/audience-segments/search`. Requires
   `RetailMedia_Audience_Read`.

2. **Create the segment.** `bulkCreateAudienceSegments` —
   `POST /accounts/{account-id}/audience-segments/create`. This is a **bulk** endpoint: it
   creates all segments with a valid configuration and returns the full segments; the ones it
   could not create come back as errors inside a **200** response.

3. **Load the contact list.** `AddRemoveContactListByAudienceSegment` —
   `POST /audience-segments/{audience-segment-id}/contact-list`. Use
   `ClearContactListByAudienceSegment` to empty it rather than deleting and recreating the
   segment.

4. **Check it is big enough to serve.** `getAudienceSegmentContactListStatistics` —
   `GET /audience-segments/{audience-segment-id}/contact-list/statistics`.

5. **Target the line item.** Read current targeting with `GetAudienceTargetsByLineItemId`,
   then either replace it wholesale with `PutAudienceTargetsByLineItemId` or add to it with
   `AppendAudienceTargetsByLineItemId`. Remove with `DeleteAudienceTargetsByLineItemId`.
   Requires `RetailMedia_Campaign_Manage` — targeting is a Campaign-domain write, **not** an
   Audience-domain one. Getting this wrong is the most common 403 in this flow.

## Steps — Marketing Solutions

1. `SearchAudienceSegments` to check, `CreateAudienceSegments` to create.
2. `UpdateContactListByAudienceSegment` (PATCH) to load members;
   `DeleteContactListByAudienceSegment` (DELETE) to clear.
3. Size it: `EstimateAudienceSegmentsSizes` for a segment that does **not** exist yet,
   `ComputeAudienceSegmentsSizes` for ones that do. Both are `..._Audience_Manage`, not Read —
   sizing is treated as a write-level operation.
4. Attach to the ad set: `UpdateAdSetAudience` —
   `PUT /ad-sets/{ad-set-id}/audience`. Requires `MarketingSolutions_Campaign_Manage`.
5. Browse Criteo-provided segments with `GetAudienceSegmentsInMarketBrands` and
   `GetAudienceSegmentsInMarketInterests`.

## Rules you must follow

- **Every create/update here is a bulk endpoint that returns 200 on partial failure.** The
  response carries the entities it managed to write plus errors/warnings for the ones it did
  not. Reconcile the returned count against the submitted count on every call — a blind 200
  check will report success for a segment that was never created.
- **No idempotency.** A retried `bulkCreateAudienceSegments` creates a second set of segments.
  Search first, and on a timeout search again rather than resending.
- **Sizing is rate-limit-sensitive.** `Compute*Sizes` and `Estimate*Sizes` are analytical; do
  not call them in a loop per segment when the bulk form accepts a list.
- **PII.** Contact lists carry customer identifiers. Do not log request bodies for these
  operations.
- **403 vs empty result.** A 403 means the account does not exist or consent is missing. A
  valid but unmatched search returns 200 with `totalItems: 0`.

## Cross-references

- `scopes/criteo-scopes.yml` — Audience vs Campaign domain scopes and why targeting needs Campaign
- `data-model/criteo-data-model.yml` — AudienceSegment → ContactList, LineItem → Targeting
- `errors/criteo-problem-types.yml` — the bulk-200 silent-failure mode
