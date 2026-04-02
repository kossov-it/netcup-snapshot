# netcup Snapshot Automation

Automated weekly offline snapshots for [netcup](https://www.netcup.de/) vServers using the SCP REST API and GitHub Actions.

## What it does

Every Monday at 04:00 UTC, the workflow:

1. Authenticates with the [Server Control Panel (SCP)](https://www.servercontrolpanel.de) REST API
2. Checks if today's snapshot already exists (skips if so — no unnecessary downtime)
3. Stops each server (if running)
4. Creates an offline snapshot (named by date, e.g. `2026-04-02`)
5. Deletes snapshots older than 28 days
6. Restarts the server (if it was running before)

Runs in parallel across all configured servers. If anything goes wrong, the server is always restarted via an EXIT trap.

## Setup

### 1. Add GitHub Secrets

Go to your repository **Settings > Secrets and variables > Actions > Secrets** and add:

| Secret | Description |
|--------|-------------|
| `SCP_USER` | Your netcup SCP login username |
| `SCP_PASS` | Your netcup SCP login password |

These are the same credentials you use to log in at [servercontrolpanel.de](https://www.servercontrolpanel.de). No API key is needed.

### 2. Add Server IDs

Go to **Settings > Secrets and variables > Actions > Variables** and create:

| Variable | Value | Description |
|----------|-------|-------------|
| `SERVER_IDS` | `["652141", "826718"]` | JSON array of your netcup server IDs |

The value must be a valid JSON array of strings. Examples:

```
Single server:   ["652141"]
Multiple:        ["652141", "826718", "829107"]
```

You can find your server IDs in the SCP URL when viewing a server (e.g. `servercontrolpanel.de/scp/servers/652141`), or via the API:

```bash
curl -s "$API_BASE/servers" -H "Authorization: Bearer $TOKEN" | jq '.[].serverId'
```

### 3. (Optional) Change the Schedule

The schedule is configured as a cron expression on line 4 of the workflow file:

```yaml
schedule:
  - cron: '0 4 * * 1'
```

| Cron | Schedule |
|------|----------|
| `0 4 * * 1` | Every Monday at 04:00 UTC **(default)** |
| `0 3 * * 0` | Every Sunday at 03:00 UTC |
| `0 2 * * 1,4` | Monday and Thursday at 02:00 UTC |
| `0 5 1,15 * *` | 1st and 15th of each month at 05:00 UTC |
| `0 4 * * *` | Every day at 04:00 UTC |

The workflow can also be triggered manually from the GitHub Actions tab at any time.

### 4. (Optional) Adjust Settings

In the workflow's Configuration section:

| Constant | Default | Description |
|----------|---------|-------------|
| `POLL_INTERVAL` | `5` | Seconds between status polls |
| `TASK_TIMEOUT_ATTEMPTS` | `30` | Max polls for snapshot task (30 x 5s = 150s) |
| `SNAPSHOT_RETENTION_DAYS` | `28` | Delete snapshots older than this |

## How it works

### Authentication

Uses OAuth2 Resource Owner Password Credentials grant (`grant_type=password`) with `client_id=scp` against the SCP Keycloak instance. Tokens are short-lived and masked in GitHub Actions logs via `::add-mask::`.

### Snapshot creation

Creates offline (cold) snapshots to ensure filesystem consistency. The server is stopped before snapshot creation and restarted afterward. The snapshot uses disk `vda` — adjust if your server uses a different disk name.

If a snapshot with today's date already exists (e.g. from a manual trigger earlier that day), the workflow skips that server entirely without stopping it.

### Snapshot retention

After each successful snapshot, snapshots older than `SNAPSHOT_RETENTION_DAYS` are automatically deleted. The cleanup:

- Parses snapshot names as dates (supports `YYYY-MM-DD` and legacy `ddmmyy` formats)
- Skips snapshots with unrecognized names (never deletes what it can't parse)
- Treats failures as warnings — cleanup issues never prevent server restart

### Safety guarantees

- **Duplicate check**: Skips the run if today's snapshot already exists (no unnecessary downtime)
- **EXIT trap**: The server is always restarted, even if the workflow fails mid-execution
- **Token refresh**: Re-authenticates before restart and cleanup in case the token expired
- **Parallel-safe**: `fail-fast: false` ensures one server failing doesn't cancel others
- **Strict mode**: `set -euo pipefail` catches errors early

## Security

- Credentials are stored as GitHub repository secrets (never in code)
- Server IDs are stored as a GitHub repository variable (not hardcoded)
- No credentials or server IDs exist in git history
- Access tokens are masked in all GitHub Actions log output
- Auth error messages do not leak raw API responses
- The workflow has no checkout step — it never clones the repository, reducing attack surface

### Storing credentials safely

GitHub repository secrets are encrypted at rest and only exposed to workflow runs. For additional security:

- Use a **dedicated SCP sub-account** with minimal permissions if netcup supports it
- Enable **2FA** on your netcup account (note: this may require adjusting the auth flow)
- **Rotate your password** periodically — update the `SCP_PASS` secret afterward
- Consider using [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments) with protection rules for production servers

## API Reference

This workflow uses the [netcup SCP REST API](https://helpcenter.netcup.com/en/wiki/server/rest-api):

- `POST /realms/scp/protocol/openid-connect/token` — OAuth2 authentication
- `GET /servers/{id}` — Server state
- `PATCH /servers/{id}` — Start/stop server
- `GET /tasks/{uuid}` — Poll task status
- `POST /servers/{id}/snapshots` — Create snapshot
- `GET /servers/{id}/snapshots` — List snapshots
- `DELETE /servers/{id}/snapshots/{name}` — Delete snapshot

## License

MIT
