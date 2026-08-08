---
name: britive-retrieve-secret
description: Retrieve a secret from the Britive vault — resolve the vault, list the secret tree, and access a secret value with justification and step-up authentication where the policy demands it.
api: Britive Secrets Manager API
spec: openapi/britive-secrets-manager-api-openapi.yml
operations:
  - getVaults
  - getVault
  - getOneLevelSecretData
  - getSecretMetaData
  - getSecretDataPost
  - getSecretData
  - downloadFile
  - listSecretVersions
  - getSecretDataForVersion
  - getApprovals
generated: '2026-08-08'
method: generated
source: openapi/britive-secrets-manager-api-openapi.yml + https://docs.britive.com/apidocs
---

# Retrieve a secret from the Britive vault

The Secret Manager surface is versioned: `https://{tenant}.britive-app.com/api/v1`.

## Steps

1. **Resolve the vault.**
   `getVaults` — `GET /api/v1/secretmanager/vault` returns the tenant's vaults. Take the vault
   `id` from here; there is only ever one in most tenants, but do not hardcode it.
   `getVault` — `GET /api/v1/secretmanager/vault/{id}` for detail.

2. **Walk the secret tree.**
   `getOneLevelSecretData` — `GET /api/v1/secretmanager/vault/{vaultId}/secrets` lists one level
   at a time. Secrets are addressed by **path**, not by a flat id, so navigate down rather than
   trying to list everything.

3. **Read metadata before reading the value.**
   `getSecretMetaData` — `GET /api/v1/secretmanager/vault/{vaultId}/secret-metadata`. Metadata
   tells you what a secret is without exposing it — prefer this whenever the user's question can
   be answered without the value.

4. **Access the value.**
   `getSecretDataPost` — `POST /api/v1/secretmanager/vault/{vaultId}/accesssecrets` (the SDK and
   the MCP `my_secrets_view` tool both use the POST form, because justification and OTP travel in
   the body). `getSecretData` — `GET` on the same path — is the read-only variant.
   Supply `justification` when the policy requires one, and `otp` when step-up authentication is
   demanded.

5. **Files and versions.**
   `downloadFile` — `GET /api/v1/secretmanager/vault/{vaultId}/downloadfile` for file secrets.
   `listSecretVersions` / `getSecretDataForVersion` —
   `GET .../secrets/{secretId}/versions[/{version}]` for history.

6. **If approval is required.**
   `getApprovals` — `GET /api/v1/approvals/` shows the pending request. Approval is a human step;
   surface the request id to the user and wait rather than polling hard.

## Rules

- **A secret value is not conversation text.** Never echo a retrieved secret into a summary, a
  log line, a commit, a file, or a follow-up tool call that does not need it. Use it and drop it.
  If the user only needs to know that a secret exists or when it was rotated, use metadata and
  never call the access operation at all.
- **Access is audited.** Every access is recorded and appears in secret-access reporting. Assume
  the user knows this and say so if they seem not to.
- **No idempotency keys.** Retrying a write against the vault is a second write, not a replay.
  See `conventions/britive-conventions.yml`.
- **Errors.** `{status, message, errorCode, details}`. `AS-0001`/`AS-0002` mean the justification
  is missing or does not match the required pattern; `AS-0003`–`AS-0007` cover missing or invalid
  ITSM ticket type/id; `AU-0009` means the OTP failed validation. See
  `errors/britive-problem-types.yml`.
