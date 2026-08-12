---
name: Authenticate to SpringServe and set the account context
description: Exchange console credentials for a SpringServe API token, confirm which account the session is operating in, and switch that context before doing any other work.
api: openapi/rubicon-project-springserve-v1-openapi.yml
operations: [auth_post, accounts_current_get, accounts_displayable_get, accounts_id_set_current_post, session_get]
generated: '2026-08-12'
method: generated
source: openapi/rubicon-project-springserve-v1-openapi.yml + https://springserve.atlassian.net/wiki/spaces/SSD/pages/1573617663/API+-+Getting+Started
---

# Authenticate to SpringServe and set the account context

Every other SpringServe / ClearLine skill starts here. Two things are true of this API that
are not true of most: the credential is a **human console user's email and password**, and
every request silently runs against whatever account that user last made active. Get the
second one wrong and you will read or write the wrong publisher's inventory.

Base URL: `https://console.springserve.com/api/v1` (ClearLine: `https://console.clearline.magnite.com/api/v1`).

## 1. Get a token — `auth_post`

`POST /api/v1/auth`, body **`application/x-www-form-urlencoded`** (not JSON), fields `email`
and `password`. The response carries both `token` and `bearer_token`.

Send it on every subsequent call as either:

- `Authorization: <token>` — bare, **no scheme prefix**, or
- `Authorization: Bearer <bearer_token>`

The token expires after **two hours**. There is no refresh endpoint — re-POST `/auth`.
There is no OAuth, no client id/secret, and no scope: the token carries the full rights of
that console user. Create a dedicated API user (`name+api@example.com` via Settings → Users)
so API-originated writes are attributable in the Changelogs audit trail.

Never log the token or the password. See `authentication/rubicon-project-authentication.yml`.

## 2. Confirm the account context — `accounts_current_get`

`GET /api/v1/accounts/current` returns the account this session will act on. Do this before
any read that will be reported on and before **every** write. Do not assume it persisted
from a previous run — it is server-side session state, not something you set per request by
default.

## 3. List the accounts you may switch to — `accounts_displayable_get`

`GET /api/v1/accounts/displayable` returns the accounts this user can select. If the target
account is not in this list, stop: the user lacks the grant, and no amount of retrying will
change that.

## 4. Switch — `accounts_id_set_current_post`

`POST /api/v1/accounts/{id}/set_current`. This mutates session state for every later call.

**Preferred alternative for multi-account work:** send the `x-auth-context` header on each
request instead of switching global context. It is per-request, so concurrent work against
several accounts cannot interleave and corrupt each other.

## 5. Inspect the session — `session_get`

`GET /api/v1/session` when you need to see what the server thinks the session is.

## Rules that apply to everything downstream

- **No idempotency.** Zero operations in either spec accept an idempotency key. A POST that
  times out may or may not have been applied — re-read before you retry a write.
- **Rate limits:** 240 requests/minute per account overall; 10/min to `/report`; 3/min per
  domain or app bundle. No `RateLimit-*` or `Retry-After` header is returned and no `429` is
  declared, so self-throttle against these numbers. See `rate-limits/`.
- **Errors** are `{"error": "<message>"}` with an optional `{"errors": {field: [msg]}}` on
  422. No RFC 9457, no stable error codes — branch on HTTP status, never on message text.
  See `errors/rubicon-project-problem-types.yml`.
- **v0 and v1 are both live.** v1 does not yet cover everything v0 does; supply-tag and
  demand-tag *creation/replacement* still live on `/api/v0`. Check both specs per endpoint.
