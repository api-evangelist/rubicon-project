---
name: Schedule a recurring SpringServe report
description: Create and maintain a scheduled report, and save its shape as a reusable report template, so recurring reporting does not spend the 10-per-minute ad-hoc reporting budget.
api: openapi/rubicon-project-springserve-v1-openapi.yml
operations: [scheduled_reports_post, scheduled_reports_get, scheduled_reports_id_get, scheduled_reports_id_patch, scheduled_reports_id_delete, scheduled_reports_bulk_update_same_attributes_put, reports_templates_post, reports_templates_get, reports_templates_id_patch, reports_templates_id_set_default_post]
generated: '2026-08-12'
method: generated
source: openapi/rubicon-project-springserve-v1-openapi.yml (components.examples.scheduled_report_create_simple)
---

# Schedule a recurring SpringServe report

The reporting cap (10 requests/minute, 3/minute by domain or app bundle) makes polling for
the same numbers every morning a bad pattern. Schedule it server-side instead.
Authenticate and set the account context first.

## 1. Create the schedule — `scheduled_reports_post`

`POST /api/v1/scheduled_reports`. The provider's published minimal example:

```json
{
  "name": "Scheduled Report Name",
  "interval": "daily",
  "start_date": "2026-10-01",
  "hour": 8,
  "report_params": {
    "date_range": "Yesterday",
    "dimensions": ["supply_tag_id"],
    "metrics": ["impressions", "revenue"]
  }
}
```

`report_params` takes the same `reportParameters` shape as an ad-hoc `reports_post` — the
same `dimensions`, `metrics`, `filters`, `having`, `interval`, `currency` and `timezone`
fields. Build and validate the report body ad-hoc once (see
`skills/rubicon-project-run-revenue-report.md`), then lift it into `report_params` verbatim.

## 2. Read and maintain

- `GET /api/v1/scheduled_reports` — `scheduled_reports_get`. Supports the standard list
  contract: `page`, `per` (default 50, max 1000), `search`, `sort`, `includes`, and
  `<attribute>::<operator>=<value>` filters.
- `GET /api/v1/scheduled_reports/{id}` — `scheduled_reports_id_get`.
- `PATCH /api/v1/scheduled_reports/{id}` — `scheduled_reports_id_patch`.
- `DELETE /api/v1/scheduled_reports/{id}` — `scheduled_reports_id_delete`.
- `PUT /api/v1/scheduled_reports/bulk_update` —
  `scheduled_reports_bulk_update_same_attributes_put`, to apply the same attributes across
  many schedules at once (e.g. move every daily report an hour later).

## 3. Save the shape as a template — `reports_templates_*`

A template is the reusable dimension/metric selection, separate from the schedule:

- `POST /api/v1/reports/templates` — `reports_templates_post`
- `GET /api/v1/reports/templates` — `reports_templates_get`
- `PATCH /api/v1/reports/templates/{id}` — `reports_templates_id_patch`
- `POST /api/v1/reports/templates/{id}/set_default` — `reports_templates_id_set_default_post`
- `DELETE /api/v1/reports/templates/{id}` — `reports_templates_id_delete`

## Write discipline

There is **no idempotency key** on this API. `scheduled_reports_post` is not safe to blind-
retry: a timed-out create may still have landed, and running it again gives you two
schedules delivering the same file. Before any retry, `scheduled_reports_get` with a
`name=<the name>` filter and confirm whether the first attempt succeeded.

Every write here is attributed to the authenticating console user and shows up in the
Changelogs audit trail — see `skills/rubicon-project-trace-account-changes.md`.
