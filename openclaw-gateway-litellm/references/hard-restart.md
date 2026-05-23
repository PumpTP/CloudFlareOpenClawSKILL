# OpenClaw hard restart reference

Use this when normal OpenClaw restart commands do not fully stop the gateway.

## When to use this

Use this when:

- `openclaw gateway stop` does not fully stop OpenClaw.
- OpenClaw still appears to be running after stop.
- Port `18789` or `18791` is still occupied.
- Cloudflare Tunnel still reaches an old or stuck OpenClaw process.
- The gateway behaves incorrectly after manual start/stop attempts.

## Full hard restart command

```bash
openclaw gateway stop 2>/dev/null
systemctl --user stop openclaw-gateway.service 2>/dev/null
pkill -f openclaw
sleep 2
ss -ltnp | grep -E "18789|18791" || echo "OpenClaw ports are clear"
openclaw gateway run
```

## What this does

1. Tries to stop OpenClaw using the normal CLI command.
2. Stops the systemd user service if it exists.
3. Kills remaining OpenClaw processes.
4. Waits briefly for ports to clear.
5. Checks whether ports `18789` or `18791` are still listening.
6. Starts OpenClaw manually again.

## Important note

This starts OpenClaw manually with:

```bash
openclaw gateway run
```

If the setup should survive reboot, use the systemd user service instead:

```bash
systemctl --user start openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

