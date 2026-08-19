---
name: Track a purchase in AWeber
description: Record a purchase against an AWeber list so the buyer is created or updated as a subscriber, tagged, and available to behavioural automations and activity reporting.
api: openapi/aweber-subscribers-api-openapi.yml
operations:
  - getAccounts
  - getLists
  - createAPurchase
  - findSubscribersForList
  - getSubscriberActivity
  - getTagsForList
generated: '2026-08-13'
method: generated
source: openapi/_original/aweber-api-openapi.yml + https://api.aweber.com/#tag/Subscribers
---

# Track a purchase in AWeber

`createAPurchase` is AWeber's ecommerce hook: one call records a tracked purchase event **and**
creates or updates the buyer as a subscriber on the list. Added in API 1.2.0 (2021-12-15).

## Before you start

- **Scopes**: `account.read`, `list.read`, `subscriber.write`.
- **Base URL**: `https://api.aweber.com/1.0`; bearer token; form-encoded body.
- **No idempotency key exists.** A retry after a network timeout will record a *second*
  purchase. Key your own dedupe on the order id before calling, and prefer letting a failed call
  fail over blind retries.

## Steps

1. **Resolve account and list** — `getAccounts`, then `getLists` (or `findLists` by name).
2. **Record the purchase** — `createAPurchase`
   (`POST /accounts/{accountId}/lists/{listId}/purchases`). The payload carries both the
   transaction and the person:

   | Purchase | Subscriber |
   |---|---|
   | `event_time` | `email` |
   | `product_name` | `name` |
   | `value` | `tags` |
   | `currency` | `custom_fields` |
   | `vendor` | `ip_address` |
   | `url` | `ad_tracking` |
   | `event_note` | `misc_notes` |

   If the email is not on the list, the subscriber is created. If it is, the existing subscriber
   is used and the purchase is attached to them.

3. **Confirm** — `findSubscribersForList`
   (`GET /accounts/{accountId}/lists/{listId}/subscribers?ws.op=find`) with `email=`, then
   `getSubscriberActivity` (`GET .../subscribers/{subscriberId}?ws.op=getActivity`). The
   purchase appears in the polymorphic activity stream; branch on `type`.
4. **Segment on it** — tags you sent are visible via `getTagsForList`
   (`GET /accounts/{accountId}/lists/{listId}/tags`), and are what behavioural automations key
   on. Tag deliberately at purchase time; there is no bulk retag endpoint.

## Error handling

- `400 BadRequestError` — missing required field, non-UTF-8 encoding, or a reserved/private
  `ip_address`.
- `403 ForbiddenError` — read `message`: *Rate Limit Error* (retry after a pause),
  *account on hold* (*"Adding subscribers is not available while the account is on hold."*),
  or a suspended account. Only the first is retryable.
- `415 InvalidContentType` — send `application/x-www-form-urlencoded`.

## Related

- Subscriber lifecycle: `skills/aweber-manage-subscriber.md`
- Event-driven alternative to polling: `asyncapi/aweber-webhooks.yml`
- Conventions and limits: `conventions/aweber-conventions.yml`
