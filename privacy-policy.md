# BulkAPI — Privacy Policy

**Publisher:** HaviBit · **App:** BulkAPI — Bulk Assets API for JSM (Atlassian Forge app)
**Last updated:** 2026-08-16

## What the app is

BulkAPI runs entirely on Atlassian's Forge platform, inside Atlassian's
infrastructure. HaviBit operates no servers of its own and receives no copy
of your data.

## Data the app stores

All storage is Atlassian Forge hosted storage (KVS), scoped to your
installation and deleted by Atlassian when the app is uninstalled:

- **API key records** — a label, scope list, creation date, key prefix, and
  a SHA-256 hash of each key. Raw keys are never stored.
- **Configuration** — rate limits, chunk sizes, workspace ID, and which
  import sources belong to which schemas.
- **Job records** — operation type, status, counts, and error messages for
  bulk jobs, plus temporary payload chunks while a job is running.

## Data the app processes

Object data submitted to the API (your Assets records) passes through the
app only to be handed to Atlassian's own Assets Import APIs. It is processed
transiently and is not sent anywhere except Atlassian's `api.atlassian.com`.

## What the app does NOT do

- No third-party services, analytics, tracking, or advertising.
- No personal data collection beyond what Atlassian provides to operate the
  app inside your site.
- No data leaves Atlassian's infrastructure.

## Your callers' data

Requests to the app's API endpoints (URL, payload) are handled inside Forge
and logged only in Atlassian's Forge logs, which the publisher can access
solely for support and debugging.

## Contact

Privacy questions: the support contact listed on the Atlassian Marketplace
listing for BulkAPI.
