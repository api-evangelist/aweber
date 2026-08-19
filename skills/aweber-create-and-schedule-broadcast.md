---
name: Create and schedule an AWeber broadcast
description: Draft a broadcast email on an AWeber list, target it with segments and include/exclude lists, schedule it to send, and cancel it if plans change.
api: openapi/aweber-broadcasts-api-openapi.yml
operations:
  - getAccounts
  - getLists
  - getSegments
  - createBroadcast
  - getBroadcast
  - updateBroadcast
  - scheduleBroadcast
  - cancelScheduledBroadcast
  - getTotalBroadcasts
  - deleteBroadcast
generated: '2026-08-13'
method: generated
source: openapi/_original/aweber-api-openapi.yml + https://api.aweber.com/#tag/Broadcasts
---

# Create and schedule an AWeber broadcast

A broadcast is a one-off email to a list. Creating and scheduling are **two separate calls** —
`createBroadcast` only produces a draft, and nothing sends until `scheduleBroadcast` succeeds.

## Before you start

- **Scopes**: `account.read` and `list.read` to resolve ids, `email.read` to read broadcasts,
  **`email.write` to create, update, schedule, cancel or delete them**. Without `email.write`
  the API answers `403 ForbiddenError` with *"Application not authorized to manage email."*, and
  the AWeber customer must grant the Manage All Email Communications permission — you cannot fix
  that from the client side.
- **Base URL**: `https://api.aweber.com/1.0`; bearer token; form-encoded bodies.
- **Sending is irreversible.** Treat `scheduleBroadcast` as the point of no return and confirm
  with a human before calling it in an agentic flow.

## Steps

1. **Resolve account and list** — `getAccounts`, then `getLists`.
2. *(optional)* **Pick a segment** — `getSegments`
   (`GET /accounts/{accountId}/lists/{listId}/segments`) and keep the segment's `self_link` for
   the `segment_link` field.
3. **Create the draft** — `createBroadcast`
   (`POST /accounts/{accountId}/lists/{listId}/broadcasts`).
   - `subject` is required.
   - **Either `body_html` or `body_text` must be present** — omitting both is a documented
     `400 BadRequestError`. `body_amp` is optional.
   - Targeting: `segment_link`, `include_lists`, `exclude_lists`.
   - Behaviour: `click_tracking_enabled`, `is_archived`, `notify_on_send`,
     `facebook_integration`, `twitter_integration`.
4. **Review before sending** — `getBroadcast`
   (`GET .../broadcasts/{broadcastId}`) and check `subject`, `segment_name` and the body you
   just wrote. Use `updateBroadcast` (`PUT .../broadcasts/{broadcastId}`) to correct a draft.
   `PUT` replaces the draft — send the full field set, not a partial one.
5. **Schedule** — `scheduleBroadcast` (`POST .../broadcasts/{broadcastId}/schedule`) with
   `scheduled_for`. A `409` here means the broadcast has already been queued or sent, or the
   account is locked — do not retry blindly.
6. **Cancel if needed** — `cancelScheduledBroadcast`
   (`POST .../broadcasts/{broadcastId}/cancel`) while it is still scheduled. Once sent, it
   cannot be recalled. `deleteBroadcast` removes a draft.
7. **Count what exists** — `getTotalBroadcasts` (`GET .../broadcasts/total`) returns counts by
   state (draft / scheduled / sent); `getBroadcasts` lists them, filtered by state.

## After it sends

- `getBroadcastOpens` (`GET .../broadcasts/{broadcastId}/opens`) and `getBroadcastClicks`
  (`GET .../broadcasts/{broadcastId}/clicks`) return the subscribers who opened and clicked.
  Pass `detailed=true` to the clicks endpoint for per-link detail (added 2025-08-22).
- `getBroadcastStatistics` (`GET .../campaigns/b{campaignId}/stats`) returns aggregate stats.
  Note the `b` prefix: a sent broadcast is addressed as a campaign of type `b`.

## Error handling

- `403` + *"Application not authorized to manage email."* — missing `email.write` / customer
  permission. Not retryable.
- `403` + *"Rate Limit Error"* — you exceeded 120 requests/minute for that AWeber customer.
  There is no `Retry-After` header; back off and retry.
- `409 ConflictError` — scheduling something already queued or sent.
- `400` — missing `subject`, or neither `body_text` nor `body_html`.

## Related

- Conventions, pagination, rate limits: `conventions/aweber-conventions.yml`
- Full error catalog: `errors/aweber-problem-types.yml`
