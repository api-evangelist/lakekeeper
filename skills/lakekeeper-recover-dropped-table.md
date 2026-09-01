---
name: lakekeeper-recover-dropped-table
description: >-
  Determine whether a dropped Iceberg table or view can still be recovered in Lakekeeper, and restore it inside the
  Warehouse's soft-delete window.
api: Lakekeeper Management API
generated: '2026-08-27'
method: generated
source: >-
  openapi/lakekeeper-management-api-openapi.yml, https://docs.lakekeeper.io/docs/latest/concepts/,
  conventions/lakekeeper-conventions.yml
operations:
  - get_warehouse
  - list_deleted_tabulars
  - undrop_tabulars
  - get_task_queue_config_tabular_expiration
  - get_task_queue_config_tabular_purge
  - list_tasks
  - get_warehouse_actions
  - tableExists
---

# Recover a dropped table or view

This is the one destructive action in Lakekeeper that can be taken back — and only sometimes. Establish *whether*
before you attempt *how*.

## 1. Read the Warehouse's delete profile first

`get_warehouse` (`GET /management/v1/warehouse/{warehouse_id}`).

Look at `TabularDeleteProfile`:

- `{"type": "hard"}` — **stop.** There is nothing to recover. The drop removed the object and, when
  `purgeRequested` was set, the data files too. No endpoint restores it.
- `{"type": "soft", "expiration-seconds": N}` — recovery is possible for `N` seconds after the drop.

The window is fixed **at the moment of the drop**, not read live. If someone changed the delete profile after the
table was dropped, the old value is what applies. `expiration-seconds` on the Warehouse today tells you the window
for tables dropped *from now on*.

## 2. Confirm the object is actually in the deleted state

`list_deleted_tabulars` (`GET /management/v1/warehouse/{warehouse_id}/deleted-tabulars`).

This is paginated with `pageToken` / `pageSize`; follow `next-page-token` until it is absent. Each
`DeletedTabularResponse` identifies the soft-deleted table or view.

If the object is not in this list and `tableExists` returns 404, it is gone — either the Warehouse was on `hard`,
or the expiration task has already run.

## 3. Restore it

`undrop_tabulars` (`POST /management/v1/warehouse/{warehouse_id}/deleted-tabulars/undrop`) with an
`UndropTabularsRequest`. Success is `204 No Content`.

## 4. Verify

`tableExists` (`HEAD /v1/{prefix}/namespaces/{namespace}/tables/{table}`) on the catalog plane, or `loadTable` if
you want the metadata back.

## Racing the expiration task

Soft-deleted objects are cleaned up by two background task queues: `tabular_expiration` (marks the window as
elapsed) and `tabular_purge` (removes the files). Read their configuration with
`get_task_queue_config_tabular_expiration` and `get_task_queue_config_tabular_purge`, and watch in-flight work with
`list_tasks` / `get_task_details`. Once purge has run there is no recovery.

## Related traps

- **A Namespace or Warehouse cannot be dropped while children exist — and soft-deleted tables count as children.**
  A drop that fails for no obvious reason is often this. There is deliberately no cascade-drop; the docs record it
  as planned for a future Management API release.
- **`403` may mean "does not exist".** Lakekeeper does not always return 404 for missing objects. Call
  `get_warehouse_actions` to separate a permission problem from an absent object.
- **Prevention beats recovery.** `set_table_protection`, `set_view_protection`, `set_namespace_protection`,
  `set_generic_table_protection` and `set_warehouse_protection` cause a delete to be refused outright.

Reference: https://docs.lakekeeper.io/docs/latest/concepts/#soft-deletion
