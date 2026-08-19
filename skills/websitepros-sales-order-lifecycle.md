---
name: websitepros-sales-order-lifecycle
description: >-
  Create, find, update and delete a sales order in the Web.com International Platform API, with the
  three-header authentication the gateway requires and the retry hazards it does not protect you
  from. Use when working a Web.com partner sales pipeline through the API.
api: Web.com International Platform API
base_url: https://api.nts.web.com
spec: openapi/websitepros-international-platform-openapi-derived.yml
operations:
  - createSalesOrder
  - listSalesOrders
  - getSalesOrderById
  - updateSalesOrder
  - deleteSalesOrder
generated: '2026-08-13'
method: generated
source: >-
  openapi/websitepros-international-platform-openapi-derived.yml +
  collections/websitepros-international-platform.postman_collection.json
---

# Web.com sales order lifecycle

## Before you start

You cannot self-onboard. Web.com grants access only after the International team reviews your
developer-portal registration, and production and development are separate registrations. You need
three things in hand before any call:

- an Azure API Management subscription key for the environment you are calling,
- a Microsoft Entra ID access token,
- your Tenant Id, issued by Web.com when the subscription is activated.

Get the token with the client-credentials grant. Post
`grant_type=client_credentials`, `client_id`, `client_secret` and `resource=api://intl.web.com`
as `application/x-www-form-urlencoded` to
`https://login.microsoftonline.com/03fbebc8-de8a-4428-b573-4c4903610dac/oauth2/token`.

Every sales-order call then carries all three headers together:

```
Content-Type: application/json
Ocp-Apim-Subscription-Key: <your subscription key>
Authorization: Bearer <entra token>
x-nts-tenant-id: <your tenant id>
```

Dropping any one of them gets you `401` with
`{ "statusCode": 401, "message": "Access denied due to missing subscription key. ..." }` and a
`WWW-Authenticate: AzureApiManagementKey` header. The message names only the subscription key even
when the missing credential is one of the other two, so do not trust it to tell you which header
you forgot.

Note the header split: sales orders use `x-nts-tenant-id`, but the domain, SSO and service-order
operations use `tenant-name` instead. They are not interchangeable.

## Create a sales order — `createSalesOrder`

`POST /sales-orders/v1`

The body is `{ "traceId": "...", "salesOrder": { ... } }`. Set your own `traceId` for correlation.

Inside `salesOrder`:

- `customer` — for a **new** customer send the full object with `id` set to the empty string `""`.
  For an **existing** customer send its `id`. Empty string is the "create me" sentinel, not `null`
  and not omission.
- `account` — same rule: `id: ""` creates a new account, an `id` places the products on an
  existing one.
- `contacts[]` — each with `roles[]`. Roles seen in Web.com's own examples: `Owner`, `Marketing`,
  `Finance`, `Legal`, `Accounting`, `General Manager`. Web.com publishes no closed enumeration, so
  treat anything else as unverified.
- `products[]` — name, quantity, `discountPercent`, `serviceAgreementLength` and
  `serviceAgreementUnit`. An empty `products: []` is valid; Web.com documents it explicitly.
- `paymentSummary` — `currencyCode`, `invoiceDeliveryMethod`, `paymentTerms`, `paymentType`.
- `status` — `pipelinePhase`, `salesPartner`, `salesRep`.

**Retry hazard.** There is no `Idempotency-Key` on this API. `traceId` is a correlation id, and
nothing Web.com publishes says a repeated `traceId` suppresses a duplicate. If the call times out,
do **not** blind-retry — call `listSalesOrders` filtered by the account name first and check
whether the order landed.

**Field-name hazard.** Web.com's published create example spells the line-item quantity `quantity`
while its published update example spells it `qty`. The documentation does not say which the API
accepts. Send `quantity` on create, `qty` on update — matching the examples — and verify with a
read-back rather than assuming either is universal.

## Find sales orders — `listSalesOrders`

`GET /sales-orders/v1?page=1&pageSize=10&sortBy=createdOnUtc`

- `page` is 1-based; `pageSize` sets the page length.
- `sortBy` takes a field name; prefix it with `-` to reverse, e.g. `sortBy=-createdOnUtc`.
- `processed=false` returns only orders that have not been processed yet.
- `criteria` is one overloaded free-text filter. Web.com's own examples use the same parameter to
  filter by sales partner (`criteria=Web.com - UK`), by sales rep and by account name, with no
  field selector saying which. URL-encode the value.

Web.com does not publish the list response envelope, so do not assume a total count, a page count
or cursor links exist. Read the first page and inspect what actually comes back.

## Read one — `getSalesOrderById`

`GET /sales-orders/v1/{salesOrderId}?traceId=...`

## Update — `updateSalesOrder`

`PUT /sales-orders/v1/{salesOrderId}?traceId=...`

This is a merge, not a strict replace: properties you omit keep their existing values, properties
you send overwrite. Send only the branches you intend to change.

## Delete — `deleteSalesOrder`

`DELETE /sales-orders/v1/{salesOrderId}`

Destructive and not reversible through this API. Confirm with a human before calling it.

## Reading errors

The only two envelopes anyone can observe anonymously are:

```json
{ "statusCode": 401, "message": "Access denied due to missing subscription key. Make sure to include subscription key when making requests to an API." }
{ "statusCode": 404, "message": "Resource not found" }
```

There is no stable `code` or `type` field to branch on, and no correlation id in the body — only
the `traceId` you supplied. A `404` from this gateway can mean the route is not deployed, or that
your subscription does not cover that product, or that the path is wrong; you cannot tell them
apart from the response.
