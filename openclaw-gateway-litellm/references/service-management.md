# OpenClaw service management

Everything about running OpenClaw as a service: systemd vs direct process, log inspection, reboot survival.

## Restart cleanly

```bash
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

If port `<GATEWAY_PORT>` is stuck after a stop:

```bash
systemctl --user stop openclaw-gateway.service
ss -ltnp | grep <GATEWAY_PORT>
```

Kill any conflicting process by PID, then start again.

## Switch from direct process to systemd

Recommended for any long-lived deployment. Survives reboots if linger is enabled.

```bash
# Find any direct openclaw process
pgrep -af openclaw

# Stop it cleanly using the PID from above
kill <PID_FROM_PGREP>

# Only if kill above doesn't terminate it:
# pkill -f openclaw-gateway

# Start as a user service
systemctl --user enable --now openclaw-gateway.service

# Allow user service to run without active login
loginctl enable-linger $USER
```

`loginctl enable-linger` is the key piece — without it, user services stop at logout.

## Switch from systemd to direct process

Only for debugging — direct process won't survive a reboot or logout.

```bash
systemctl --user disable --now openclaw-gateway.service
openclaw gateway start --foreground
```

Verify only one instance is bound to the port:

```bash
ss -ltnp | grep <GATEWAY_PORT>
```

Two processes on the same port → undefined behavior. Kill one.

## Inspect logs

### OpenClaw gateway (user systemd service)

```bash
# Last 200 lines
journalctl --user -u openclaw-gateway.service -n 200 --no-pager

# Follow live
journalctl --user -u openclaw-gateway.service -f

# Time-bounded
journalctl --user -u openclaw-gateway.service --since "10 min ago"
journalctl --user -u openclaw-gateway.service --since "2026-05-13 10:00"

# Errors only
journalctl --user -u openclaw-gateway.service --since today \
  | grep -iE "error|fail|denied|timeout|refused"
```

### Cloudflare Tunnel logs

Owned by the `cloudflare-tunnel` skill. Reach for it when local OpenClaw checks pass but public URL doesn't.

## Reboot survival test

Confirms the full stack auto-recovers after a VM restart. Run this whenever you change service configuration or after a fresh install.

### Pre-reboot: confirm enablement

```bash
systemctl --user is-enabled openclaw-gateway.service
sudo systemctl is-enabled CLOUDFLARE_SERVICE
loginctl show-user $USER | grep Linger
```

Expected output: `enabled`, `enabled`, `Linger=yes`.

Any other result → fix before rebooting (see "If services don't return" below).

### Reboot

```bash
sudo reboot
```

### Post-reboot: verify

After reconnecting via SSH:

```bash
openclaw gateway status
systemctl --user status openclaw-gateway.service --no-pager
sudo systemctl status CLOUDFLARE_SERVICE --no-pager
curl -I http://127.0.0.1:<GATEWAY_PORT>
curl -I https://PUBLIC_HOSTNAME
```

Expected:
- OpenClaw gateway running
- `CLOUDFLARE_SERVICE` active
- Local gateway returns success
- Public hostname returns `HTTP/2 200`
- Dashboard opens at `https://PUBLIC_HOSTNAME`

### If OpenClaw doesn't return after reboot

```bash
loginctl enable-linger $USER
systemctl --user enable --now openclaw-gateway.service
```

`enable-linger` is required for user services to start without a login session — this is the most common reason a user service "disappears" after reboot.

### If Cloudflare Tunnel doesn't return

```bash
sudo systemctl enable --now CLOUDFLARE_SERVICE
```

For deeper Cloudflare issues, switch to the `cloudflare-tunnel` skill.

## Common service issues

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Service stops at logout | Linger not enabled | `loginctl enable-linger $USER` |
| Two processes on port | Direct process + systemd both running | Kill direct, keep systemd |
| Service starts then immediately dies | Bad JSON config | `python3 -m json.tool <CONFIG_PATH>`, fix, restart |
| Service won't start, port in use | Another process holds the port | `ss -ltnp \| grep <GATEWAY_PORT>`, kill or change port |
| Service active but unreachable | Bound to wrong interface | Check config — must bind `127.0.0.1`, not `0.0.0.0` |

