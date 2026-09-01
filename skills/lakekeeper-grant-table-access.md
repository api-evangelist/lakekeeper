---
name: lakekeeper-grant-table-access
description: >-
  Grant a user or service principal access to a Lakekeeper table or namespace using Roles, and verify the grant
  took effect by asking the API what the principal may actually do.
api: Lakekeeper Management API
generated: '2026-08-27'
method: generated
source: >-
  openapi/lakekeeper-management-api-openapi.yml, https://docs.lakekeeper.io/docs/latest/authorization/,
  https://docs.lakekeeper.io/docs/latest/authorization-openfga/
operations:
  - whoami
  - search_user
  - list_user
  - create_role
  - list_roles
  - search_role
  - add_role_members
  - list_role_members
  - list_role_transitive_members
  - list_user_roles
  - list_user_transitive_roles
  - get_role_actions
  - get_table_actions
  - get_namespace_actions
  - get_warehouse_actions
  - remove_role_member
---

# Grant access to a table or namespace

Lakekeeper's permission model is relationship-based and hierarchical: grants made at a Warehouse or Namespace
level are inherited downwards to the tables and views inside. That is why OpenFGA (a Zanzibar-style ReBAC engine)
is the default authorizer rather than a flat role table.

**Grant to Roles, not to Users.** A Role is a first-class principal that can be granted permissions and assumed,
and Roles nest — which is what makes a permission model maintainable at more than a handful of people.

## 1. Identify the principal

`whoami` (`GET /management/v1/whoami`) confirms who *you* are. `search_user`
(`POST /management/v1/search/user`) or `list_user` finds the target. Users mirrored from an OIDC provider carry
ids of the form `oidc~<subject>`.

## 2. Find or create the Role

`list_roles` (`GET /management/v1/role`) or `search_role` (`POST /management/v1/search/role`); otherwise
`create_role` (`POST /management/v1/role`) with the Project the Role belongs to.

If your organisation resolves group membership from an identity provider (Entra ID, Okta, LDAP), the Role can be
bound to that source system via `provider-id` + `source-id` — see `update_role_source_system`. Do not hand-manage
members of a provider-managed Role; the API guards against writes in provider-managed namespaces.

## 3. Add the member

`add_role_members` (`POST /management/v1/role/{role_id}/members`). Members can be Users **or other Roles** —
role-in-role nesting is supported, with the depth bounded at write time.

## 4. Grant the Role permission on the object

Permissions live in the configured Authorizer, not on the object. With OpenFGA enabled, the Management API exposes
39 `permissions-openfga` operations for reading and writing grants on servers, projects, warehouses, namespaces,
tables and views. With the OPA bridge, policy lives in Rego outside Lakekeeper. With Lakekeeper+, Cedar policies
serve the same role.

## 5. Verify — do not assume

This is the step people skip, and it is the one that matters on this API.

Call the object's `*/actions` endpoint **as the target principal**:

- `get_table_actions` (`GET /management/v1/warehouse/{warehouse_id}/table/{table_id}/actions`)
- `get_namespace_actions` (`GET /management/v1/warehouse/{warehouse_id}/namespace/{namespace_id}/actions`)
- `get_warehouse_actions` (`GET /management/v1/warehouse/{warehouse_id}/actions`)

These return what the caller may actually do. Use `list_user_transitive_roles` and
`list_role_transitive_members` to see the effective membership after nesting, rather than the direct membership.

**Note the deprecated forms.** `get_table_access_by_id`, `get_namespace_access_by_id`, `get_view_access_by_id`,
`get_warehouse_access_by_id`, `get_role_access_by_id`, `get_project_access`, `get_project_access_by_id` and
`get_server_access` are all marked `deprecated: true` in the spec. Every one has a live `*_actions` replacement.
Use the replacement.

## Revoking

`remove_role_member` (`DELETE /management/v1/role/{role_id}/members/{member_type}/{member_id}`). Re-run the
`*/actions` check afterwards — inherited and transitive grants mean removing one membership does not always remove
access.

## Two traps

- **`403` may mean the object does not exist.** Lakekeeper does not always 404 for missing objects, so a 403 with
  seemingly correct grants is frequently a wrong id rather than a wrong permission.
- **`501 Not Implemented`** is declared on some permission operations — the surface exists in the contract but the
  configured backend does not implement it (for example when the AllowAll authorizer is in use).

Reference: https://docs.lakekeeper.io/docs/latest/authorization/
