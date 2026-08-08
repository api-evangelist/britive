---
name: britive-query-audit-logs
description: Query the Britive audit log with a correctly-built filter expression, export it as CSV, and register a webhook to stream matching events out.
api: Britive Services API
spec: openapi/britive-services-api-openapi.yml
operations:
  - getFields
  - getOperators
  - getAuditLogs
  - getAuditLogsCsv
  - getAuditLogWebhooks
  - saveAuditLogWebhook
  - getAuditLogWebhookByNotificationMediumId
  - deleteAuditLogWebhookByNotificationMediumId
generated: '2026-08-08'
method: generated
source: openapi/britive-services-api-openapi.yml + https://docs.britive.com/apidocs
---

# Query and stream the Britive audit log

The audit log is queried with a filter-expression language whose vocabulary is served by the API
itself. Discover the vocabulary before you write the expression — do not invent field names.

Base URL: `https://{tenant}.britive-app.com/api`

## Steps

1. **Discover the field vocabulary.**
   `getFields` — `GET /api/logs/fields`.

2. **Discover the operator vocabulary.**
   `getOperators` — `GET /api/logs/operators`.

3. **Query.**
   `getAuditLogs` — `GET /api/logs` with the time window and the filter expression built only
   from the two vocabularies above.

4. **Export.**
   `getAuditLogsCsv` — `GET /api/logs/csv` returns the same result as CSV. This is a separate
   operation, not content negotiation — do not send `Accept: text/csv` to `/api/logs`.

5. **Stream it out (optional).**
   `getAuditLogWebhooks` — `GET /api/logs/webhooks` lists configured webhooks.
   `saveAuditLogWebhook` — `POST /api/logs/webhooks` registers one. The body is
   `AuditLogWebhook {notificationMediumId, filter, description}` — a webhook is bound to a
   **notification medium**, not to a raw URL, so the medium (Teams incoming webhook, Slack,
   email) must exist first.
   `getAuditLogWebhookByNotificationMediumId` / `deleteAuditLogWebhookByNotificationMediumId` —
   `GET`/`DELETE /api/logs/webhooks/{notificationMediumId}`.

## Rules

- **Never fabricate a field or operator.** If the requested filter cannot be expressed in the
  discovered vocabulary, say so; a silently-wrong audit query is worse than no answer.
- **No delivery guarantees are published.** Britive documents no retry policy, ordering
  guarantee or webhook signature scheme. Do not tell a user that deliveries are signed,
  verified, ordered or retried. See `asyncapi/britive-events-webhooks.yml`.
- **No payload schema is published** for what Britive POSTs to a webhook. Do not assert a
  payload shape you have not observed.
- **Pagination.** `page` (default 0), `size` (default 20), `sort`. No total count is returned.
- **Errors.** `{status, message, errorCode, details}`; `R-0001` covers report/content load
  failures. See `errors/britive-problem-types.yml`.
