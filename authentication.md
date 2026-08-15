# Authentication

BulkAPI exposes two API surfaces with different authentication models. This
document explains both, and records the research behind the choices
(last verified: August 2026).

## Surface 1: Web triggers + per-installation API keys (recommended)

The four web trigger endpoints (`bulk`, `status`, `search`, `setup`)
authenticate with **per-installation API keys**:

- Generated in **Jira admin → Apps → BulkAPI → API Keys** — the single place
  keys are managed. Shown exactly once.
- Sent as `Authorization: Bearer bak_...` (preferred) or as the `auth` field
  in the JSON body.
- Stored as SHA-256 hashes in Forge KVS — inherently scoped to one
  installation; there is nothing shared across customers.
- Up to 10 active keys per installation; revocation is instant.
- **Keys carry scopes** — grant each caller only what it needs:

  | Scope | Grants |
  |-------|--------|
  | `read` | `/search`, `/status`, `/setup` |
  | `write` | `create`, `update`, `delete` operations |
  | `sync` | destructive mirror syncs |

  A request outside the key's scopes gets `403` with the missing scope named.
  Keys created before scopes existed are treated as full-access.

```bash
curl -s -X POST "$BULK_URL" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer bak_..." \
  -d '{ "operation": "create", ... }'
```

**Why not Atlassian API tokens?** Atlassian is actively migrating apps and
integrations *away* from user API tokens: Marketplace apps must not collect or
store them, token traffic is rate-limited separately, and all tokens now
expire. An app-issued key with per-installation storage is the compliant
equivalent, with the same developer ergonomics.

## Surface 2: Forge app REST APIs (`apiRoute`) + OAuth 2.0 (3LO)

The `/v1/*` apiRoute endpoints are hosted by Atlassian and authenticated by
Atlassian — the app never sees credentials:

1. The customer (or their integrator) creates an **OAuth 2.0 (3LO)
   integration** in the [developer console](https://developer.atlassian.com/console),
   selects this app, and grants its custom scopes
   (`read:bulk-assets:custom`, `write:bulk-assets:custom`).
2. A user completes the consent flow once; the integration then exchanges
   authorization codes for access tokens.
3. Requests carry `Authorization: Bearer <3LO access token>`.

This is Atlassian's officially supported path for app-exposed REST APIs. Its
cost is the interactive consent step and refresh-token plumbing.

## Research: can authentication be simplified? (August 2026)

| Option | Status | Verdict |
|--------|--------|---------|
| Atlassian API tokens against our endpoints | Actively discouraged platform-wide; apps must not collect them | Not viable |
| **Service accounts + OAuth client credentials** | Shipped Oct 2025 — machine-to-machine, no browser consent, no long-lived secrets | **Product REST APIs only** (Jira/Confluence scopes). Does **not** support Forge app custom scopes, so it cannot call our apiRoutes |
| apiRoute + client credentials | Not supported — apiRoute remains Preview and documents 3LO authorization-code only | Watch item |
| Per-installation app keys (current) | Fully under our control | Remains the recommended default |

**Recommendations:**

- Keep per-installation keys as the primary path — simplest onboarding, no
  Atlassian-side setup, Marketplace-compliant.
- Offer apiRoute + 3LO for organizations that mandate Atlassian-native OAuth.
- For callers that also use Jira/Confluence product APIs directly, point them
  at **service accounts with client credentials** for that traffic — org-managed,
  non-human identity with no stored secrets.
- Re-check periodically whether Atlassian extends client credentials to Forge
  app scopes; if that lands, it becomes the natural replacement for both paths
  (machine auth against our apiRoutes with zero app-side key management).

Sources: [Access REST APIs exposed by a Forge app](https://developer.atlassian.com/platform/forge/access-rest-apis-exposed-by-a-forge-app/),
[OAuth support for service accounts](https://community.atlassian.com/forums/Trust-Security-articles/OAuth-Support-for-Service-Accounts/ba-p/3126134),
[Migrate from API tokens](https://community.developer.atlassian.com/t/reminder-migrate-from-using-api-tokens-to-officially-supported-authentication-for-atlassian-apps-integrations/97221)
