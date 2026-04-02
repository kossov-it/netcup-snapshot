# netcup Snapshot Automation

Automated weekly offline snapshots for [netcup](https://www.netcup.de/) vServers via the [SCP REST API](https://helpcenter.netcup.com/en/wiki/server/rest-api) and GitHub Actions.

## How it works

Every Monday at 04:00 UTC, the workflow runs for each configured server:

1. Checks if today's snapshot already exists (skips if so — no downtime)
2. Stops the server (if running)
3. Creates an offline snapshot named by date (e.g. `20260402`)
4. Deletes snapshots older than 28 days
5. Restarts the server

Servers are processed in parallel. An EXIT trap ensures the server is **always** restarted, even if the workflow fails mid-execution.

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
| `TASK_TIMEOUT_ATTEMPTS` | `30` | Max polls for snapshot task (30 x 5s = 150s) |
| `POLL_INTERVAL` | `5` | Seconds between status polls |

## Safety

- **Always restarts**: EXIT trap restarts the server even on failure
- **Duplicate-safe**: Skips if today's snapshot exists (no unnecessary downtime)
- **Token refresh**: Re-authenticates before restart in case the token expired
- **Parallel-safe**: One server failing doesn't cancel others (`fail-fast: false`)
- **Graceful cleanup**: Old snapshot deletion failures are warnings only

## Security

- Credentials stored as encrypted GitHub Secrets (never in code or git history)
- Server IDs stored as a GitHub Variable (not hardcoded)
- Access tokens masked in logs via `::add-mask::`
- Auth errors don't leak raw API responses
- No repository checkout step — reduces attack surface

## License

MIT
