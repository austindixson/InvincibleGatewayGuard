---
name: gateway-guard
displayName: InvincibleGatewayGuard
version: 1.0.0
description: Ensures OpenClaw gateway auth consistency. Use when checking or fixing gateway token/password mismatch, device_token_mismatch errors, or before delegating to sub-agents. Writes gateway.auth to openclaw.json only when incorrect.
---

# Gateway Guard

Keeps OpenClaw gateway authentication in sync with `openclaw.json`. Use when the user or agent sees gateway auth issues, `device_token_mismatch`, or needs to ensure the gateway is running with the correct token/password before spawning sub-agents.

## When to use

- User or logs report "Gateway auth issue", "device_token_mismatch", or "unauthorized"
- Before running the router and `sessions_spawn` (orchestrator flow): check gateway status first
- After installing or updating OpenClaw: verify gateway and config match
- When the TUI disconnects or won't connect: fix auth and restart gateway

## Commands (use absolute path for exec from any cwd)

```bash
python3 <skill-dir>/scripts/gateway_guard.py status [--json]
python3 <skill-dir>/scripts/gateway_guard.py ensure [--apply] [--json]
```

- **status** — Report whether the running gateway's auth matches `openclaw.json`. Exit 0 if ok, 1 if mismatch.
- **ensure** — Same check; if mismatch and `--apply`, restart the gateway with credentials from config. Writes `gateway.auth` to `openclaw.json` **only when it is missing or wrong** (never overwrites correct config).

## Behavior

- Reads `openclaw.json` → `gateway.auth` (token or password) and `gateway.port`.
- Compares with the process listening on that port (and optional guard state file).
- If `ensure --apply`: restarts gateway via `openclaw gateway stop` then `openclaw gateway --port N --auth token|password --token|--password SECRET`.
- If token is missing in config (token mode only): generates a token, writes it to config once, then proceeds. Does not overwrite config when it is already correct.

## JSON output (for orchestration)

- **status --json** / **ensure --json**: `ok`, `secretMatchesConfig`, `running`, `pid`, `reason`, `recommendedAction`, `configPath`, `authMode`, `gatewayPort`. When not ok, `recommendedAction` is "run gateway_guard.py ensure --apply and restart client session".

## Configuration

Edit `config.json` in the skill root to customize behavior:

```json
{
  "gateway_port": 18789,
  "kill_timeout_sec": 5,
  "auth_path": "gateway.auth",
  "restart_wait_sec": 2.0,
  "command_timeout_sec": 8
}
```

| Key | Default | Description |
|-----|---------|-------------|
| `gateway_port` | `18789` | Default gateway port (overridden by openclaw.json) |
| `kill_timeout_sec` | `5` | Seconds to wait after SIGTERM before SIGKILL |
| `auth_path` | `gateway.auth` | Config key path for auth settings |
| `restart_wait_sec` | `2.0` | Seconds to wait after restart before checking status |
| `command_timeout_sec` | `8` | Default timeout for subprocess commands |

## Guard State File Format

The guard maintains an optional state file at `logs/gateway-guard.state.json` within the skill directory. This JSON file tracks the last known gateway status, including fields such as `lastCheck` (ISO 8601 timestamp), `pid` (integer process ID of the gateway), `authMode` ("token" or "password"), `secretHash` (SHA-256 hash of the current secret for comparison without storing plaintext), and `status` ("ok" or "mismatch"). The state file is written atomically (write-to-temp then rename) to prevent corruption if the process is interrupted mid-write. If the state file is missing or unreadable, the guard falls back to a live process check via `lsof` and treats the state as unknown.

When the guard detects a mismatch between the running gateway and `openclaw.json`, it records the discrepancy in the state file along with a `reason` field describing what differs (e.g., "config token does not match running gateway token"). This allows orchestrators to read the state file for diagnostics without re-running the full check. The state file is never used as the source of truth for auth credentials; it is purely informational.

## Recovery Procedures

If the gateway enters a broken state (process running but not responding, or auth credentials out of sync), the recommended recovery sequence is: (1) run `status --json` to capture the current mismatch details, (2) run `ensure --apply` which will stop the existing gateway process (SIGTERM, then SIGKILL after `kill_timeout_sec`), update `openclaw.json` if needed, and restart the gateway with correct credentials, (3) verify recovery by running `status --json` again and confirming `ok: true`. If `ensure --apply` fails (e.g., the gateway process cannot be killed), check for zombie processes with `ps aux | grep openclaw` and manually terminate them before retrying. In environments where the gateway is managed by a LaunchAgent or systemd unit, stop the service manager unit first to prevent automatic restarts during recovery.

## Requirements

- OpenClaw `openclaw.json` with `gateway.auth` (mode `token` or `password`) and `gateway.port`.
- `openclaw` CLI on PATH for `ensure --apply` (restart).
