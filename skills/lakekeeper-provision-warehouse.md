---
name: lakekeeper-provision-warehouse
description: >-
  Bootstrap a fresh Lakekeeper deployment and provision a Warehouse with a storage profile, a soft-delete policy
  and protection, then verify it is reachable by query engines.
api: Lakekeeper Management API
generated: '2026-08-27'
method: generated
source: openapi/lakekeeper-management-api-openapi.yml, https://docs.lakekeeper.io/docs/latest/bootstrap/
operations:
  - bootstrap
  - get_server_info
  - get_server_actions
  - create_project
  - list_projects
  - create_warehouse
  - get_warehouse
  - list_warehouses
  - update_warehouse_delete_profile
  - set_warehouse_protection
  - get_warehouse_actions
  - getConfig
---

# Provision a Lakekeeper Warehouse

Base URL is your own deployment — Lakekeeper is self-hosted, so there is no vendor host. The Management API is
served under `/management/v1`. Authenticate with a bearer JWT from the OIDC provider the deployment is configured
against; there is no API-key path.

## 1. Confirm the server is up and bootstrapped

Call `get_server_info` (`GET /management/v1/info`). It returns `server-id` and `default-project-id`.

If the server has never been bootstrapped, call `bootstrap` (`POST /management/v1/bootstrap`) once. **This runs
exactly once per catalog by design.** If it has already run and you genuinely need to run it again, that is an
operator action outside the API: `lakekeeper reopen-bootstrap` on the server binary.

Before assuming you may do any of this, call `get_server_actions` (`GET /management/v1/server/actions`) — it tells
you what the current principal is permitted to do. This matters more here than on most APIs; see step 5.

## 2. Choose the Project

Projects are the top-level tenant boundary. `list_projects` (`GET /management/v1/project-list`) enumerates them;
`create_project` (`POST /management/v1/project`) makes a new one.

Select the Project on every subsequent call with the `x-project-id` request header. Omit it and the server's
default project is used — which is fine for a single-tenant deployment and a silent mistake in a multi-tenant one.

## 3. Create the Warehouse

`create_warehouse` (`POST /management/v1/warehouse`) with a `CreateWarehouseRequest`. It carries the
`storage-profile` and an optional `storage-credential`.

Two rules from the provider's own docs that are not enforceable by the schema:

- **Never share an object-store location between Warehouses.** Vended credentials are prefix-scoped to the
  Warehouse location, so a shared prefix leaks data across tenants.
- **A Warehouse's Project cannot be changed after creation.** Get it right the first time.

The storage configuration is validated server-side at creation, so a bad profile fails here rather than later at
first write. On failure you get the Iceberg error envelope: `{"error":{"message","type","code","stack"}}` — read
`error.message`, because the Management API declares only a blanket `4XX` and will not tell you the status shape
in advance.

## 4. Set the delete profile before anyone writes data

`update_warehouse_delete_profile` (`POST /management/v1/warehouse/{warehouse_id}/delete-profile`).

`TabularDeleteProfile` is a `oneOf`:

- `{"type": "hard"}` — a dropped table is gone immediately. **No reversal path exists.**
- `{"type": "soft", "expiration-seconds": N}` — a dropped table can be restored with `undrop_tabulars` for `N`
  seconds.

**Do this before the Warehouse is in use.** The expiration delay is fixed at the moment each table is dropped, so
switching to `soft` later does nothing for tables already dropped under `hard`.

Optionally call `set_warehouse_protection` (`POST /management/v1/warehouse/{warehouse_id}/protection`) so the
Warehouse itself cannot be deleted by accident. The same protection mechanism exists per namespace, table, view
and generic table.

## 5. Verify

- `get_warehouse` (`GET /management/v1/warehouse/{warehouse_id}`) — confirm the storage profile, delete profile,
  activation status and protection flag are what you set.
- `get_warehouse_actions` (`GET /management/v1/warehouse/{warehouse_id}/actions`) — confirm the principal that
  will actually be used by engines can do what it needs to.
- `getConfig` (`GET /v1/config` on the catalog plane) — the engine-facing handshake. Its response also carries
  `idempotency-key-lifetime` when idempotency is enabled, which is how a client discovers it can safely retry.

## The one failure everybody hits

**Lakekeeper does not always return 404 for objects that do not exist.** If you get a `403` while holding what
look like correct grants, the object very likely is not there. This is deliberate anti-enumeration behaviour. Call
the relevant `*/actions` endpoint before concluding that permissions are wrong.

Reference: https://docs.lakekeeper.io/docs/latest/gotchas/
