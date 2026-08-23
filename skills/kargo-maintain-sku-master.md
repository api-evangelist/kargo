---
name: Maintain the Kargo SKU master
description: Keep Kargo's SKU reference data current so the vision system can reason about pallet geometry, and read it back to reconcile against your own item master.
api: openapi/kargo-document-intake-openapi.yml
operations:
  - upsertSkuMaster
  - getSkuMaster
graphql_operations:
  - upsertSKUs
  - deleteSKUs
generated: '2026-08-23'
method: generated
source: https://api.kargo.zone/v1/docs/openapi.yaml
---

# Maintain the Kargo SKU master

The SKU master is reference data keyed on `business` + `facility` + `sku_id`. Kargo's
cameras use the pallet geometry on it — `cases_per_pallet`, `cases_per_layer`,
`layers_per_pallet` — to reason about a stack, so a stale SKU master degrades the vision
results rather than producing an obvious error.

## Write — `upsertSkuMaster`

`POST https://api.kargo.zone/v1/sku_master` with `business`, `facility` and a `skus`
array. Each entry requires `sku_id`; the rest is optional: `description`,
`unit_of_measure`, `unit_of_weight`, `alternate_sku_ids`, `cases_per_pallet`,
`cases_per_layer`, `layers_per_pallet`, `sku_metadata`.

This is a true upsert on `sku_id` — safe to replay. The response returns
`numRecordsAdded`, `numRecordsUpdated` and the resulting `skus`, so you can assert on the
counts rather than diffing.

Two `422` traps worth guarding before you send: **duplicate `sku_id` values in the same
request**, and wrong field types. Deduplicate your batch client-side.

Partial updates work — send `sku_id` plus only the fields you are changing. The published
spec carries a `partialUpdate` example for exactly this.

XML is accepted (`application/xml` or `text/xml`) if your item master emits it.

## Read — `getSkuMaster`

`GET https://api.kargo.zone/v1/sku_master?business=...&facility=...`

`business` and `facility` are required. Optional: `sku_id` to fetch one, and `limit` /
`offset` for pagination. The response carries `skus`, `total`, `limit`, `offset` and
`next_offset` — page by following `next_offset` until it stops advancing.

An invalid `limit` or `offset` returns `422`, not a clamped result.

## Delete — not on this API

There is no REST delete for SKU master records. The reversal path is the GraphQL
`deleteSKUs` mutation at `https://api.kargo.zone/public_graphql`, on the surface Kargo
calls legacy. If your integration is REST-only, you have no way to remove a SKU. Plan for
that before you load a bad batch.

Kargo publishes no window on `deleteSKUs` — no statement of how long after an upsert a
delete remains possible. Do not assume one.

## Auth

Same Auth0 client-credentials bearer token as every other Kargo surface. `403` means the
token is not permitted for that business/facility; `404` means the facility does not
exist.
