---
name: Manage SpringServe domain block and allow lists
description: Create a domain targeting list, bulk-load domains into it, replace its contents atomically, and attach it to the resources it should target.
api: openapi/rubicon-project-springserve-v1-openapi.yml
operations: [domain_lists_get, domain_lists_post, domain_lists_id_get, domain_lists_id_patch, domain_lists_id_domains_get, domain_lists_id_domains_bulk_create_post, domain_lists_id_domains_bulk_replace_post, domain_lists_id_domains_bulk_delete_delete, domain_lists_id_resource_type_get, domain_lists_id_resource_type_post, domain_lists_id_duplicate_post, domain_lists_id_delete]
generated: '2026-08-12'
method: generated
source: openapi/rubicon-project-springserve-v1-openapi.yml (components.examples.domain_list_create_simple)
---

# Manage SpringServe domain block and allow lists

Domain lists are the brand-safety and inventory-quality control surface: a named list of
domains, attached to supply or demand resources as an include or exclude. The same shape
repeats across the other targeting lists (app bundle, app name, geo, ISP, postal code, city,
state, country, metro area, deal id, supply name) — learn it once here.

Authenticate and set the account context first.

## 1. Create the list — `domain_lists_post`

`POST /api/v1/domain_lists`. The provider's published minimal example is just:

```json
{"name": "Domain List Name"}
```

The list is created empty; the domains go in separately.

## 2. Load domains

Three distinct semantics — pick deliberately:

| Operation | `operationId` | Semantics |
|---|---|---|
| `POST /api/v1/domain_lists/{id}/domains/bulk_create` | `domain_lists_id_domains_bulk_create_post` | **Add** to what is there |
| `POST /api/v1/domain_lists/{id}/domains/bulk_replace` | `domain_lists_id_domains_bulk_replace_post` | **Replace** the whole list |
| `DELETE /api/v1/domain_lists/{id}/domains/bulk_delete` | `domain_lists_id_domains_bulk_delete_delete` | **Remove** the named domains |

File-upload variants exist for each: `domain_lists_id_domains_file_bulk_create_post`,
`domain_lists_id_domains_file_bulk_replace_post`,
`domain_lists_id_domains_file_bulk_delete_delete`. Use these for lists of any real size —
they avoid paging a very large JSON body through a 240 req/min account budget.

**Prefer `bulk_replace` for anything driven from an external source of truth.** It is the
only one of the three whose outcome does not depend on the prior state, which matters
because this API has no idempotency key: a `bulk_create` that times out and is retried
double-adds, while a `bulk_replace` retried converges on the same list.

## 3. Read back — `domain_lists_id_domains_get`

`GET /api/v1/domain_lists/{id}/domains`. Standard list contract (`page`, `per` up to 1000,
`search`, `sort`). Always read back after a bulk write — there is no write receipt and no
transaction id to reconcile against.

## 4. Attach it to targeting — `domain_lists_id_resource_type_*`

- `GET /api/v1/domain_lists/{id}/{resource_type}/available` —
  `domain_lists_id_resource_type_available_get`: what can this list be attached to.
- `GET /api/v1/domain_lists/{id}/{resource_type}` — `domain_lists_id_resource_type_get`:
  what it is currently attached to.
- `POST /api/v1/domain_lists/{id}/{resource_type}` — `domain_lists_id_resource_type_post`:
  attach.
- `POST /api/v1/domain_lists/{id}/{resource_type}/bulk_delete` —
  `domain_lists_id_resource_type_bulk_delete_post`: detach in bulk.
- `POST /api/v1/domain_lists/{id}/{resource_type}/bulk_update` —
  `domain_lists_id_resource_type_bulk_update_same_attributes_post`: apply the same targeting
  attributes across many resources.

A list with no attachments blocks nothing. Creating the list and loading the domains is only
two thirds of the job — verify the attachment before reporting the change as done.

## 5. Housekeeping

- `POST /api/v1/domain_lists/{id}/duplicate` — `domain_lists_id_duplicate_post` (and
  `..._duplicate_get` to preview) to fork a list before an experimental change.
- `GET /api/v1/domain_lists/{id}/permissions` — `domain_lists_id_permissions_get` to check
  whether this user may write before attempting it; a `403` returns
  `{"error": "You are not authorized to access this page."}` with no remediation hint.
- `DELETE /api/v1/domain_lists/{id}` — `domain_lists_id_delete`. Detach first; deleting an
  attached list silently changes what every attached resource targets.

## Safety

These lists gate live delivery. Emptying a block list by an accidental `bulk_replace` with
an empty body opens inventory to whatever it was blocking, immediately and without warning.
Snapshot with `domain_lists_id_domains_get` before every replace, and never send a replace
whose payload you have not counted.
