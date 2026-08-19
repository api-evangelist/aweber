---
name: Manage an AWeber subscriber
description: Find a subscriber on an AWeber list, add them if they are missing, update their tags and custom fields, read their activity, and remove them — using the AWeber REST API 1.0.
api: openapi/aweber-subscribers-api-openapi.yml
operations:
  - getAccounts
  - getLists
  - findSubscribersForList
  - addSubscriber
  - getSubscriber
  - updateSubscriberByID
  - getSubscriberActivity
  - deleteSubscriberByID
generated: '2026-08-13'
method: generated
source: openapi/_original/aweber-api-openapi.yml + https://api.aweber.com/#tag/Subscribers
---

# Manage an AWeber subscriber

Everything in AWeber hangs off an account and a list. You cannot address a subscriber without
both ids, so resolve them first — every call below is scoped to
`/accounts/{accountId}/lists/{listId}`.

## Before you start

- **Auth**: OAuth 2.0 authorization code against `https://auth.aweber.com/oauth2/authorize`,
  token at `https://auth.aweber.com/oauth2/token`. Send `Authorization: Bearer <token>`.
  Public clients (anything that cannot hold a secret) **must** use PKCE; confidential clients
  **must not** send `code_verifier`/`code_challenge`. Getting that backwards is a named error.
- **Scopes**: `account.read`, `list.read`, `subscriber.read` for the reads;
  `subscriber.write` for add/update/delete/move.
- **Base URL**: `https://api.aweber.com/1.0`.
- **Write bodies are form-encoded** (`application/x-www-form-urlencoded`), not JSON. Sending
  JSON where the endpoint does not accept it returns `415 InvalidContentType`.
- **There is no idempotency key.** A retried `addSubscriber` after a timeout can create a
  duplicate. Set `update_existing=true` on the add if a repeat should update instead of fail.
- **Budget**: 120 requests per minute per AWeber customer account, with no rate-limit headers.
  Pace yourself; you get no warning before the 403.

## Steps

1. **Resolve the account** — `getAccounts` (`GET /accounts`). Take `entries[0].id` when the
   token has a single account; otherwise pick by `id`/`uuid`.
2. **Resolve the list** — `getLists` (`GET /accounts/{accountId}/lists`), or `findLists`
   (`GET /accounts/{accountId}/lists?ws.op=find`) to match by name. Collections are paginated
   with `ws.size` (1-100, default 100) and `ws.start`; follow `next_collection_link` rather
   than incrementing offsets.
3. **Look for the subscriber first** — `findSubscribersForList`
   (`GET /accounts/{accountId}/lists/{listId}/subscribers?ws.op=find`) with `email=`. To search
   every list on the account at once, use `findSubscribersForAccount`
   (`GET /accounts/{accountId}?ws.op=findSubscribers`). Note the `?ws.op=` operation form: it
   is part of the published path, not an optional query parameter.
4. **Add if absent** — `addSubscriber`
   (`POST /accounts/{accountId}/lists/{listId}/subscribers`) with `email` plus optional `name`,
   `tags`, `custom_fields`, `ad_tracking`, `misc_notes`, `ip_address`, `update_existing`.
   Do not pass a private or reserved IP in `ip_address` — that is a documented `400`.
5. **Update** — `updateSubscriberByID`
   (`PATCH /accounts/{accountId}/lists/{listId}/subscribers/{subscriberId}`), or
   `updateSubscriberByEmail` (`PATCH .../subscribers`) when you only hold the address. You can
   change `status`, `tags`, `custom_fields`, `name`, `ad_tracking`, `misc_notes`.
   Unknown custom fields are rejected when `strict_custom_fields` is set.
6. **Read the history** — `getSubscriberActivity`
   (`GET .../subscribers/{subscriberId}?ws.op=getActivity`) returns a polymorphic activity
   stream (sent message, open, click, link, subscribed, tracked event); branch on `type`.
7. **Remove** — `deleteSubscriberByID` (`DELETE .../subscribers/{subscriberId}`) or
   `deleteSubscriberByEmail` (`DELETE .../subscribers`). Success is `204` with no body.

## Error handling

Errors come back as `{"error": {"status", "message", "type", "documentation_url"}}` — this is
**not** RFC 9457 problem+json.

- `401 UnauthorizedError` — read `message`: *Invalid Token* means refresh or re-authorize;
  *Invalid Account* means the AWeber customer's account is inactive and no retry will help.
- `403 ForbiddenError` — **overloaded**. `message: "Rate Limit Error"` is retryable after a
  pause; *Account Temporarily Suspended*, *account on hold* and missing-permission variants are
  not. Always branch on the message, never on the status alone.
- `400 BadRequestError` — missing required parameter, read-only field, non-UTF-8 encoding, or
  `ws.size` outside 1-100.
- `405` — you tried `move` on an `unconfirmed` subscriber, or wrote to a read-only resource.

## Related

- Move a subscriber between lists: `moveSubscriber` (`POST .../subscribers/{subscriberId}`).
  The subscriber must not be `unconfirmed`.
- React to subscriber changes instead of polling: `asyncapi/aweber-webhooks.yml`
  (`subscriber.added`, `subscriber.subscribed`, `subscriber.unsubscribed`).
- Conventions, pagination and rate limits: `conventions/aweber-conventions.yml`.
