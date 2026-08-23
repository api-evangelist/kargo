---
name: Schedule a Kargo shipment and validate it against what the dock sees
description: Create or upsert a shipment with its orders and expected pallet items in Kargo, then consume the pallet webhooks to verify that what physically moved through the dock matches what was expected.
api: openapi/kargo-document-intake-openapi.yml
operations:
  - createDocument
graphql_operations:
  - createShipmentAndOrder
  - pushMessages
generated: '2026-08-23'
method: generated
source: https://docs.kargo.ai/rest-api
---

# Schedule a Kargo shipment and validate it

Kargo's job is to compare what a customer *said* would move through a dock door against
what its cameras actually observed. This skill covers the customer half: getting the
expectation in, and reading the observation out.

## 1. Get a token

`POST https://mykargo.us.auth0.com/oauth/token` with
`{"client_id": ..., "client_secret": ..., "audience": "https://api.kargo.zone/public_graphql",
"grant_type": "client_credentials"}`.

Kargo issues the client_id and client_secret — there is no self-service signup. The token
is valid 24 hours. **Cache it.** Read `expires_in` and only request a new one when the
current token expires. Send it as `Authorization: Bearer <access_token>`.

## 2. Send the expected shipment — `createDocument`

`POST https://api.kargo.zone/v1/documents`

Required on every request: `business`, `facility`, `direction` (`INBOUND` or `OUTBOUND`).
Single-order requests also require top-level `orderNumber`; multi-order requests require
`shipmentNumber` and a non-empty `orders` array where each entry has its own
`orderNumber`.

Every item needs `lpn`, `sku`, `quantity` and `quantityUnit` when you are creating items
under the default strategy. A missing field returns `422`.

Set `Correlation-Id` on the request. Kargo echoes it back and it is the value to quote
when asking Kargo support to find a request in their logs.

## 3. Pick the strategy deliberately — this is the dangerous step

`orderItemUpdateStrategy` can be set at shipment level (applies to every order in the
document) or per order in `orders[]`, where the order-level value wins.

| Strategy | Effect | Safe to replay? |
|---|---|---|
| `OVERWRITE` | Replaces the whole item list. Anything not in the request is **removed**. | Content-safe, id-unsafe: rows are recreated with new ids. |
| `MERGE` | Updates the item matched on `lpn` + `sku`. | Yes. This is the idempotent one. |
| `APPEND` | Adds items, keeps existing ones. | **No.** A repeat containing an existing `lpn` + `sku` returns `409`. |
| `DELETE` | Removes the items matched on `lpn` + `sku`. | Yes. |

**If you supply `items` and omit the strategy, Kargo applies `OVERWRITE`.** Never send a
partial item list without an explicit strategy — you will silently delete the rest of the
order. If you are correcting one item, use `MERGE`. If you are adding one, use `APPEND`
and be ready for the `409`.

## 4. Read the response

`201` means something was created; `200` means everything already existed and was only
updated. Both return `created[]`, `updated[]`, `removed[]` and the resulting `shipment`.
Each entry is `{type, id, reference}` where `type` is `SHIPMENT`, `ORDER` or
`ORDER_ITEM`.

Reference resolution, as Kargo publishes it: shipment reference is `shipmentNumber`
(falling back to `orderNumber` if absent), order reference is `orderNumber`, order item
reference is `identifier` if provided, else `lpn`, else `sku`.

Persist the returned `id` values only if you can tolerate them changing — an `OVERWRITE`
replay issues new `ORDER_ITEM` ids for identical content.

## 5. Consume the observation

As each pallet passes a Kargo tower, Kargo POSTs a pallet event to your endpoint with
`kargoShipmentId`, `kargoPalletId`, `occurredAt`, `direction` (`LOADING` / `UNLOADING`),
`orders`, `dockId`, and the label fields configured for you (`LPN`, `SKUs`,
`ExpirationDate`, `LotNumber`, ...). Custom fields always arrive as **lists**, because a
label may carry several of a value.

Two things Kargo tells you to handle yourself:

- A pallet that is both loaded and unloaded produces **two** events. Cancel out the
  load/unload pair in your consumer logic.
- There is no published retry policy and no payload signature — only HTTP Basic. If you
  miss a delivery, recover it with the GraphQL `pushMessages` query, which returns all
  push messages since a timestamp for a business or facility. That is the only replay
  path Kargo offers.

Compare each observed `LPN` against the items you sent in step 2. A mismatch is the
finding — that is the entire product.

## Errors

All REST errors are RFC 9457 `application/problem+json` with `type` / `title` / `status`
/ `detail`. `type` is always `about:blank`; do not try to dereference it.

`401` re-auth. `403` your token is not scoped to that business/facility. `404` the
business or facility does not exist. `409` an `APPEND` collision or duplicate `lpn`+`sku`
in your payload — switch to `MERGE`. `415` set `Content-Type` to `application/json`,
`application/xml` or `text/xml`. `422` a bad enum, a wrong item shape for your strategy,
or a new order missing `business`/`facility`/`direction`.

## Limits of this API

No rate limits are published. No status page exists. There is no shipment or order
delete on any surface — only items and SKUs can be removed, and no reversal window is
stated anywhere, so treat every write as effectively permanent at the shipment level.
