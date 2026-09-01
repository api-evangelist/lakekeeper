---
name: lakekeeper-read-table-with-vended-credentials
description: >-
  Connect to a Lakekeeper catalog as a client, load Iceberg table metadata, and obtain short-lived prefix-scoped
  storage credentials to read the data files — without ever holding a long-lived object-store key.
api: Lakekeeper Iceberg REST Catalog API
generated: '2026-08-27'
method: generated
source: >-
  openapi/lakekeeper-catalog-api-openapi.yml, openapi/lakekeeper-generic-table-api-openapi.yml,
  https://docs.lakekeeper.io/docs/latest/engines/, https://docs.lakekeeper.io/docs/latest/storage/
operations:
  - getConfig
  - listNamespaces
  - listTables
  - loadTable
  - tableExists
  - loadCredentials
  - planTableScan
  - fetchPlanningResult
  - cancelPlanning
  - fetchScanTasks
  - listViews
  - loadView
  - list_generic_tables
  - load_generic_table
  - load_generic_table_credentials
---

# Read a table through Lakekeeper

Credential vending is the point of this API. Lakekeeper holds the object-store credential; a client gets a
short-lived, prefix-scoped credential for exactly the table it asked for. Never configure an engine with a
long-lived S3/GCS/ADLS key when Lakekeeper is in the path — that defeats the entire access-control model.

Most of the time you do not call these operations by hand: Spark, Trino, StarRocks, Flink, DuckDB, Athena and
PyIceberg all speak Iceberg REST natively and only need configuration. See
https://docs.lakekeeper.io/docs/latest/engines/.

## 1. Handshake

`getConfig` (`GET /v1/config`) — the first call any Iceberg REST client makes. Pass the `warehouse` query
parameter to select the Warehouse. The response carries the catalog defaults and overrides your client should
apply.

It also carries `idempotency-key-lifetime` when idempotency is enabled. **Read it.** That field is how you
discover both that you may send an `Idempotency-Key` header on mutating requests and how long a key remains valid
(default `PT30M`). Lakekeeper replays the original response for a repeated key rather than applying the change
twice.

## 2. Find the table

`listNamespaces` (`GET /v1/{prefix}/namespaces`) — supports `parent` for nested namespaces, and `pageToken` /
`pageSize` pagination.

`listTables` (`GET /v1/{prefix}/namespaces/{namespace}/tables`) and `listViews` for views.

`tableExists` (`HEAD .../tables/{table}`) is the cheap existence check.

## 3. Load metadata

`loadTable` (`GET /v1/{prefix}/namespaces/{namespace}/tables/{table}`).

`loadTable` accepts an `If-None-Match` request header and can answer `304 Not Modified`, so a client that caches
table metadata should send the ETag it holds rather than refetching the whole document. `snapshots` controls how
much snapshot history comes back.

For views, `loadView`. Lakekeeper supports DEFINER and INVOKER security models on views, so what a view resolves
to can depend on who is asking.

## 4. Get credentials

`loadCredentials` (`GET /v1/{prefix}/namespaces/{namespace}/tables/{table}/credentials`).

The returned credential is short-lived and scoped to the table's storage prefix. Treat it as a secret with a
lifetime measured in minutes: do not log it, do not cache it beyond its expiry, and re-request rather than
refreshing it yourself.

Warehouses backed by a KMS-encrypted bucket advertise `s3.sse.type=kms` with the configured `aws-kms-key-arn`, so
writes through vended credentials use the customer's own key. Since Iceberg 1.11, Lakekeeper also emits
`signer.uri` / `signer.endpoint` properties for clients that use remote signing rather than vended credentials.

## 5. Scan planning (optional, server-side)

For large scans the catalog can plan on the server:

1. `planTableScan` (`POST .../tables/{table}/plan`) starts a plan and returns a `plan-id`.
2. `fetchPlanningResult` (`GET .../plan/{plan-id}`) polls it.
3. `fetchScanTasks` (`POST .../tables/{table}/tasks`) retrieves the tasks.
4. `cancelPlanning` (`DELETE .../plan/{plan-id}`) abandons it — **use this** if you stop early, so the server does
   not keep planning work you will not consume.

## Non-Iceberg tables

Lance, Delta, CSV and Parquet tables live in the same Namespaces and use the Data API at `/lakekeeper/v1`:
`list_generic_tables`, `load_generic_table`, and `load_generic_table_credentials` — the same vending model, a
different plane. The official `pylakekeeper` Python client wraps exactly this surface (and only this surface — it
is deliberately not a general Iceberg REST client).

## Errors worth handling

- `419` — credentials expired mid-operation. Refresh the token and retry. This is an Iceberg-REST-specific status
  most HTTP clients have no opinion about.
- `409` — concurrent modification on a commit. Reload metadata and retry, with an `Idempotency-Key` so the retry
  cannot double-apply.
- `503` — may carry `Retry-After` when an admission gate fails closed because an upstream it depends on is
  unreachable. Honour it.
- `403` — may mean the object does not exist rather than that you lack permission. See
  https://docs.lakekeeper.io/docs/latest/gotchas/.

The error body is the Iceberg wrapper `{"error":{"message","type","code","stack"}}` with media type
`application/json` — **not** RFC 9457 `application/problem+json`.
