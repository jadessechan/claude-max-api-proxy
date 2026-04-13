# claude-max-api-proxy — Systemd Auto-Start Setup

## Overview

The proxy runs as a systemd user service that starts automatically on boot and restarts on crash. No manual intervention is needed to keep it running.

## Service Details

| Property       | Value                                              |
|----------------|----------------------------------------------------|
| Service name   | `claude-proxy.service`                             |
| Service file   | `~/.config/systemd/user/claude-proxy.service`      |
| Port           | 3456                                               |
| Endpoint       | `http://127.0.0.1:3456/v1/chat/completions`        |
| Restart policy | Always, with 5-second delay between restarts       |
| Linger         | Enabled (service runs even when not logged in)      |

## Service Configuration

```ini
[Unit]
Description=Claude Max API Proxy (OpenAI-compatible)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/node /root/claude-max-api-proxy/dist/server/standalone.js 3456
Restart=always
RestartSec=5
TimeoutStopSec=15
Environment=HOME=/root
Environment=PATH=/root/.local/bin:/usr/local/bin:/usr/bin:/bin
WorkingDirectory=/root/claude-max-api-proxy

[Install]
WantedBy=default.target
```

## Changes Made (2026-04-13)

1. **Fixed PATH** — Added `/root/.local/bin` to the service `PATH` so it can find the `claude` CLI, which is required at startup for authentication.
2. **Enabled the service** — Ran `systemctl --user enable claude-proxy` to start it automatically on boot.
3. **Killed orphan process** — A manually-started instance was occupying port 3456; it was stopped so systemd could take over management.

## Common Commands

```bash
# Check status
systemctl --user status claude-proxy

# View logs
journalctl --user -u claude-proxy -f

# Restart
systemctl --user restart claude-proxy

# Stop (temporarily)
systemctl --user stop claude-proxy

# Disable auto-start
systemctl --user disable claude-proxy
```

## Troubleshooting

- **Port conflict** — If the service fails with "port already in use", find the conflicting process with `ss -tlnp | grep 3456` and kill it, then restart the service.
- **CLI not found** — Ensure the `claude` binary exists at `/root/.local/bin/claude`. If it moves, update the `PATH` in the service file and run `systemctl --user daemon-reload`.
- **Auth failure** — Run `claude` interactively to re-authenticate, then restart the service.
