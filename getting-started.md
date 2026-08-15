# Getting Started

From zero to your first bulk import in about ten minutes.

## 1. Deploy and install

```bash
npm install
forge deploy
forge install --site <your-site>.atlassian.net --product jira
```

## 2. Configure your schema — one click

Open **Jira admin → Apps → BulkAPI**. A **Setup wizard** at the top tracks the
whole path live — workspace → schemas → scoped API key → sync switch — and
disappears once everything is green. Every schema without an import source
shows a **Configure for BulkAPI** button. One click:

- creates the standard source ("BulkAPI Create + Update (…)") with an
  auto-built mapping over every object type (label attribute as identity,
  all simple attributes included, selector = object type name),
- registers it for `schemaName` auto-discovery — create/update work
  immediately,
- creates the sync source ("BulkAPI Sync — destructive (…)") with an
  identity-only mapping. One manual step remains (the setting is UI-only on
  Atlassian's side): Edit mapping → each object type → **Missing objects:
  Remove, Threshold 1**. Until you flip it, sync calls are safely refused.

Prefer manual setup, or need custom mappings? The step-by-step below still
works.

## 2b. Manual setup (alternative)

1. **Object schema + types**: create them in Assets, or script it — see
   [`scripts/create-dev-schema.js`](../scripts/create-dev-schema.js) for a
   REST-based bootstrap you can adapt.
2. **Import source**: in the schema, *Schema settings → Import → Create
   import → "Bulk Assets Import"*. Then **register it with BulkAPI**: on the
   new import's row, *⋯ menu → Configure app → Register as Create + Update
   source* (or *Register as Sync (destructive) source*). Registration seeds
   the attribute mapping automatically and wires the source into
   `schemaName` auto-discovery — attributes whose type the import engine
   cannot represent (tag lists, Time) are skipped and listed; add those by
   hand in Edit mapping if you need them.

   **Naming tip** — import sources are pipes, not operations. One source
   carries both `create` and `update` (the engine upserts); `delete` by IDs
   never touches an import source at all; `sync` needs its own destructive
   source. Name them for their role so the Import tab reads clearly:
   - `BulkAPI Create + Update (<schema>)` — the standard source
   - `BulkAPI Sync — destructive (<schema>)` — the Missing Objects: Remove source
3. **API key**: in *Jira admin → Apps → BulkAPI → API Keys → Generate* — the
   single place keys are managed. Copy it immediately — it is shown once.
   Endpoint URLs live on the same admin page (Endpoints tab); the import's
   own Configure app panel shows its Import ID and Workspace ID.

## 3. Discover your endpoints

```bash
curl -s -X POST "<setup-url>" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <api-key>" \
  -d '{"workspaceId": "<workspace-id>"}'
```

The response lists every endpoint, every schema with its object types, and
which schemas still need an import source.

## 4. First import

```bash
curl -s -X POST "<bulk-url>" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <api-key>" \
  -d '{
    "operation": "create",
    "workspaceId": "<workspace-id>",
    "schemaName": "Movies",
    "mapping": {
      "objectTypeMappings": [{
        "objectTypeName": "Movie",
        "selector": "objects",
        "attributesMapping": [
          { "attributeName": "Name", "attributeLocators": ["Name"], "externalIdPart": true },
          { "attributeName": "Title", "attributeLocators": ["Title"] }
        ]
      }]
    },
    "objects": [
      { "Name": "Inception", "Title": "Inception" }
    ]
  }'
```

You get `202 Accepted` with a `jobId`.

## 5. Await the result

```bash
curl -s -X POST "<status-url>" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <api-key>" \
  -d '{"jobId": "<jobId>", "waitSeconds": 20}'
```

`waitSeconds` holds the response until the job is terminal (or 20 s passes) —
loop on it and you have an await, no client-side timers. A finished job carries
per-type results (`objectsCreated`, `objectsUpdated`, `objectsIdentical`,
`objectsDeleted`) and, for destructive syncs, a count-verified `verification`
block.

## Next steps

- [API reference](api-reference.md) — every operation and field
- [Authentication](authentication.md) — keys vs OAuth, and the research behind them
- [Troubleshooting](troubleshooting.md) — the platform's sharp edges, documented
- [Examples](../examples/) — Node client, Python, curl cookbook
