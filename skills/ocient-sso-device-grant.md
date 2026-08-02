---
name: Authenticate to Ocient with OpenID Connect SSO
description: Obtain an Ocient access token through the OpenID Connect single sign-on flow, including the browser redirect flow and the device grant flow for headless agents.
api: openapi/ocient-http-query-api-openapi-original.json
generated: '2026-08-02'
method: generated
source: openapi/ocient-http-query-api-openapi-original.json
operations:
  - postOcientHttpQueryApiSsoAuthentication
  - getOcientHttpQueryApiCallback
  - postOcientHttpQueryApiSsoToken
  - postOcientHttpQueryApiSsoDeviceGrant
  - postOcientHttpQueryApiSsoDeviceGrantVerify
  - postOcientHttpQueryApiTokenRefresh
---

# Authenticate to Ocient with OpenID Connect SSO

Ocient supports two authentication methods: password authentication and OpenID Connect (OIDC)
single sign-on against an external identity provider. This skill covers the SSO path over the
HTTP Query API.

## Before you start

- A database — including the `system` database — can have **0 or 1** SSO integrations,
  configured by an administrator. If none is configured, use password authentication instead
  (see `ocient-execute-sql-over-http`).
- The existence of an SSO integration has no effect on users who authenticate with a password.
- Ocient recommends MFA on all accounts and treating local accounts as emergency access only.

## Choose a flow

Pick the **device grant** flow when there is no browser — headless agents, CLI tools, or
servers. Pick the **redirect** flow when a user can complete an interactive login.

## Flow A — device grant (headless)

1. Call `postOcientHttpQueryApiSsoDeviceGrant` (`POST /v1/sso_device_grant`) to retrieve an
   OpenID device grant code that the Ocient System can verify.
2. Present the returned user code / verification URI to the human so they can approve it at
   the identity provider.
3. Poll `postOcientHttpQueryApiSsoDeviceGrantVerify` (`POST /v1/sso_device_grant_verify`) to
   verify the earlier device grant request and return an authorization token.

## Flow B — browser redirect

1. Call `postOcientHttpQueryApiSsoAuthentication` (`POST /v1/sso_authentication`). This
   initiates the OIDC authentication process by redirecting to the authorization server.
2. After a successful login the authorization server redirects to the callback path.
   `getOcientHttpQueryApiCallback` (`GET /v1/callback`) provides the authentication token for
   that process.

## Flow C — exchange an existing IdP token

If you already hold an OIDC identifier token or access token from the identity provider, call
`postOcientHttpQueryApiSsoToken` (`POST /v1/sso_token`) to exchange it for an **Ocient** access
token. Use this when the agent is already federated with the same IdP.

## Use and maintain the token

Send the resulting token as `Authorization: Bearer <token>` on
`postOcientHttpQueryApiExecute` and the System Information endpoints.

Call `postOcientHttpQueryApiTokenRefresh` (`POST /v1/token_refresh`) — headers `Authorization`
and `Content-Type`, JSON body with `database` — before the token expires. Refresh proactively;
there is no re-auth grace period once a token lapses mid-query.

## Notes and cautions

- `postOcientHttpQueryApiLogout` clears session cookies but **does not invalidate access
  tokens**. To revoke access you must act at the identity provider or through Ocient user
  administration, not by calling logout.
- The published OpenAPI declares no `securitySchemes` even though the API requires
  authentication; the `authorization` header is modelled as an ordinary required header
  parameter. Do not read the empty `securitySchemes` as "no auth required".
- Ocient supports any OIDC identity provider (Okta is the documented example).
