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

## Why not OAuth?

An OAuth (3LO) surface was researched and prototyped, then deliberately dropped: the flow requires each caller to register an OAuth integration and complete a human consent dance that no app may automate, and Atlassian service accounts cannot hold Forge app scopes (verified Aug 2026). The API-key surface above delivers the same installation-scoped, revocable, scope-limited access with one click instead.
