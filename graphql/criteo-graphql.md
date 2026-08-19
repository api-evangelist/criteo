# Criteo GraphQL — NOT A CRITEO CONTRACT

> **Read this first.** Criteo publishes **no GraphQL endpoint.** The schema in
> `criteo-schema.graphql` is a *conceptual* model that an earlier round derived from Criteo's
> REST surface. It is a thinking aid, not something you can query, and nothing in this repo
> should treat it as a Criteo-served contract.

**Endpoint:** none — verified 2026-08-13
**Status:** derived / illustrative
**Method:** derived from the REST API. Never fetched from Criteo, never introspected.

## What was actually checked (2026-08-13)

GraphQL introspection was attempted as part of STEP 0b contract discovery against every
Criteo host — `api.criteo.com`, `developers.criteo.com`, `mcp.criteo.com`, `www.criteo.com`.
No `/graphql` surface exists on any of them. Criteo's own developer portal, its `llms.txt`
docs index (444 lines) and its published Agent Skill all describe a REST API exclusively.

The `apis.yml` pointer of `type: GraphQL` that used to reference this file was **removed** in
the 2026-08-13 enrichment pass, because that pointer asserted Criteo ships a GraphQL API and
Criteo does not. `conformance/criteo-conformance.yml` records `graphql: conforms: false` with
the same evidence.

## The real machine-readable contracts

Criteo publishes three live OpenAPI 3.0.1 documents, harvested verbatim into
`openapi/_original/`:

| Product | Version | Operations | Source |
|---|---|---|---|
| Retail Media | 2026-07 | 115 | https://api.criteo.com/2026-07/retailmedia/open-api-specifications.json |
| Marketing Solutions | 2026-07 | 96 | https://api.criteo.com/2026-07/marketingsolutions/open-api-specifications.json |
| Commerce Grid | 2026-01 | 8 | https://api.criteo.com/2026-01/commercegrid/open-api-specifications.json |

Every version is addressable at
`https://api.criteo.com/{version}/{marketingsolutions|retailmedia|commercegrid}/open-api-specifications.json`.

- Documentation: https://developers.criteo.com/
- Specification index: https://developers.criteo.com/criteo-apis/docs/criteo-api-swagger
- Reference: https://developers.criteo.com/retail-media/reference
