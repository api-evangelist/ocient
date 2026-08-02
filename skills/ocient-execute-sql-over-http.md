---
name: Execute SQL over the Ocient HTTP Query API
description: Authenticate against an Ocient System and run SQL statements over the HTTP Query API, handling tokens, result formats, and errors.
api: openapi/ocient-http-query-api-openapi-original.json
generated: '2026-08-02'
method: generated
source: openapi/ocient-http-query-api-openapi-original.json
operations:
  - postOcientHttpQueryApiLogin
  - postOcientHttpQueryApiExecute
  - getOcientHttpQueryApiExecute
  - postOcientHttpQueryApiTokenRefresh
  - postOcientHttpQueryApiLogout
  - getOcientHttpQueryApiInfo
---

# Execute SQL over the Ocient HTTP Query API

Use this skill to run SQL against an Ocient System over HTTPS instead of JDBC or pyocient.

## Before you start

- The HTTP Query API runs on the **SQL Nodes** of an Ocient deployment, not on a public
  Ocient-hosted endpoint. The base URL is `https://{sql_node}` — substitute the SQL Node
  address for your system. The default port is 443, configurable in the connectivity pool.
- The API must be enabled on the connectivity pool (`ALTER CONNECTIVITY_POOL` with the
  OpenAPI port). If requests are refused, confirm that first.
- Every SQL Node serves its own definition at `/openapi.yaml` and `/openapi.json`.
- The HTTP Query API **does not support transactions**. If you need `COMMIT`/`ROLLBACK`,
  use the JDBC driver or pyocient instead.

## 1. Check connectivity and version

Call `getOcientHttpQueryApiInfo` (`GET /v1/info`). It needs no authentication and returns
the default database, the OpenAPI version, and a status object. Use it to verify the node is
reachable and that your client is version-compatible before authenticating.

## 2. Get a token

Call `postOcientHttpQueryApiLogin` (`POST /v1/login`) with a JSON body containing
`username`, `password`, and `database`. The username is a **fully qualified user name (FQUN)**
in the form `<user_name>@<database>` — for example `alice@example_database`.

The response carries a bearer token, and the call also sets a session cookie.

Alternatively, pass HTTP Basic credentials directly on the request.

For OpenID Connect single sign-on instead of a password, use the
`ocient-sso-device-grant` skill.

## 3. Execute a statement

Call `postOcientHttpQueryApiExecute` (`POST /v1/execute/{database}`).

- Path parameter: `database` (required).
- Required headers: `authorization` (`Bearer <token>`) and `content-type`.
- Optional query parameters: `format`, `schema`.
- Optional headers: `accept`, `accept-encoding`, `preferred-encoding`,
  `preferred-compression-level`.
- JSON body properties: `statement`, `database`, `format`, `params`, `fetch_size`,
  `max_rows`.

Set `format` to control the result shape (for example `collection`). Use `fetch_size` to
control batching and `max_rows` to cap the result set — do this rather than pulling an
unbounded result from a petabyte-scale table.

Use `params` for parameterized statements. `getOcientHttpQueryApiExecute`
(`GET /v1/execute/{database}`) is the alternative read-style form that passes `statement`,
`database`, `format`, and `fetch_size` as query parameters — it **does not support the
`params` body parameter**, so prefer the POST form whenever you are binding values.

Both execute operations support streaming responses as well as regular ones.

## 4. Keep the session alive

Call `postOcientHttpQueryApiTokenRefresh` (`POST /v1/token_refresh`) with the `Authorization`
and `Content-Type` headers and a JSON body containing `database`. Call it *before* the current
token expires to keep access uninterrupted.

## 5. Finish

Call `postOcientHttpQueryApiLogout` (`POST /v1/logout`). This clears associated cookies but
**does not invalidate access tokens** — do not treat logout as a token revocation.

## Error handling

Responses carry a `status` object with `reason` and `sql_state`. Ocient error codes are
negative integers grouped in ranges; see `errors/ocient-error-codes.yml` in this repo, or
query `SELECT * FROM sys.sql_messages` in the database. Common ones:

- `-304` — user and password combination incorrect. Re-check the FQUN form.
- `-205` — no database with that name exists.
- `-201` / `-203` / `-206` — connection failed, closed unexpectedly, or target node offline.
- `-500` — generic syntax error in the statement.
- `-336` — a prior statement is still running on the same connection.

## Safety notes

- SQL execution is a **write-capable** surface: `postOcientHttpQueryApiExecute` will run DDL
  and DML, not just `SELECT`. Do not send generated SQL without review.
- Because there are no transactions on this API, a failed multi-statement sequence cannot be
  rolled back here. Use JDBC or pyocient for all-or-nothing work.
