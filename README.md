# InvincibleGatewayGuard

**OpenClaw skill: keep gateway auth in sync with config.**

Detects and fixes gateway auth drift (running process vs `openclaw.json`). Writes `gateway.auth` to config **only when it's missing or wrong**, so you get a stable token and no unnecessary restarts.

## What it does

- **Check** -- `status` reports whether the gateway's auth matches `openclaw.json`.
- **Fix** -- `ensure --apply` restarts the gateway with credentials from config and (if needed) writes correct auth to config once.
- **Safe writes** -- Only updates `gateway.auth` when incorrect; never overwrites a correct config.

Use it when you see `device_token_mismatch`, "Gateway auth issue", or before delegating to sub-agents so the gateway is in a known-good state.

## Install

```bash
clawhub install invincible-gateway-guard
# or:
git clone https://github.com/RuneweaverStudios/InvincibleGatewayGuard.git
cp -r InvincibleGatewayGuard ~/.openclaw/workspace/skills/
```

## Quick start

```bash
# Check (machine-readable)
python3 scripts/gateway_guard.py status --json

# Fix if mismatch (restart gateway with config auth; write config only if wrong)
python3 scripts/gateway_guard.py ensure --apply --json
```

## Configuration

Edit `config.json` to customize timeouts and ports:

```json
{
  "gateway_port": 18789,
  "kill_timeout_sec": 5,
  "restart_wait_sec": 2.0,
  "command_timeout_sec": 8
}
```

## Requirements

- OpenClaw `openclaw.json` with `gateway.auth` (token or password) and `gateway.port`
- `openclaw` CLI on PATH (for `ensure --apply`)
- Python 3.6+

## Options

| Command   | Options   | Description |
|-----------|-----------|-------------|
| `status`  | `--json`  | Check consistency; exit 0 if ok, 1 if mismatch |
| `ensure`  | `--apply` | Fix by restarting gateway with config auth |
| `ensure`  | `--json`  | Emit JSON result for orchestration |

## License

MIT
