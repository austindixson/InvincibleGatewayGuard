---
name: gateway-guard
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

## Requirements

- OpenClaw `openclaw.json` with `gateway.auth` (mode `token` or `password`) and `gateway.port`.
- `openclaw` CLI on PATH for `ensure --apply` (restart).
