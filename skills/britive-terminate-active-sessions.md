---
name: britive-terminate-active-sessions
description: Find and forcibly check in a user's active Britive privileged sessions during an incident, and confirm the revocation in the audit log.
api: Britive Services API
spec: openapi/britive-services-api-openapi.yml
operations:
  - getCheckedOutProfilesForAdmin_1
  - getCheckedOutProfilesForUser
  - checkInProfileForTransaction
  - checkInAllProfilesForUser
  - disabledStatuses
  - getAuditLogs
generated: '2026-08-08'
method: generated
source: openapi/britive-services-api-openapi.yml + https://docs.britive.com/apidocs
---

# Terminate active privileged sessions

Incident response: an identity is suspected compromised and every privilege it currently holds
must be revoked. This is an administrative flow — it requires an identity with session
administration rights, not the self-service `my access` surface.

Base URL: `https://{tenant}.britive-app.com/api`

## Steps

1. **See who holds active access.**
   `getCheckedOutProfilesForAdmin_1` — `GET /api/paps/sessions` lists active sessions across all
   users. Supports `searchText`, `page`, `size` and `sort`.

2. **Scope to the identity.**
   `getCheckedOutProfilesForUser` — `GET /api/paps/sessions/{id}` returns that user's active
   sessions with their `transactionId`s.

3. **Revoke.**
   - Surgical: `checkInProfileForTransaction` — `DELETE /api/paps/sessions/{id}` ends one session
     by transaction id. This operation is naturally idempotent: checking in an already
     checked-in session succeeds with no effect, so it is safe to repeat.
   - Full revoke: `checkInAllProfilesForUser` — `DELETE /api/paps/sessions/user/{userId}` ends
     every application session for that user in one call.
   - Sessions whose access type is `DEVICE` **cannot** be checked in through this API. If you
     see one, say so explicitly rather than reporting a clean revoke.

4. **Escalate if the identity itself is compromised.**
   `disabledStatuses` — `POST /api/users/{targetUserId}/disabled-statuses` disables the account
   so it cannot log in and check out again. Only do this when asked; it is more severe than
   revoking sessions and requires an administrator to undo.

5. **Prove it.**
   `getAuditLogs` — `GET /api/logs`. Each check-in emits an `ADMIN_ACCESS_CHECKIN` audit event.
   Build the filter with the vocabulary from `getFields` and `getOperators` rather than guessing
   field names. Report the audit events as the evidence of revocation — do not claim success
   from the 2xx alone.

## Rules

- **Say what you actually revoked.** Report per-session outcomes. A partial revoke reported as a
  full revoke is the worst possible failure in this flow.
- **Timing caveat.** Britive's own documentation warns that because of API-layer caching an
  identity may retain access briefly after a forced logout or disable. Do not tell a responder
  access ended "immediately".
- **Resource sessions are separate.** Access Broker resource sessions live under
  `/api/resource-manager/...`, not `/api/paps/sessions`. Revoking application profiles does not
  revoke resource checkouts.
- **Errors.** `{status, message, errorCode, details}`. `RM-0003` means check-in of that profile
  is not allowed, `RM-0004` an invalid transaction, `RM-0005` the caller is not authorized to
  perform the check-in. See `errors/britive-problem-types.yml`.
