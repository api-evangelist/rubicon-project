---
name: Audit SpringServe supply and demand tags
description: Enumerate publisher supply tags and buyer demand tags with the list contract (filter, sort, includes, pagination), read the demand waterfall on a supply tag, and make a safe targeted update.
api: openapi/rubicon-project-springserve-v1-openapi.yml
operations: [supply_tags_get, supply_tags_id_get, supply_tags_id_demand_tag_priorities_get, demand_tags_get, demand_tags_id_get, demand_tags_post, demand_tags_id_patch]
generated: '2026-08-12'
method: generated
source: openapi/rubicon-project-springserve-v1-openapi.yml (+ openapi/rubicon-project-springserve-v0-openapi.yml for supply-tag writes)
---

# Audit SpringServe supply and demand tags

Supply tags are the publisher-side inventory that generates ad requests; demand tags are the
buyer-side sources that bid into them. This is the core object graph of the ad server.
Authenticate and set the account context first.

## 1. Enumerate — `supply_tags_get` / `demand_tags_get`

`GET /api/v1/supply_tags` and `GET /api/v1/demand_tags`. Every list endpoint on this API
honours the same contract:

- **Pagination** — `page` (1-indexed) and `per` (default **50**, max **1000**). The response
  carries `count`, `total_count`, `current_page`, `total_pages`, `results`. There is no
  cursor option, so deep paging is offset-bound; page in descending size, not ascending.
- **Filtering** — `<attribute>=<value>` on any first-level attribute, plus operators
  `::gt`, `::gte`, `::lt`, `::lte`, `::in`, `::null`. e.g. `updated_at::gte=2026-08-01`,
  `id::in=1,2,3`, `active=true`.
- **Sorting** — `sort=name,-id` (leading `-` for descending).
- **Search** — free-text `search=`.
- **Includes** — `includes=` takes a comma-delimited list of associations. The response's
  `includable_fields` tells you what is available. **Pass `includes=` empty to fetch no
  associations** — on a large account this is the single biggest speed-up available and the
  documented way to stay inside the 240 req/min account cap.
- **Extra data** — `additional_data=quickstats` pulls computed performance blocks onto list
  rows; `additional_data_fields` on the response lists what is offered.

For an audit sweep: `per=1000&includes=` and filter on `updated_at::gte` to walk only what
changed since the last run.

## 2. Read one — `supply_tags_id_get` / `demand_tags_id_get`

`GET /api/v1/supply_tags/{id}` and `GET /api/v1/demand_tags/{id}`. Ids are bare integers
with no type prefix, so a supply tag id and a demand tag id are indistinguishable — carry
the type alongside the id in your own state or you will address the wrong object.

## 3. Read the waterfall — `supply_tags_id_demand_tag_priorities_get`

`GET /api/v1/supply_tags/{id}/demand_tag_priorities` returns the demand tag priority order
on that supply tag — which buyers get the impression first. This is the relationship the
list endpoints do not give you, and it is what makes a supply/demand audit meaningful rather
than two disconnected inventories.

## 4. Change something — `demand_tags_id_patch`, `demand_tags_post`

- `PATCH /api/v1/demand_tags/{id}` — `demand_tags_id_patch`. Send only the fields you are
  changing.
- `POST /api/v1/demand_tags` — `demand_tags_post`. Returns `201`, or `422` with
  `{"errors": {"<field>": ["<message>"]}}`. The spec publishes create examples for each
  demand class: tag, bidder, line item, house ad, DSP sub line item and marketplace deal.
  Read `components.examples.demand_tag_*` in the spec and start from the one that matches
  your `demand_class` rather than assembling a body field by field.

**Supply-tag writes are v0-only.** v1 exposes supply tags read-only; creation and
replacement live at `POST /api/v0/supply_tags` (`supply_tags_post`) and
`PUT /api/v0/supply_tags/{id}` (`supply_tags_id_put`) in
`openapi/rubicon-project-springserve-v0-openapi.yml`. This is the concrete case of the
provider's own warning that v1 does not yet cover everything v0 does.

## Safety

Changing a demand tag rate or a waterfall priority moves live money. There is no
idempotency key, no dry-run, and no sandbox environment — writes land on production
inventory. Read the object first, patch the narrowest field set that achieves the change,
then re-read to confirm. On a timeout, re-read before retrying; never blind-retry a write.
