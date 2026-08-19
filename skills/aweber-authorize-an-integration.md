---
name: Authorize an AWeber integration
description: Take an AWeber application from a labs.aweber.com registration to a working access token — OAuth 2.0 authorization code with PKCE for public clients, refresh, and revoke.
api: openapi/aweber-authentication-api-openapi.yml
operations:
  - getAToken
  - revokeAToken
  - getAccounts
generated: '2026-08-13'
method: generated
source: >-
  openapi/_original/aweber-api-openapi.yml + https://api.aweber.com/#tag/OAuth-2.0-Overview
  + https://api.aweber.com/#tag/Troubleshooting
---

# Authorize an AWeber integration

AWeber has two account worlds and confusing them is the most common failure. A **developer
account** at `https://labs.aweber.com` is free, unrelated to billing, and is where you register
the app and get the client id. A **customer account** at `aweber.com` is what holds the lists
and subscribers, and is what grants your app access. You need both.

## Before you start

Register the app at `https://labs.aweber.com/apps` and record:

- `client_id`
- `client_secret` — **confidential clients only**
- the exact **OAuth Redirect URL**

Decide which client type you are:

| Type | Holds a secret? | PKCE |
|---|---|---|
| Confidential (server-side) | yes | **must NOT** send `code_verifier`/`code_challenge` |
| Public (mobile app, WordPress plugin, SPA) | no | **must** send `code_challenge` |

Sending PKCE parameters as a confidential client, or omitting them as a public client, are both
named, documented errors — not silent fallbacks.

## Steps

1. **Build the authorize URL** and open it in the customer's browser:

   ```
   https://auth.aweber.com/oauth2/authorize
     ?response_type=code
     &client_id=YOUR_CLIENT_ID
     &redirect_uri=YOUR_CALLBACK
     &scope=YOUR_REQUESTED_SCOPES
     &state=STATE_TOKEN
   ```

   Public clients add `code_challenge` and `code_challenge_method=S256`.
   `redirect_uri` must match the registered OAuth Redirect URL **exactly** — a trailing-slash
   difference is the documented cause of "Invalid redirect URI".

2. **Ask for the scopes you actually use** (space separated, URL-encoded):

   | Scope | Grants |
   |---|---|
   | `account.read` | accounts and integrations |
   | `list.read` | lists, custom fields, tags, sign-up forms |
   | `list.write` | create/edit/delete custom fields |
   | `subscriber.read` | subscribers and their activity |
   | `subscriber.write` | add/edit/delete/move subscribers |
   | `email.read` | broadcasts, follow-ups, opens, clicks, campaign stats |
   | `email.write` | create, update, schedule and cancel broadcasts |
   | `landing-page.read` | landing pages |

   `subscriber.read-extended` still exists but has been equivalent to `subscriber.read` since
   2021-06-16 — do not request it in new code.

3. **Exchange the code** — `getAToken` (`POST /oauth2/token`) with `grant_type=authorization_code`,
   the `code`, your `redirect_uri`, `client_id`, and either `client_secret` (confidential) or
   `code_verifier` (public). The response carries `access_token`, `refresh_token` and
   `expires_in`.

4. **Verify the grant** — `getAccounts` (`GET /accounts`) with
   `Authorization: Bearer <access_token>`. A `200` with entries means the customer's account is
   reachable; cache the account id, you need it on every other call.

5. **Refresh before expiry** — `getAToken` again with `grant_type=refresh_token`. Persist the
   new refresh token if one is returned.

6. **Revoke on disconnect** — `revokeAToken` (`POST /oauth2/revoke`). Call this when a customer
   disconnects rather than dropping the token on the floor.

## Webhooks need this too

To receive `subscriber.added` / `subscriber.subscribed` / `subscriber.unsubscribed`, the
integration must be authorized with **`subscriber.read` and `account.read`**, and the customer
must set a Webhook URL at `https://www.aweber.com/users/apps` matching a prefix in your
Callback URL Allow-list. See `asyncapi/aweber-webhooks.yml`.

## Error handling

Token-endpoint failures use the RFC 6749 shape (`invalid_client`, `invalid_request`,
`server_error`), not the AWeber error envelope. API-call failures use the AWeber envelope:

- `401` *Invalid Token* — expired, revoked by the customer, or mistyped. Refresh or re-authorize.
- `401` *Invalid Request* — the call did not use OAuth 2.0 at all.
- `403` — read `message` to tell a rate limit from a suspended account from a missing permission.

## Note on OAuth 1.0a

`/oauth/request_token` and `/oauth/access_token` are still published and still answer — an
unauthenticated request to any 1.0 endpoint replies *"Missing oauth parameters:
oauth_consumer_key"*. They are legacy. AWeber requires OAuth 2.0 for all new applications; do
not build against 1.0a.
