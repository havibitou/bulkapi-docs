# Performance

Measured on a Jira Cloud dev site (Aug 2026), 1000 objects per job unless
noted. The comparison baseline is the conventional alternative: calling
Atlassian's single-object REST endpoints once per object, serially, at
typical Cloud latency (~300 ms/request).

| Operation (1000 objects) | BulkAPI | One-request-per-object REST | Speedup |
|---|---|---|---|
| Create | **~17 s** | ~5 min (serial ~300 ms each) | ~18× |
| Update (strict) | **~19 s** | ~5 min | ~16× |
| Delete by IDs | **~100 s** (parallel pool, rate-capped) | ~7 min | ~4× |
| Mirror-delete via sync | **~10 s** for 2 999 objects in one execution | ~15 min | ~90× |

```
Create 1000 objects (seconds, lower is better)

BulkAPI      ██ 17
Per-object   ████████████████████████████████████ ~300
```

Why it's fast: one HTTP call carries the whole batch; the app chunks it
through Atlassian's own bulk import engine (500 objects per submission),
polls aggressively, and for deletes runs a parallel worker pool at the
configured rate limit. The floor (~10–15 s per import execution) is
Atlassian-side engine time, not the app.

Numbers vary with site load, attribute counts, and rate limits — treat
them as representative, not contractual. The app ships a benchmark-style
integration suite; ask support if you want to reproduce the methodology.
