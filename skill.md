---
name: mailbeast
description: Use when working with the MailBeast cold-email API (https://api.mailbeast.ai/v1) or its MCP server (https://mcp.mailbeast.ai/mcp) - creating campaigns and sequences, adding leads, connecting sending mailboxes, reading and replying to emails, checking plan usage and campaign analytics.
license: Proprietary
---

# MailBeast API and MCP

MailBeast is a cold-email platform: sending mailboxes with warmup, multi-step
campaigns, a unified inbox for replies, analytics, and Lead Finder for prospect
discovery. Docs: https://developer.mailbeast.ai (full page list: `/llms.txt`).

## Access

- REST API: `https://api.mailbeast.ai/v1/...`, header `Authorization: Bearer mb_live_…`.
  The key decides the workspace - never put an organization id in the path.
- MCP server: `https://mcp.mailbeast.ai/mcp` with the same key as a Bearer header,
  or `https://mcp.mailbeast.ai/mcp/<key>` for clients that only take a URL. It lists
  only the tools the key's scopes allow.
- API access requires the Pro+ plan or an enterprise contract (`403` otherwise).
- Scopes are `resource:action` (`campaigns:read`, `leads:write`, `emails:send`, …)
  plus wildcards (`campaigns:all`, `all:read`, `all:all`). `apikeys:manage` is never
  covered by a wildcard.
- Rate limit: 120 requests per minute per workspace; `429` carries `Retry-After`.

## Rules that matter

- **Check limits before spending.** `GET /v1/usage` (MCP: `get_workspace`). Adding
  leads spends `monthly_imports`; sending spends `monthly_emails`. A limit hit is
  `402` with `code: plan_limit_exceeded` and `details` naming the limit.
- **Campaigns send from all connected mailboxes** of the workspace when created
  through the API; the `accounts` list does not restrict sending.
- **Create, test, then start.** Create the campaign as a draft, read
  `sequence[].variants[].id` from `GET /v1/campaigns/{id}`, send a test with
  `POST /v1/campaigns/{id}/test-email` (12 per day per workspace), then
  `POST /v1/campaigns/{id}/start`. Starting needs at least one lead and a sequence.
- **`PATCH /v1/campaigns/{id}`**: `sequence` and `accounts`, when sent, REPLACE the
  existing ones.
- **Schedule**: one window for all `sendingDays` (`sendingMinutesStart`/`End`, minutes
  from midnight in `timezone`), or per-day windows via `schedule.intervals`
  (`[{ day: 1-7 (Mon-Sun), startMinutes, endMinutes }]`, 30-minute step, no overlaps
  within a day) - that switches the campaign to `mode: "advanced"` and REPLACES all
  windows; `mode: "simple"` switches back. Read the current one from
  `schedule.mode` / `schedule.intervals`.
  `endDate` = last sending day (until its end in the campaign timezone), then the
  campaign completes; `endDate: null` on update removes it.
- **Leads**: 1-100 per `POST /v1/campaigns/{cid}/leads`; deduplicated by email per
  campaign; malformed addresses are counted as `invalid`, not rejected.
- **Merge tags are snake_case**: `{{first_name}}`, `{{last_name}}`, `{{full_name}}`,
  `{{email}}`, `{{company_name}}`, `{{job_title}}`, `{{website}}`, `{{linkedin_profile}}`,
  `{{location}}`, custom fields as `{{custom_<field>}}` (lead `customFields.industry`
  → `{{custom_industry}}`), sender `{{senderFirstName}}` / `{{senderLastName}}` /
  `{{senderFullName}}`. Any other tag - including camelCase `{{firstName}}` or a tag
  with spaces inside the braces (`{{ first_name }}`) - renders as EMPTY text.
- **Lead statuses** you may set: `lead_replied_interested`,
  `lead_replied_meeting_request`, `lead_replied_out_of_office`,
  `lead_replied_wrong_person`, `lead_replied_not_interested`,
  `lead_replied_unsubscribed`, `lead_replied_uncategorized`, `lead_converted`,
  `lead_lost`, `blacklisted`. Delivery statuses (`email_sent`, `scheduled`, …) are
  set by the platform.
- **Mailboxes**: prefer OAuth for Google Workspace / Microsoft 365
  (`POST /v1/accounts/oauth/{google|microsoft}/init` returns a URL the user opens,
  then poll `.../status?state=`). SMTP/IMAP passwords are never returned.
  `failed` / `revoked` mailboxes do not send - `statusMessage` says why;
  `POST /v1/accounts/{id}/resume` re-tests a failed one.
- **Emails**: `GET /v1/emails` has no bodies; `GET /v1/emails/{id}` does. Replies and
  forwards go out from the mailbox that owns the conversation.
- **Pagination**: lists return `meta.nextCursor`; pass it back as `cursor`.
- **Retries**: send an `Idempotency-Key` header on writes. A retry with the same key
  and body within 24h returns the original response (`X-Idempotency-Replayed: true`)
  instead of repeating the action. Replies, forwards and test emails answer `503`
  (nothing sent, safe to retry) if duplicate protection is briefly unavailable.
- **Test emails**: `personalization: "lead"` with a `leadId` renders real data;
  `dummy` leaves lead tags empty. Replacing a campaign's `sequence` changes step and
  variant ids - re-read the campaign before `test-email`.

## Errors

Every error is `{ "error": { "type", "code", "message", "status", "requestId", "details"? } }`.
`400 validation_failed` (see `details`), `401 unauthenticated`,
`402 plan_limit_exceeded`, `403 permission_denied` (missing scope or plan),
`404 not_found`, `409 conflict`, `422 idempotency_key_reused`, `429 rate_limited`.
