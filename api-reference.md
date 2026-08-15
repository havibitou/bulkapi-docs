# API Reference

All endpoints are `POST`, accept `application/json`, and authenticate via
`Authorization: Bearer <api-key>` (see [authentication](authentication.md)).
Async operations return `202` with a `jobId`; results are read from `/status`.

Endpoint URLs are per-installation — discover them via `/setup` or the config
panel.

---

## POST /bulk

One endpoint, four operations selected by `operation`.

### `create` / `update`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `operation` | string | yes | `"create"` or `"update"` (both upsert — see `strict`) |
| `workspaceId` | string | yes | Assets workspace ID |
| `schemaName` | string | yes* | Enables importId auto-discovery (*or pass `importId`) |
| `importId` | string | no | Explicit import source; skips auto-discovery |
| `schemaId` | string | no | Auto-resolved from `schemaName` when omitted |
| `mapping` | object | yes | See [mapping format](#mapping-format) |
| `objects` | array | yes | 1–1000 entries (limit configurable) |
| `strict` | boolean | no | `update` only: downgrade to `partial` if anything was *created* |

Matching: the mapping attribute with `externalIdPart: true` is the identity.
Matching entries update existing objects; the rest are created. `strict: true`
turns silent creations into a diagnosed `partial`.

### `delete`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `operation` | string | yes | `"delete"` |
| `workspaceId` | string | yes | |
| `objectIds` | array | yes | 1–1000 numeric object IDs (as strings) |

Per-object REST deletes through a rate-limited parallel pool
(`rateLimitRps`, default 10/s). For large query-shaped deletions prefer `sync`.

### `sync` (destructive mirror)

Mirrors the mapped object type(s) to the submitted keep-set using the import
engine's Missing Objects: Remove mechanism — objects registered with the sync
source but absent from the data are deleted in one execution. Requires a
[dedicated destructive import source](../README.md#sync-destructive-mirror).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `operation` | string | yes | `"sync"` |
| `workspaceId` | string | yes | |
| `importId` | string | yes | The destructive source — never auto-discovered |
| `mapping` | object | yes | externalIdPart attribute must be the label attribute |
| `objects` | array | yes | The KEEP set (everything else dies) |
| `confirmDeleteAll` | boolean | when `objects` empty | Explicit ack for "delete all" |

Safety model: fail-closed preflight (source must verifiably have Missing
Objects = Remove with threshold 1), regular imports refused on destructive
sources, mapping never resubmitted, blind runs auto-retried
(see `sync` block on `/status`), terminal results count-verified.

---

## POST /status

### Job lookup / await

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `jobId` | string | yes | |
| `waitSeconds` | number | no | 0–20; long-poll until terminal or timeout |

Job record highlights:

```jsonc
{
  "status": "queued | running | completed | partial | failed",
  "succeeded": 998, "failed": 0, "deleted": 0,
  "errors": [],
  "importResult": {                    // per-type engine results
    "byObjectType": { "Movie": { "objectsCreated": 998, "objectsUpdated": 0,
                                  "objectsIdentical": 2, "objectsDeleted": 0, ... } },
    "startedAt": "...", "endedAt": "..."
  },
  "sync": {                            // sync jobs only — live lifecycle
    "phase": "queued | attempting | awaiting-eligibility | done | failed",
    "attempts": 2, "maxAttempts": 10,
    "expectedDeletions": 998,
    "lastAttemptAt": "...", "nextAttemptAt": "..."
  },
  "verification": {                    // sync terminal only
    "expectedRemaining": 2, "actualRemaining": 2, "verified": true
  }
}
```

### Auxiliary actions

- `{ "action": "list-imports" }` — known schema→importId mappings
- `{ "action": "get-progress", "jobId" | "importId" + "workspaceId" }` —
  live engine progress (percent, step)

---

## POST /search

Exactly one mode per request:

**Paged AQL** — `{ workspaceId, aql, page?, resultsPerPage?, includeAttributes? }`
→ `{ objects, total, page, resultsPerPage, isLast }`

**Complete set** — `{ workspaceId, aql, fetchAll: true }`
→ `{ objects, total, complete }` (cap 2000; `warning` set when truncated)

**Bulk get** — `{ workspaceId, objectKeys: [...] }` or
`{ workspaceId, objectTypeName, names: [...] }`
→ `{ objects, total, complete: true, notFound: [...] }`

Identifier lists are chunked into `IN` queries server-side (Cloud AQL key
property is `Key`). Up to `maxObjectsPerRequest` identifiers.

---

## POST /setup

`{ workspaceId }` → endpoint URLs, all schemas with object types, import
source readiness per schema, and setup instructions for those missing one.

---

## Mapping format

```json
{
  "objectTypeMappings": [{
    "objectTypeName": "Movie",
    "selector": "objects",
    "attributesMapping": [
      { "attributeName": "Name", "attributeLocators": ["Name"], "externalIdPart": true },
      { "attributeName": "Release Date", "attributeLocators": ["Release Date"] }
    ]
  }]
}
```

Attribute types are auto-detected from the live schema (cached 10 min) — text,
integer, double, date, datetime, select, etc. are sent correctly without any
type declarations.

## Limits & tuning (Jira admin → Apps → BulkAPI → Settings)

| Setting | Default | Notes |
|---------|---------|-------|
| Max objects per request | 1000 | |
| Import chunk size | 500 | chunks per KVS record / submitData call |
| Delete chunk size | 100 | |
| Delete rate limit | 10 rps | parallel pool saturates this |

## Errors

| Status | Meaning |
|--------|---------|
| 400 | Validation failure (`details` array) or guarded operation refused |
| 401 | Missing/invalid API key, or none configured |
| 403 | Key lacks the required scope (read for search/status/setup, write for create/update/delete, sync for sync) |
| 404 | Unknown jobId |
| 405 | Non-POST |
| 424 | No live installation behind this trigger URL (see [troubleshooting](troubleshooting.md)) |
| 500 | Unexpected failure |
