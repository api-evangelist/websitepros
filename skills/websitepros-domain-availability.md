---
name: websitepros-domain-availability
description: >-
  Check whether a domain name is available for registration through the Web.com International
  Platform API, and generate available-name suggestions from partial words across a set of TLDs.
  Use when qualifying a domain for a Web.com customer before opening a sales or service order.
api: Web.com International Platform API
base_url: https://api.nts.web.com
spec: openapi/websitepros-international-platform-openapi-derived.yml
operations:
  - checkDomainAvailability
  - spinDomainSuggestions
generated: '2026-08-13'
method: generated
source: >-
  openapi/websitepros-international-platform-openapi-derived.yml +
  collections/websitepros-international-platform.postman_collection.json
---

# Web.com domain availability

## Headers

The domain operations use `tenant-name`, **not** the `x-nts-tenant-id` header the sales-order
operations use:

```
Content-Type: application/json
Ocp-Apim-Subscription-Key: <your subscription key>
Authorization: Bearer <entra token>
tenant-name: <your tenant name>
```

The bearer token comes from the client-credentials grant against
`https://login.microsoftonline.com/03fbebc8-de8a-4428-b573-4c4903610dac/oauth2/token` with
`resource=api://intl.web.com`.

## Check one name — `checkDomainAvailability`

`GET /api/domain/check/{domain}`

The domain goes in the path, fully qualified, e.g. `/api/domain/check/test.com`. Web.com does not
publish the response body, so read what comes back rather than assuming a shape.

Verified live on 2026-08-13: this route answers `401` with
`WWW-Authenticate: AzureApiManagementKey realm="https://api.nts.web.com/api/domain"` to an
unauthenticated caller, which confirms it is deployed and gated as documented.

## Suggest names — `spinDomainSuggestions`

`POST /api/domain/spin`

```json
{
  "subdomain": "test",
  "extensions": ["com", "net", "org"]
}
```

`subdomain` is the partial word or words to build from; `extensions` is the list of TLDs to test.

**Caveat.** Unlike its sibling, this documented path returned `404 Resource not found` rather than
a `401` challenge on the production gateway when probed anonymously on 2026-08-13. That may mean
the route is scoped to a subscription, or it may mean it is not deployed on production. Treat a
404 here as ambiguous and confirm with Web.com rather than concluding your request was malformed.

## Sequencing

Both operations are read-only and safe to call repeatedly — there is nothing to make idempotent.
Neither returns pricing or reserves a name; use them to qualify a domain before you create a
sales order with `createSalesOrder` (see `websitepros-sales-order-lifecycle`).
