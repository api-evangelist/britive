---
name: britive-checkout-privileged-access
description: Check out a Britive privileged access profile just-in-time, retrieve console or programmatic credentials, and check it back in when the task is done.
api: Britive Services API
spec: openapi/britive-services-api-openapi.yml
operations:
  - getMyAccess
  - myAccessUI
  - getAccessProfileSetting
  - checkoutProfile
  - profileApprovalRequest
  - getTokens
  - url
  - checkinProfile
generated: '2026-08-08'
method: generated
source: openapi/britive-services-api-openapi.yml + https://docs.britive.com/apidocs
---

# Check out privileged access just-in-time

Britive grants no standing privilege. To act on a cloud or SaaS target you check out a
profile (a PAP), use the credentials it mints, and check it back in. The grant is bounded;
if you do not check in, it expires.

Base URL: `https://{tenant}.britive-app.com/api`
Auth: `Authorization: Bearer <token>` (single `bearerAuth` http/bearer scheme, global).

## Steps

1. **Find what you can check out.**
   `getMyAccess` — `GET /api/access` returns every profile the calling identity may check out.
   `myAccessUI` — `GET /api/access/frequently-used` returns the frequently-used subset.
   Each entry gives you the `profileId` and the `environmentId` you need next. Never guess
   these ids; there is no name-based lookup.

2. **Check the profile's settings before you check out.**
   `getAccessProfileSetting` — `GET /api/access/{profileId}/environments/{environmentId}/settings`.
   This tells you whether a justification is required, whether an ITSM ticket is required, and
   whether step-up authentication (an OTP) will be demanded. Reading this first avoids a
   round-trip rejection.

3. **If approval is required, raise it.**
   `profileApprovalRequest` — `POST /api/access/{profileId}/environments/{environmentId}/approvalRequest`
   with the justification. Approvers are notified over the tenant's configured notification
   medium. Do not poll aggressively; approval is a human step.

4. **Check out.**
   `checkoutProfile` — `POST /api/access/{profileId}/environments/{environmentId}`.
   The response carries a `transactionId`. Hold on to it — it is the only handle you have on
   the session.

5. **Get credentials.**
   For programmatic access: `getTokens` — `GET /api/access/{transactionId}/tokens`.
   For a browser console: `url` — `GET /api/access/{id}/url`.
   Treat both as secrets. Never log them, never write them to disk, never echo them back to a
   user in plain text unless the user explicitly asked for the credential.

6. **Check in as soon as you are done.**
   `checkinProfile` — `PUT /api/access/{transactionId}`.
   Check in the moment the task completes, not at the end of the conversation. Zero standing
   privilege is the point of the product; a session you forgot about is the failure mode it
   exists to prevent.

## Rules

- **Least privilege first.** Try the task with the access you already have. Only check out when
  it actually fails, and check out the narrowest profile that will work — read-only before
  administrator.
- **Errors.** Failures come back as `{status, message, errorCode, details}` on
  `application/json`. The `errorCode` is stable and namespaced: `MA-*` for check-in/check-out,
  `CO-*` and `CI-*` for application-specific check-out/check-in, `AS-*` when a justification or
  ITSM ticket is missing or malformed, `AU-*` for token problems. Read `errorCode`, not the
  message string. Full catalog: `errors/britive-problem-types.yml`.
- **`AS-0001`/`AS-0002`** mean the justification is missing or does not match the required
  regular expression — ask the user for a better justification rather than retrying.
- **`AU-0016`/`AU-0017`/`AU-0018`** mean the API token is revoked, expired or invalid.
  Re-authenticate; do not retry.
- **No idempotency keys.** Britive publishes no `Idempotency-Key` mechanism
  (`conventions/britive-conventions.yml`). A retried check-out is a *second* check-out.
  On a timeout, do not blindly retry — call `getCheckedOutProfilesForUser` first to see whether
  the session already exists.
- **Pagination.** List operations take `page` (default 0), `size` (default 20) and `sort`. No
  total count or next-link is returned, so stop when a page comes back short.
