---
name: Trace who changed what in a SpringServe account
description: Use the Changelogs audit surface to attribute a delivery change to a specific edit, a specific object and a specific user — the only forensic trail this API exposes.
api: openapi/rubicon-project-springserve-v1-openapi.yml
operations: [changelogs_get, changelogs_id_get, changelogs_versioned_type_versioned_type_get, changelogs_versioned_type_versioned_type_versioned_id_get, changelogs_versioned_types_for_current_account_get, changelogs_user_emails_get]
generated: '2026-08-12'
method: generated
source: openapi/rubicon-project-springserve-v1-openapi.yml + https://springserve.atlassian.net/wiki/spaces/SSD/pages/1620377834/Changelog
---

# Trace who changed what in a SpringServe account

SpringServe's "Changelog" is **not** an API release changelog — it is a per-account audit
log of who edited which object and when. It is the only forensic surface this API publishes,
and it is the first place to look when delivery changes and nobody admits to touching it.

Authenticate and set the account context first: the log is scoped to the active account.

## 1. What kinds of object are logged here — `changelogs_versioned_types_for_current_account_get`

`GET /api/v1/changelogs/versioned_types_for_current_account`. Start here rather than
guessing type names; the set differs by account depending on which products are enabled.

## 2. Sweep the log — `changelogs_get`

`GET /api/v1/changelogs`. Standard list contract: `page`, `per` (default 50, max 1000),
`sort`, `search`, `includes`, and `<attribute>::<operator>=<value>` filters. The useful cut
is a time window:

```
GET /api/v1/changelogs?created_at::gte=2026-08-01&per=1000&sort=-created_at
```

## 3. Narrow to one object type — `changelogs_versioned_type_versioned_type_get`

`GET /api/v1/changelogs/versioned_type/{versioned_type}` for every change to one class of
object (all demand tags, all supply tags).

## 4. Narrow to one object — `changelogs_versioned_type_versioned_type_versioned_id_get`

`GET /api/v1/changelogs/versioned_type/{versioned_type}/{versioned_id}` — the full edit
history of a single supply tag, demand tag or list. This is the call that answers "why did
this tag's rate change on Tuesday".

## 5. Attribute it — `changelogs_user_emails_get`

`GET /api/v1/changelogs/user_emails` returns the users who appear in the log, so you can
resolve an entry to a person.

## 6. One entry — `changelogs_id_get`

`GET /api/v1/changelogs/{id}`.

## Why this matters for API integrations specifically

Authentication is a console user's own email and password, so **an integration's writes are
recorded against a human being**, not against a service identity. If your automation shares
credentials with a person, this log cannot distinguish them and the audit trail is
worthless for both.

Create a dedicated API user (`name+api@example.com` via Settings → Users) before wiring any
write path. It costs nothing and it is the difference between an audit trail and a guess.

## Correlating with delivery

Pair a change window from this log with a report over the same window
(`skills/rubicon-project-run-revenue-report.md`, `interval: "Hour"`) to show the delivery
effect of a specific edit. There is no request-id or correlation-id header on this API, so
timestamp plus object id is the only join key available.
