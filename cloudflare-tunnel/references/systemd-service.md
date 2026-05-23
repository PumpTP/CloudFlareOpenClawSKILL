# Run `cloudflared` as a systemd service

For any long-lived setup. Survives reboots automatically (no linger needed — it's a system service, not user).

## Install the service

`cloudflared` ships a built-in installer that generates the systemd unit:

```bash
sudo cloudflared service install
```

This:
- Creates `/etc/systemd/system/cloudflared.service`
- Uses `CONFIG_YML` and the credentials file referenced there
- Enables the service to start at boot

## Start and check

```bash
sudo systemctl start CLOUDFLARE_SERVICE
sudo systemctl enable CLOUDFLARE_SERVICE
sudo systemctl status CLOUDFLARE_SERVICE --no-pager
```

Expected: active (running), recent log entries showing "Registered tunnel connection".

## Reboot survival

System services auto-start on boot once enabled — no `loginctl enable-linger` needed (that's only for user services).

Verify enablement:

```bash
sudo systemctl is-enabled CLOUDFLARE_SERVICE
```

Expected: `enabled`.

Test by rebooting:

```bash
sudo reboot
```

After reconnecting:

```bash
sudo systemctl status CLOUDFLARE_SERVICE --no-pager
curl -I https://PUBLIC_HOSTNAME
```

If the service didn't come back:

```bash
sudo systemctl enable --now CLOUDFLARE_SERVICE
```

## Restart after config changes

```bash
sudo systemctl restart CLOUDFLARE_SERVICE
sudo systemctl status CLOUDFLARE_SERVICE --no-pager
```

Always validate `CONFIG_YML` first:

```bash
sudo cloudflared tunnel ingress validate
```

A broken config can leave the service in a crash loop.

## Inspect logs

```bash
# Last 200 lines
sudo journalctl -u CLOUDFLARE_SERVICE -n 200 --no-pager

# Follow live
sudo journalctl -u CLOUDFLARE_SERVICE -f

# Time-bounded
sudo journalctl -u CLOUDFLARE_SERVICE --since "10 min ago"

# Errors only
sudo journalctl -u CLOUDFLARE_SERVICE --since today \
  | grep -iE "error|fail|disconnect|refused|timeout"
```

Look for:
- `Registered tunnel connection` — healthy
- `Initiating graceful shutdown` — restarting or stopping
- `Unable to connect to the origin` — local service down (not a tunnel problem)
- `connection lost` — temporary; `cloudflared` reconnects automatically

## Uninstall the service

If you need to remove it:

```bash
sudo cloudflared service uninstall
```

This stops the service and removes the systemd unit. The tunnel itself and its credentials are unaffected — you can still run it manually with `cloudflared tunnel run TUNNEL_NAME`.

## Reinstall after `cloudflared` upgrade

After upgrading the `cloudflared` package, reinstall the service to pick up any unit-file changes:

```bash
sudo cloudflared service uninstall
sudo cloudflared service install
sudo systemctl restart CLOUDFLARE_SERVICE
```

