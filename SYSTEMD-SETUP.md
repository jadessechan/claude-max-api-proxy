# claude-max-api-proxy — Systemd Auto-Start Setup

## Overview

The proxy runs as a **systemd system service** under a dedicated non-root user (`openclaw`). It starts automatically on boot, restarts on crash, and survives logouts without needing `loginctl enable-linger`.

## Why a dedicated non-root user?

The proxy's subprocess manager intentionally skips `--dangerously-skip-permissions` when running as root, because the Claude CLI rejects that flag under UID 0 (see [`src/subprocess/manager.ts`](src/subprocess/manager.ts)). Without that flag, Claude tries to prompt for permission on every tool call — but the subprocess has piped stdio with no TTY, so prompts vanish silently and calls hang.

Running as a non-root user (`openclaw` here, but any unprivileged account works) lets the flag pass through. No prompts, no silent hangs. The user's filesystem permissions become your safety boundary instead.

## Service Details

| Property         | Value                                             |
|------------------|---------------------------------------------------|
| Service name     | `claude-proxy.service`                            |
| Service file     | `/etc/systemd/system/claude-proxy.service`        |
| Runs as          | `openclaw:openclaw` (system user)                 |
| Port             | 3456                                              |
| Endpoint         | `http://127.0.0.1:3456/v1/chat/completions`       |
| Restart policy   | Always, 5-second delay                            |
| Linger           | Not needed (system service)                       |

## One-time setup

### 1. Create a dedicated system user

```bash
sudo useradd --system --create-home --home-dir /home/openclaw --shell /bin/bash openclaw
```

### 2. Place the proxy and the Claude CLI where the service user can reach them

Root's home is typically `0700`, so an unprivileged service user can't traverse into `/root/.local/bin/claude`. Stage both the proxy and the CLI under `/opt/`:

```bash
# Proxy
sudo cp -a /path/to/claude-max-api-proxy /opt/claude-max-api-proxy
sudo chown -R openclaw:openclaw /opt/claude-max-api-proxy

# Claude CLI (copy the versioned install so `openclaw` can exec it)
sudo cp -a ~/.local/share/claude /opt/claude
sudo chown -R openclaw:openclaw /opt/claude
sudo mkdir -p /opt/claude/bin
sudo ln -sf /opt/claude/versions/<version> /opt/claude/bin/claude
sudo chown -h openclaw:openclaw /opt/claude/bin/claude
```

### 3. Give the service user its OAuth credentials

The Claude CLI expects `~/.claude/.credentials.json`. Copy yours into the service user's home (or run `claude` interactively as that user to OAuth fresh):

```bash
sudo cp -a ~/.claude /home/openclaw/.claude
sudo chown -R openclaw:openclaw /home/openclaw/.claude
sudo chmod 600 /home/openclaw/.claude/.credentials.json
```

### 4. Write the service unit

`/etc/systemd/system/claude-proxy.service`:

```ini
[Unit]
Description=Claude Max API Proxy (OpenAI-compatible)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=openclaw
Group=openclaw
ExecStart=/usr/local/bin/node /opt/claude-max-api-proxy/dist/server/standalone.js 3456
Restart=always
RestartSec=5
TimeoutStopSec=15
Environment=HOME=/home/openclaw
Environment=PATH=/opt/claude/bin:/usr/local/bin:/usr/bin:/bin
Environment=CLAUDE_BIN=/opt/claude/bin/claude
WorkingDirectory=/opt/claude-max-api-proxy

[Install]
WantedBy=multi-user.target
```

Key lines:
- `User=openclaw` / `Group=openclaw` — the whole point of this setup.
- `CLAUDE_BIN=/opt/claude/bin/claude` — the proxy reads this env var and won't fall back to `PATH` lookups.
- `WantedBy=multi-user.target` — system-level target, not `default.target` (which is user-scoped).

### 5. Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now claude-proxy.service
```

### 6. Verify

```bash
systemctl status claude-proxy.service
ss -tlnp | grep 3456                                  # bound as 'openclaw'
curl -s http://127.0.0.1:3456/v1/models | head -c 200 # proxy responds
```

## Common Commands

```bash
# Status
sudo systemctl status claude-proxy

# Logs (subprocess stderr also surfaces here — key visibility tool)
sudo journalctl -u claude-proxy -f

# Restart
sudo systemctl restart claude-proxy

# Stop / disable
sudo systemctl stop claude-proxy
sudo systemctl disable claude-proxy
```

## Troubleshooting

- **Port conflict** — `ss -tlnp | grep 3456` to find the process; stop it before starting the service.
- **`claude: command not found`** — the service user can't traverse into the path holding the CLI. Confirm `CLAUDE_BIN` points at a file the service user can execute (`sudo -u openclaw test -x $CLAUDE_BIN && echo OK`).
- **`Authentication: FAILED` at startup** — run `sudo -u openclaw claude` to re-OAuth as the service user, then restart the service. Or copy a fresh `.credentials.json` from a known-good account.
- **Still hitting silent prompts** — confirm the service is actually running as a non-root user (`ps -o user= -p $(pidof -s node)`). If it's still root, `--dangerously-skip-permissions` is being skipped and every tool call will stall.
- **Telegram / OpenClaw bot returns nothing** — subprocess likely crashed silently. `journalctl -u claude-proxy -f` shows the actual error (file not found, EACCES on a path the service user can't reach, etc.).

## Migrating from an older `--user` service running as root

If you already had the proxy running via `systemctl --user` under root:

```bash
# Stop and disable the old --user service
systemctl --user disable --now claude-proxy.service

# Do the one-time setup above (create user, stage /opt, write system unit)

# Start the new system service
sudo systemctl enable --now claude-proxy.service

# Keep the old config around for 48h as rollback; remove once stable
```
