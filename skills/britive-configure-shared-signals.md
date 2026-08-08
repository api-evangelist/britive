---
name: britive-configure-shared-signals
description: Wire Britive into a Shared Signals Framework ecosystem — register an inbound issuer and map CAEP/RISC events to Britive actions, or register an outbound receiver and map Britive events to standard event URIs.
api: Britive Services API
spec: openapi/britive-services-api-openapi.yml
operations:
  - getSsfCatalog
  - listSsfIssuers
  - createSsfIssuer
  - getSsfIssuer
  - updateSsfIssuer
  - deleteSsfIssuer
  - getIssuerMappings
  - putIssuerMappings
  - listSsfReceivers
  - createSsfReceiver
  - getReceiverMappings
  - putReceiverMappings
  - querySsfResults
generated: '2026-08-08'
method: generated
source: openapi/britive-services-api-openapi.yml + https://docs.britive.com/docs/shared-signals-framework
---

# Configure Shared Signals (SSF / CAEP / RISC)

Britive both consumes and emits Shared Signals Framework security events, carried as Security
Event Tokens (RFC 8417). Inbound sources are configured as **issuers** (SSF calls them
transmitters); outbound destinations are configured as **receivers**.

Base URL: `https://{tenant}.britive-app.com/api`

## Always start with the catalog

`getSsfCatalog` — `GET /api/ssf/admin/catalog` returns the supported event vocabulary and the
Britive actions it can be mapped to (`SsfEventDefinition {type, uri, name, description}` and
`BritiveEventTypeDefinition {type, displayName, description}`). Every event URI and action you
use in a mapping must come from this response. Do not hand-write CAEP or RISC URIs from memory.

## Inbound — Britive reacts to someone else's signals

1. `listSsfIssuers` — `GET /api/ssf/admin/issuers` to see what is already configured.
2. `createSsfIssuer` — `POST /api/ssf/admin/issuers` with
   `CreateIssuerRequest {issuerUrl, displayName, description, enabled}`.
   **The issuer URL is immutable.** Britive's documentation is explicit: once created it cannot
   be changed — to use a different URL you delete and re-create the issuer. Confirm the URL with
   the user before creating.
3. `getIssuerMappings` / `putIssuerMappings` —
   `GET`/`PUT /api/ssf/admin/issuers/{issuerId}/mappings` with
   `EventActionMapping {eventUri, action}`. `PUT` **replaces** the whole mapping set; read the
   current mappings first and send the merged list, or you will silently drop mappings.

### The three Britive actions, in increasing severity

| Action | Effect |
| --- | --- |
| `Force Checkin` | Revokes the identity's active privileged sessions. They can re-authenticate and check out again. |
| `Force Logout` | Force Checkin, plus invalidates all authentication sessions. Full re-authentication required. |
| `Disable User` | Force Logout, plus disables the account until an administrator re-enables it. |

Actions are cumulative. Choose the least severe action that satisfies the intent, and state
plainly which one you are mapping — `Disable User` locks a person out of their job.

## Outbound — Britive tells someone else

1. `listSsfReceivers` / `createSsfReceiver` — `GET`/`POST /api/ssf/admin/receivers` with
   `CreateReceiverRequest {receiverUrl, displayName, description, enabled, headers}`.
   Custom headers (for example an `Authorization` token for the receiver) are stored securely
   and are never returned in plaintext — do not attempt to read them back.
2. `getReceiverMappings` / `putReceiverMappings` —
   `GET`/`PUT /api/ssf/admin/receivers/{receiverId}/mappings` with
   `ReceiverEventMapping {britiveEventType, eventUri}`. Same replace-not-merge semantics.

## Verify

`querySsfResults` — `GET /api/ssf/results` returns every processed event in both directions with
its type, subject, action taken, outcome and timestamp. **Results are retained for 90 days.**
Confirm a mapping works by finding its result here, not by trusting the 2xx on the `PUT`.

## Rules

- `PUT` mapping operations are full replacements. Read, merge, then write.
- Never invent an event URI or an action name — take both from `getSsfCatalog`.
- Britive warns that API-layer caching can leave an identity with access for a short period
  after a `Force Logout` or `Disable User`. Do not describe the effect as instantaneous.
- Errors: `{status, message, errorCode, details}`. See `errors/britive-problem-types.yml`.
