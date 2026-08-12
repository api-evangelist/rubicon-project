---
name: Run a SpringServe revenue or delivery report
description: Submit an ad-hoc report against the SpringServe reporting API, poll it to completion, and download the result — respecting the 10-requests-per-minute reporting cap.
api: openapi/rubicon-project-springserve-v1-openapi.yml
operations: [reports_post, report_post, reports_id_get, reports_download_id_get]
generated: '2026-08-12'
method: generated
source: openapi/rubicon-project-springserve-v1-openapi.yml (components.examples.report_create_simple / report_create_full, ref/reports.yaml#reportParameters)
---

# Run a SpringServe revenue or delivery report

Reporting is the highest-value read surface on this API and the one with the tightest cap.
Authenticate and set the account context first (see
`skills/rubicon-project-authenticate-and-set-account.md`).

## 1. Submit the report — `reports_post`

`POST /api/v1/reports` (`report_post` on `/api/v1/report` is a documented alias — pick one
and stay on it). Body is JSON matching `reportParameters`:

| Field | Notes |
|---|---|
| `date_range` | e.g. `Yesterday`; use `Custom` plus `custom_date_range` for explicit dates |
| `dimensions` | array, e.g. `["supply_tag_id","demand_tag_id"]` |
| `metrics` | array, e.g. `["impressions","revenue","fill_rate","cpm"]` |
| `interval` | `Hour`, `Day`, `Week`, `Month`, `Quarter` or null |
| `filters` | `[{dimension, type: values\|lists, values[], is_include, is_exact_match}]` |
| `having` | `[{metric, max, value}]` — post-aggregation metric thresholds |
| `currency`, `timezone`, `sort`, `async`, `csv` | optional |

The provider's own published minimal example:

```json
{"date_range":"Yesterday","dimensions":["supply_tag_id"],"metrics":["impressions","revenue"]}
```

Set `async: true` for anything wide or long-range; a synchronous call on a large account
will hold the connection and burn your 10/minute budget while it does.

## 2. Poll — `reports_id_get`

`GET /api/v1/reports/{id}` with the id returned by the submit. It accepts `page`, `limit`,
`sort` and `source`. Poll on a fixed interval that keeps total reporting calls under
**10 per minute** — the submit and every poll count against the same cap, and there is no
`Retry-After` header to tell you when you have crossed it.

## 3. Download — `reports_download_id_get`

`GET /api/v1/reports/download/{id}`, optional `zipped`. Use this for CSV output rather than
paging a large result set through `reports_id_get`.

## Caps and pitfalls

- **10 requests/minute** to the reporting endpoints, per account. **3 requests/minute** when
  the report is broken out by domain or app bundle — the most commonly requested cut is also
  the most restricted.
- The account-wide cap of **240 requests/minute** still applies on top.
- No rate-limit response headers and no declared `429`. Track your own call budget.
- Only a `200` response is declared on all three operations; failures still arrive as the
  standard `{"error": "..."}` envelope. Do not assume a `200` shape without checking.
- The 2-hour token TTL will expire mid-poll on a long async report. Re-authenticate and
  resume with the same report `id` — the report is server-side, not session-bound.

For a report you need on a cadence, use `skills/rubicon-project-schedule-recurring-report.md`
instead of polling on a timer.
