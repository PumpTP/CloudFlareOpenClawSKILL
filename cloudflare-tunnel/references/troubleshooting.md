# Cloudflare Tunnel troubleshooting

The four most common failure modes. Each section shows symptoms, diagnostics, and likely causes.

## 502 Bad Gateway

Symptom: `curl -I https://PUBLIC_HOSTNAME` returns `HTTP/2 502`.

Meaning: the tunnel reached your VM, but `cloudflared` couldn't reach the local service.

Diagnose:

```bash
# Is the local service running?
curl -I http://127.0.0.1:LOCAL_PORT

# Is cloudflared running?
sudo systemctl status CLOUDFLARE_SERVICE --no-pager

# Recent tunnel logs
sudo journalctl -u CLOUDFLARE_SERVICE -n 50 --no-pager
```

Likely causes:

- **Local service is down or on a different port** — the most common cause. Fix the upstream service. For OpenClaw, switch to the `openclaw-gateway-litellm` skill.
- **`config.yml` points at the wrong port** — `service: http://127.0.0.1:LOCAL_PORT` doesn't match where the service actually listens. Run `ss -ltnp` to see the real port, fix `config.yml`, restart `cloudflared`.
- **Service binds to a different interface** — service on `0.0.0.0` but config says `127.0.0.1` should still work; service only on a public interface won't. Use `127.0.0.1` everywhere.
- **`http` vs `https` mismatch** — local service requires HTTPS but config says `http://`. Either switch the service to plain HTTP (preferred for `127.0.0.1`) or update `config.yml` to `https://` with `originRequest: { noTLSVerify: true }` if self-signed.

## 530 error

Symptom: `curl -I https://PUBLIC_HOSTNAME` returns `530`.

Meaning: Cloudflare resolved DNS, but the tunnel itself isn't reachable — the tunnel is offline, misnamed, or DNS points at the wrong tunnel.

Diagnose:

```bash
# Is cloudflared running?
sudo systemctl status CLOUDFLARE_SERVICE --no-pager

# What tunnels exist?
cloudflared tunnel list

# What does DNS resolve to?
dig +short PUBLIC_HOSTNAME
```

Likely causes:

- **`cloudflared` service stopped** — restart it: `sudo systemctl restart CLOUDFLARE_SERVICE`
- **Tunnel UUID in `config.yml` doesn't match an existing tunnel** — compare `tunnel:` line with `cloudflared tunnel list`. Fix or recreate.
- **DNS CNAME points at the wrong tunnel UUID** — old tunnel was deleted but DNS still points at it. In Cloudflare dashboard, update the CNAME to the correct `TUNNEL_UUID.cfargotunnel.com`.
- **Credentials file missing or unreadable** — check `ls -l CREDENTIALS_FILE` and journal logs for "credentials file" errors.

## DNS not found / hostname doesn't resolve

Symptom: `curl https://PUBLIC_HOSTNAME` returns "Could not resolve host", or browser shows "DNS_PROBE_FINISHED_NXDOMAIN".

Diagnose:

```bash
dig +short PUBLIC_HOSTNAME
```

Likely causes:

- **No DNS record exists** — create it: `cloudflared tunnel route dns TUNNEL_NAME PUBLIC_HOSTNAME`, or add manually in dashboard (see `config-and-routing.md`).
- **DNS record exists but is "DNS only" (grey cloud)** — must be Proxied (orange cloud). Toggle in dashboard.
- **Domain not on Cloudflare** — `CLOUDFLARE_DOMAIN` must have Cloudflare as the authoritative nameserver. Check the Overview tab in dashboard. If nameservers are still pointed at your old registrar, DNS won't resolve through Cloudflare.
- **DNS propagation delay** — usually under a minute, but can take longer. Wait, then retest.

## Tunnel disconnected / connection lost

Symptom: `journalctl` shows repeated `connection lost`, `unable to connect to edge`, or `tunnel disconnected`.

Diagnose:

```bash
sudo journalctl -u CLOUDFLARE_SERVICE --since "30 min ago" \
  | grep -iE "disconnect|connection lost|unable"
```

Likely causes:

- **Brief disconnects are normal** — `cloudflared` maintains 4 connections to Cloudflare's edge and rotates them. Logs show disconnect/reconnect cycles. As long as at least one connection is live, traffic flows. Only worry if logs show all connections down for sustained periods.
- **VM lost outbound internet** — test: `curl -I https://www.cloudflare.com`. If that fails, fix VM networking first.
- **Outbound port blocked** — `cloudflared` uses port 7844 outbound (QUIC) and 443 (HTTPS fallback). Corporate firewalls sometimes block QUIC. Force HTTP/2 mode in `config.yml`:

  ```yaml
  protocol: http2
  ```

- **Credentials revoked or expired** — rare, but possible if the tunnel was deleted in the dashboard while the service runs locally. Logs will say "credentials invalid". Recreate the tunnel.
- **Cloudflare incident** — check https://www.cloudflarestatus.com/ for ongoing issues.

## Slow / intermittent / 524

`524 A timeout occurred` means the local service took longer than 100 seconds to respond. Either the upstream is slow (fix the upstream), or you need to raise the Cloudflare timeout (Enterprise plan feature). For long-running operations, switch to streaming responses.

## Quick health check after any fix

```bash
sudo systemctl status CLOUDFLARE_SERVICE --no-pager
sudo journalctl -u CLOUDFLARE_SERVICE -n 30 --no-pager
curl -I https://PUBLIC_HOSTNAME
```

All three should look healthy. If the public URL still fails but the local service is fine, the problem is somewhere in the Cloudflare layer — recheck DNS proxy status, tunnel UUID, and ingress rules.

