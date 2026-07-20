# netcup Snapshot Automation

Weekly offline snapshots for [netcup](https://www.netcup.de/) vServers via the [SCP REST API](https://www.netcup.com/en/helpcenter/documentation/server/rest-api) and GitHub Actions.

## How it works

Every Monday 04:00 UTC, per server in `SERVER_IDS`, in parallel:

1. Skip if today’s `YYYYMMDD` snapshot exists — no downtime.
1. Resolve disk (first only; warns on multi-disk), read power state.
1. Gracefully stop (ACPI `OFF`) if running — offline snapshots need `SHUTOFF`.
1. Prune our dated snapshots to the newest `SNAPSHOT_KEEP − 1` (the new one makes `KEEP`).
1. Dry-run: **possible** → go; **out of space** → run storage optimization if enabled (**deletes all snapshots**; data/OS safe), else fail; **blocked** → fail; **inconclusive** → try anyway.
1. Create snapshot `YYYYMMDD`.
1. Restart — only if it was running at the start.

An `EXIT` trap (with `TERM`/`INT` routed in) always attempts a restart, even on failure/cancel/timeout. Per-server `concurrency` prevents overlapping runs. State-changing writes retry through netcup’s transient **409/503** locks until they serialize or `LOCK_RETRY_DEADLINE`.

## Setup

**Secret** (`Settings → Secrets and variables → Actions`): `SCP_REFRESH_TOKEN` — an offline refresh token, the auth flow in netcup's [REST API docs](https://www.netcup.com/en/helpcenter/documentation/server/rest-api). The workflow exchanges it for a short-lived bearer token (`client_id=scp`, `grant_type=refresh_token`) on every run. Generate it once:

1. Request a device code:

   ```bash
   curl -X POST 'https://www.servercontrolpanel.de/realms/scp/protocol/openid-connect/auth/device' \
     -d 'client_id=scp' -d 'scope=offline_access openid'
   ```

1. Open the returned `verification_uri_complete` in a browser, log in to SCP and approve the grant.
1. Exchange the `device_code` for tokens (within `expires_in`, 10 min):

   ```bash
   curl -X POST 'https://www.servercontrolpanel.de/realms/scp/protocol/openid-connect/token' \
     -d 'grant_type=urn:ietf:params:oauth:grant-type:device_code' \
     -d 'device_code=<device-code>' -d 'client_id=scp'
   ```

1. Store the `refresh_token` value from the response as the `SCP_REFRESH_TOKEN` secret.

The offline token is reusable and never expires as long as it's used at least once every 30 days — the weekly schedule keeps it alive. If the workflow is disabled for longer, regenerate the token. Revoke a leaked token via the `…/openid-connect/revoke` endpoint or the SCP Account Console (Applications → scp → Remove access).

**Variable** — one required: `SERVER_IDS`, a JSON array of strings like `["123456", "789012"]` (ID is in the SCP URL).

**Schedule** (optional): edit the cron, default `0 4 * * 1`. Also runs manually from the Actions tab.

## Overrides (optional)

Create these repo Variables only to override the defaults already in the workflow:

|Variable                     |Default|Effect                                     |
|-----------------------------|-------|-------------------------------------------|
|`SNAPSHOT_KEEP`              |`3`    |Dated snapshots to retain (count-based)    |
|`ENABLE_STORAGE_OPTIMIZATION`|`true` |Allow the destructive out-of-space fallback|

Everything else is a `readonly` constant in the workflow: `POLL_INTERVAL` 5 s, `POLL_ATTEMPTS` 12 (~60 s), `SNAPSHOT_POLL_ATTEMPTS` 24 (~120 s), `OPT_POLL_ATTEMPTS` 240 (~20 min ceiling), `RESTART_MAX_ATTEMPTS` 3, `LOCK_RETRY_INTERVAL` 10 s, `LOCK_RETRY_DEADLINE` 180 s, `TOKEN_REFRESH_SECS` 240 s. Worst case ≈ the 20 min optimization ceiling → `timeout-minutes: 30`; raise both together.

## Safety

- **Always restarts**, even on failure/cancel/timeout; a failed restart fails the job (red, not silent green).
- **No needless downtime** — skips if today’s snapshot exists; never powers on a server that was off.
- **Cleanup keeps the newest `SNAPSHOT_KEEP`** dated (`YYYYMMDD`/`YYYY-MM-DD`) snapshots by `creationTime`; manual snapshots untouched.
- **Destructive fallback is opt-out** — optimization wipes all snapshots, only on out-of-space + `ENABLE_STORAGE_OPTIMIZATION≠false`.
- **Hardened** — encrypted secrets, masked tokens, URL-encoded auth, minimal logging, `permissions: {}`, no checkout. Only idempotent `GET`s retry at the transport layer. Auth failures log Keycloak's HTTP status and error text, never credentials.

## License

MIT
