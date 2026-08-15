# Troubleshooting

Every entry here was hit for real during development. Symptoms first.

## "Where is Update?" — reading the Import tab

The Import tab lists **sources (pipes), not operations**. `create` and
`update` share the standard source — the engine upserts, and the per-execution
details split the outcome into *Created objects* vs *Updated objects*.
"Read details" shows only the **latest** execution; use *View history* for
earlier runs. Plain `delete` (by object IDs) bypasses imports entirely and
never appears here; only `sync` deletions run through the destructive source.

## HTTP 424 "Failed Dependency" from every endpoint

The web trigger URLs belong to an **installation**. A 424 from the gateway
means the invocation has no live installation behind it — the app was
uninstalled, or the site itself is gone (dev sites are deactivated after
inactivity; the error page says "Your Atlassian Cloud site is currently
unavailable"). Check `forge install list`. Reinstalling issues **new**
trigger URLs — rediscover via `forge webtrigger list` or Jira admin → Apps →
BulkAPI → Endpoints.

A *single* 424 among successes is different: one invocation failed (cold
start, platform flake). Retry it.

## Sync completes but deletes nothing (`deleted: 0`)

Three distinct causes, in the order to check:

1. **Threshold misconfigured.** Threshold N means "remove after the object is
   missing from N consecutive runs" — a large threshold reads as "never".
   The app refuses to run syncs when it can see this, and the config panel
   shows the per-type threshold. Fix: Edit mapping → Edit object type
   mapping → Threshold Number = 1.
2. **Eligibility ledger not caught up.** The engine only deletes objects that
   are *registered* with the sync source, and registrations become visible
   asynchronously (measured: anywhere from under a minute to ~19 minutes).
   The app handles this itself — blind attempts are detected against a live
   count and retried every 3 minutes (watch `sync.phase =
   "awaiting-eligibility"` and `sync.nextAttemptAt` on `/status`). If the job
   ends `failed` after 10 attempts, the diagnosis is in `errors`.
3. **Objects never registered.** Objects created *outside* the sync source
   join its ledger on their first sync run and become deletable on a later
   one (the probe-retry usually covers this automatically, since attempt 1
   registers them).

Instrumentation: sync runs log the engine's step trace — a blind run
visibly *skips* the "Handling missing objects" step.

## Regular import refused: "import source is destructive"

By design. A create/update on a Missing-Objects-Remove source would delete
everything absent from that batch. Point regular imports at your standard
import source; only `operation: "sync"` may target the destructive one.

## Mapping changes reset "Missing objects" to Ignore

Known platform behavior: any mapping PUT/PATCH silently drops the
`missingObjects` configuration (the field is UI-only; the API strips it).
This is why sync never resubmits mappings. If you edit the sync source's
attribute mappings, re-apply Missing objects = Remove + Threshold 1 in the UI
afterwards.

## AQL quirks

- The object key property is **`Key`**, not `objectKey` (`No matching
  attribute for AQL clause (objectKey)`).
- The search `total` field caps/lags on large result sets — treat it as a
  hint. The `/object/aql/totalcount` endpoint is authoritative (the app uses
  it for sync verification).
- Pagination is driven by `startAt`/`maxResults` **query parameters**;
  page-style fields in the request body are silently ignored.

## Delayed retry fires immediately

`delayInSeconds` must be a property **of the event object** pushed to the
queue (`{ body, delayInSeconds }`). Passing it as a second `push()` argument
is silently ignored. (Fixed in the app; documented here because it's easy to
reintroduce.)

## Job stuck in `running` with `pollTimedOut: true`

The consumer's poll window (~105 s) expired while the Assets execution was
still going. Nothing is lost: the next `/status` call checks live execution
state and finalizes the record when it's terminal (lazy finalization).

## 401 responses

| Message | Meaning |
|---|---|
| `Missing API key…` | Send `Authorization: Bearer <key>` or `auth` body field |
| `No API keys configured…` | Generate one in Jira admin → Apps → BulkAPI → API Keys |
| `Invalid API key.` | Wrong or revoked key |
