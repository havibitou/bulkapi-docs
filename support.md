# Support

**Email:** `support@havibit.eu`

We aim to answer within **2 business days** (EET/EEST, Estonia). Trial and
evaluation questions are just as welcome as customer issues.

## Before you write

The [Troubleshooting guide](troubleshooting) covers the most common cases —
401/403 responses, "no import source" errors, sync refusals, and mapping
rejections — usually with a one-line fix.

## What to include in a bug report

Copy-paste friendly checklist:

1. **What you called** — endpoint (bulk / status / search / setup) and the
   request body *with the API key removed*.
2. **What came back** — full JSON response, and the `jobId` for any job.
3. **When** — date and time with timezone (Forge logs are UTC).
4. **Where** — your Jira site URL and the schema/object type involved.
5. **Expected vs. actual** — one sentence each.

Never include an API key, Atlassian API token, or password in a support
email. Keys can be revoked and reissued in seconds from **Jira admin →
Apps → BulkAPI → API Keys** if one leaks.

## Security reports

Suspected vulnerabilities: same address, subject starting **[SECURITY]** —
these are read first. Please practice responsible disclosure; we will
acknowledge, fix, and credit.

## Feature requests

Also welcome by email. Requests that describe the *job you are trying to
do* (not just the feature) get built sooner.
