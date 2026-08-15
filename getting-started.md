# Getting started

BulkAPI adds a fast, key-secured REST API for bulk operations on Jira
Service Management **Assets** objects. Everything below happens in your
Jira site — no servers, no source code, no CLI.

## 1. Install

Install **BulkAPI — Bulk Assets API for JSM** from the Atlassian
Marketplace onto your Jira site (site admin required). Assets requires JSM
Premium/Enterprise.

## 2. Open the admin page

**Jira admin (⚙) → Apps → BulkAPI.** The Overview tab shows a setup
checklist and every Assets schema with its readiness.

## 3. Configure a schema (one click, usually)

Click **Configure** next to a schema. The app creates and wires the import
sources it needs. If your site refuses app-side creation, the panel guides
you: create the import in Assets (schema names link straight there:
*Create import → Bulk Assets Import*), then on the import's row **⋯ →
Configure app → Register with BulkAPI**.

## 4. Create an API key

**API Keys tab → Generate.** Pick scopes: `read` (search/status/setup),
`write` (create/update/delete), `sync` (destructive mirror — leave off
unless needed). The key is shown once. One key works for every schema.

## 5. First call

Endpoint URLs live on the **Endpoints** tab; full request contracts with
prefilled examples on the **Documentation** tab. A first create:

```bash
curl -s -X POST "<bulk-operations-url>" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer bak_YOURKEY" \
  -d '{
    "operation": "create",
    "schemaName": "Movies",
    "mapping": {"objectTypeMappings": [{
      "objectTypeName": "Movie", "selector": "objects",
      "attributesMapping": [
        {"attributeName": "Name", "attributeLocators": ["Name"], "externalIdPart": true},
        {"attributeName": "Title", "attributeLocators": ["Title"]}
      ]}]},
    "objects": [{"Name": "Alien", "Title": "v1"}]
  }'
```

The response is `{"jobId": "job_…"}` — poll the Job Status endpoint with
`{"jobId": "...", "waitSeconds": 20}` until `completed`.

### Python (stdlib only)

```python
import json, urllib.request

def call(url, body, key):
    req = urllib.request.Request(url, json.dumps(body).encode(),
        {"Content-Type": "application/json", "Authorization": f"Bearer {key}"})
    return json.load(urllib.request.urlopen(req))

job = call(BULK_URL, {"operation": "create", "schemaName": "Movies",
                      "mapping": MAPPING, "objects": OBJECTS}, KEY)
while True:
    s = call(STATUS_URL, {"jobId": job["jobId"], "waitSeconds": 20}, KEY)
    if s["job"]["status"] in ("completed", "partial", "failed"): break
print(s["job"]["importResult"])
```

## 6. Safety rails you get for free

- `update` with `"strict": true` refuses to silently create.
- `delete` and `sync` support `"dryRun": true` — see exactly what would be
  deleted before anything is.
- Destructive sync requires a dedicated armed source, a `sync`-scoped key,
  and explicit confirmation for anything that wipes a whole type.
- **Deletions are permanent** — Assets has no recycle bin. Dry-run first.

See the [API reference](api-reference) for every field, and
[Performance](performance) for what "fast" means in numbers.
