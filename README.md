# netcup Snapshot Automation

Automated weekly offline snapshots for [netcup](https://www.netcup.de/) vServers via the [SCP REST API](https://www.netcup.com/en/helpcenter/documentation/server/rest-api) and GitHub Actions.

## How it works

Every Monday at 04:00 UTC, for each server in `SERVER_IDS`:

1. Skip if today’s snapshot already exists (no downtime).
1. Resolve the disk name and read the current power state.
1. Gracefully stop the server (ACPI `OFF`) if running — the API requires `SHUTOFF` for an offline snapshot.
1. Create an offline snapshot named `YYYYMMDD`.
1. Delete this script’s own snapshots older than `SNAPSHOT_RETENTION_DAYS`.
1. Restart the server.

Servers run in parallel. An `EXIT` trap — with `TERM`/`INT` routed into it — guarantees a restart attempt even on failure, cancellation, or timeout. Per-server `concurrency` stops overlapping runs from racing.

**Transient write locks.** netcup serializes writes per server: a write issued while a previous state change or snapshot is still settling is rejected with **409** (undocumented runtime guard) or **503** (documented maintenance). The write never started, so every state-changing call retries through both until it serializes or `LOCK_RETRY_DEADLINE` is hit. The workflow comments carry the full rationale and the enum/state details, all verified against the live spec (`GET /scp-core/api/v1/openapi`).

## Setup

### 1. Secrets — *Settings → Secrets and variables → Actions → Secrets*

|Secret    |Description       |
|----------|------------------|
|`SCP_USER`|SCP login username|
|`SCP_PASS`|SCP login password|

Same credentials as [servercontrolpanel.de](https://www.servercontrolpanel.de) — no API key needed.

### 2. Server IDs — *… → Variables*

|Variable    |Example               |
|------------|----------------------|
|`SERVER_IDS`|`["123456", "789012"]`|

A JSON array of strings. Find each ID in the SCP URL (`.../servers/123456`).

### 3. Schedule (optional)

```yaml
schedule:
  - cron: '0 4 * * 1'   # Mon 04:00 UTC (default)
```

e.g. `0 3 * * 0` (Sun 03:00), `0 2 * * 1,4` (Mon+Thu 02:00), `0 4 * * *` (daily). Also runnable manually from the Actions tab.

### 4. Tuning (optional)

|Constant                 |Default|Description                                       |
|-------------------------|-------|--------------------------------------------------|
|`SNAPSHOT_RETENTION_DAYS`|`28`   |Delete snapshots older than this                  |
|`POLL_INTERVAL`          |`5`    |Seconds between status polls                      |
|`POLL_ATTEMPTS`          |`12`   |Max polls for state checks / short tasks (~60 s)  |
|`SNAPSHOT_POLL_ATTEMPTS` |`30`   |Max polls for the snapshot task (~150 s)          |
|`RESTART_MAX_ATTEMPTS`   |`3`    |Restart retries in the recovery trap              |
|`LOCK_RETRY_INTERVAL`    |`10`   |Seconds between 409/503 write retries             |
|`LOCK_RETRY_DEADLINE`    |`180`  |Max seconds to wait out a write lock / maintenance|

The per-server worst case (three write deadlines plus the poll budgets) sits under `timeout-minutes: 25`. If you raise `LOCK_RETRY_DEADLINE`, raise `timeout-minutes` to match — the restart trap only runs if the runner isn’t killed first.

## Safety & security

- **Always restarts.** `EXIT` trap restarts on any failure; `TERM`/`INT` route into it so cancellation/timeout still attempts a restart. A failed restart exits non-zero, so the job goes red instead of silently green.
- **No needless downtime.** Skips when today’s snapshot exists; never powers on a server that was already off.
- **Transient-safe.** Writes retry through 409/503; only idempotent `GET`s are retried at the transport layer, so a write is never blindly re-sent.
- **Conservative cleanup.** Deletes only snapshots named `YYYYMMDD`/`YYYY-MM-DD` older than retention, by the API’s `creationTime` — manual snapshots are preserved.
- **Token refresh** before each long-running step, in case the access token expired.
- **Hardened.** Secrets in encrypted GitHub Secrets, IDs in a Variable, URL-encoded auth body, tokens masked via `::add-mask::`, only a single `message` line logged (no raw bodies), `permissions: {}` and no checkout step.

## Optional (not enabled)

- **Pre-stop dry run** (`POST …/snapshots:dryrun`) — could catch a structurally impossible snapshot before stopping, but isn’t confirmed safe to trust against a *running* server.
- **Forced power-off** (`PATCH …?stateOption=POWEROFF`) — defeats the point of a consistent snapshot; prefer investigating a guest that won’t halt cleanly.

## License

MIT
