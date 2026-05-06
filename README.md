# netcup Snapshot Automation

Automated weekly offline snapshots for [netcup](https://www.netcup.de/) vServers via the [SCP REST API](https://www.netcup.com/en/helpcenter/documentation/server/rest-api) and GitHub Actions.

## How it works

Every Monday at 04:00 UTC, the workflow runs for each configured server:

1. Checks if today's snapshot already exists (skips if so — no downtime)
2. Resolves the server's disk name and reads its current state
3. Stops the server (if running) — the REST API requires `SHUTOFF` for offline snapshots
4. Creates an offline snapshot named by date (e.g. `20260402`)
5. Deletes snapshots older than 28 days (only those matching this script's naming convention)
6. Restarts the server

Servers are processed in parallel. An EXIT trap ensures the server is **always** restarted, even if the workflow fails mid-execution. Per-server `concurrency` prevents overlapping runs from racing.

## Setup

### 1. GitHub Secrets

**Settings > Secrets and variables > Actions > Secrets**

| Secret | Description |
|--------|-------------|
| `SCP_USER` | Your SCP login username |
| `SCP_PASS` | Your SCP login password |

Same credentials as [servercontrolpanel.de](https://www.servercontrolpanel.de). No API key needed.

### 2. Server IDs

**Settings > Secrets and variables > Actions > Variables**

| Variable | Example value |
|----------|---------------|
| `SERVER_IDS` | `["123456", "789012"]` |

Must be a valid JSON array of strings. Find your server IDs in the SCP URL (e.g. `.../servers/123456`).

### 3. Schedule (optional)

Edit the cron expression in the workflow file:

```yaml
schedule:
  - cron: '0 4 * * 1'   # Mon 04:00 UTC (default)
```

Examples: `0 3 * * 0` (Sun 03:00), `0 2 * * 1,4` (Mon+Thu 02:00), `0 4 * * *` (daily 04:00).

You can also trigger manually from the Actions tab.

### 4. Tuning (optional)

Constants in the workflow's Configuration section:

| Constant | Default | Description |
|----------|---------|-------------|
| `SNAPSHOT_RETENTION_DAYS` | `28` | Delete snapshots older than this |
| `POLL_INTERVAL` | `5` | Seconds between status polls |
| `POLL_ATTEMPTS` | `12` | Max polls for state checks and short tasks (≈60 s) |
| `SNAPSHOT_POLL_ATTEMPTS` | `30` | Max polls for the snapshot task (≈150 s) |
| `RESTART_MAX_ATTEMPTS` | `3` | How many times to retry the restart in the recovery trap |

## Safety

- **Always restarts**: EXIT trap restarts the server even on failure
- **Duplicate-safe**: Skips if today's snapshot exists (no unnecessary downtime)
- **Token refresh**: Re-authenticates before restart and before each delete in case the token expired
- **Parallel-safe**: One server failing doesn't cancel others (`fail-fast: false`)
- **Concurrency-safe**: Per-server `concurrency` group prevents overlapping runs against the same server
- **Conservative cleanup**: Only deletes snapshots whose names match this script's pattern (`YYYYMMDD` or legacy `YYYY-MM-DD`); manual snapshots are preserved
- **Authoritative timestamps**: Cleanup uses the API's `creationTime`, not name parsing
- **Graceful cleanup**: Old snapshot deletion failures are warnings only

## Security

- Credentials stored as encrypted GitHub Secrets (never in code or git history)
- Server IDs stored as a GitHub Variable (not hardcoded)
- Auth body URL-encoded (passwords with `&`, `=`, `+`, `%` etc. won't corrupt the request)
- Access tokens masked in logs via `::add-mask::`
- API errors logged as a single `message` line — no raw response bodies
- Workflow `permissions: {}` removes all `GITHUB_TOKEN` scopes (workflow needs none)
- No repository checkout step — reduces attack surface

## License

MIT
