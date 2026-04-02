# netcup Snapshot Automation

Automated weekly offline snapshots for [netcup](https://www.netcup.de/) vServers using the SCP REST API and GitHub Actions.

## What it does

Every Monday at 04:00 UTC, the workflow:

1. Authenticates with the [Server Control Panel (SCP)](https://www.servercontrolpanel.de) REST API
2. Stops each server (if running)
3. Creates an offline snapshot (named by date, e.g. `2026-04-02`)
4. Deletes snapshots older than 28 days
5. Restarts the server (if it was running before)

Runs in parallel across all configured servers. If anything goes wrong, the server is always restarted via an EXIT trap.

## Setup

### 1. Add GitHub Secrets

Go to your repository **Settings > Secrets and variables > Actions** and add:

| Secret | Description |
|--------|-------------|
| `SCP_USER` | Your netcup SCP login username |
| `SCP_PASS` | Your netcup SCP login password |

These are the same credentials you use to log in at [servercontrolpanel.de](https://www.servercontrolpanel.de). No API key is needed.

### 2. Configure Server IDs

Edit the `server_id` matrix in `.github/workflows/weekly-snapshot.yml`:

```yaml
matrix:
  server_id: ["652141", "826718", "829107"]
```

Replace with your own server IDs. You can find them in the SCP URL when viewing a server, or via the API:

```bash
curl -s "$API_BASE/servers" -H "Authorization: Bearer $TOKEN" | jq '.[].serverId'
```

### 3. (Optional) Adjust Settings

In the workflow's Configuration section:

| Constant | Default | Description |
|----------|---------|-------------|
| `POLL_INTERVAL` | `5` | Seconds between status polls |
| `TASK_TIMEOUT_ATTEMPTS` | `30` | Max polls for snapshot task (30 × 5s = 150s) |
| `SNAPSHOT_RETENTION_DAYS` | `28` | Delete snapshots older than this |

## How it works

### Authentication

Uses OAuth2 Resource Owner Password Credentials grant (`grant_type=password`) with `client_id=scp` against the SCP Keycloak instance. Tokens are short-lived and masked in GitHub Actions logs via `::add-mask::`.

### Snapshot creation

Creates offline (cold) snapshots to ensure filesystem consistency. The server is stopped before snapshot creation and restarted afterward. The snapshot uses disk `vda` — adjust if your server uses a different disk name.

### Snapshot retention

After each successful snapshot, snapshots older than `SNAPSHOT_RETENTION_DAYS` are automatically deleted. The cleanup:

- Parses snapshot names as dates (supports `YYYY-MM-DD` and legacy `ddmmyy` formats)
- Skips snapshots with unrecognized names (never deletes what it can't parse)
- Treats failures as warnings — cleanup issues never prevent server restart

### Safety guarantees

- **EXIT trap**: The server is always restarted, even if the workflow fails mid-execution
- **Token refresh**: Re-authenticates before restart and cleanup in case the token expired
- **Parallel-safe**: `fail-fast: false` ensures one server failing doesn't cancel others
- **Strict mode**: `set -euo pipefail` catches errors early

## Security

- Credentials are stored as GitHub repository secrets (never in code)
- No credentials exist in git history
- Access tokens are masked in all GitHub Actions log output
- Auth error messages do not leak raw API responses
- The workflow has no checkout step — it never clones the repository, reducing attack surface

### Storing credentials safely

GitHub repository secrets are encrypted at rest and only exposed to workflow runs. For additional security:

- Use a **dedicated SCP sub-account** with minimal permissions if netcup supports it
- Enable **2FA** on your netcup account (note: this may require adjusting the auth flow)
- **Rotate your password** periodically — update the `SCP_PASS` secret afterward
- Consider using [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments) with protection rules for production servers

## Manual trigger

You can trigger the workflow manually from the GitHub Actions tab (uses `workflow_dispatch`). Note: if a snapshot with today's date already exists, the API will reject the duplicate.

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
