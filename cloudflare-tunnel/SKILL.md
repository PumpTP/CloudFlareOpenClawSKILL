---
name: cloudflare-tunnel
description: Use for Cloudflare Tunnel or cloudflared tasks on Linux, install, login, create tunnel, route DNS, edit config.yml ingress rules, run cloudflared as systemd, inspect tunnel logs, fix public hostname failures, 502, 530, DNS not found, tunnel disconnected, or exposing a local 127.0.0.1 service through a Cloudflare-proxied domain. For OpenClaw gateway or LiteLLM config problems, use openclaw-gateway-litellm instead.
---


# Cloudflare Tunnel

Purpose: expose a local service at `127.0.0.1:LOCAL_PORT` through a Cloudflare-proxied hostname without opening inbound firewall ports.

## Defaults

| Placeholder | Default |
|---|---|
| `CLOUDFLARE_DOMAIN` | `example.com` |
| `PUBLIC_HOSTNAME` | `app.example.com` |
| `LOCAL_PORT` | `18789` |
| `TUNNEL_NAME` | `example-tunnel` |
| `CLOUDFLARE_SERVICE` | `cloudflared.service` |
| `CONFIG_YML` | `/etc/cloudflared/config.yml` |
| `CREDENTIALS_FILE` | `/etc/cloudflared/TUNNEL_UUID.json` |

## Routing logic

Before opening references, classify the task:

| User needs | Open |
|---|---|
| Install `cloudflared`, authenticate, create/list tunnel | `references/install-and-auth.md` |
| Configure hostname, DNS, CNAME, ingress, `config.yml` | `references/config-and-routing.md` |
| Install/start/restart/enable service, check logs, reboot survival | `references/systemd-service.md` |
| Fix 502, 530, DNS not found, tunnel disconnected, intermittent public URL | `references/troubleshooting.md` |
| Protect OpenClaw public hostname with Cloudflare Access | `references/cloudflare-access.md` |
| Local app is down but tunnel is probably fine | Use the app's own skill, not this one |

Load only the smallest matching reference. Do not read every reference for simple restart/status questions.

## Fast diagnosis

```bash
sudo systemctl status CLOUDFLARE_SERVICE --no-pager
curl -I http://127.0.0.1:LOCAL_PORT
curl -I https://PUBLIC_HOSTNAME
```

| Local curl | Public curl | Likely layer |
|---|---|---|
| works | works | healthy |
| works | fails | Cloudflare Tunnel, DNS, or ingress |
| fails | fails | local service first, then tunnel |
| fails | works | inconsistent test; rerun checks |

## Common commands

```bash
sudo systemctl restart CLOUDFLARE_SERVICE
sudo journalctl -u CLOUDFLARE_SERVICE -n 100 --no-pager
sudo cloudflared tunnel ingress validate
cloudflared tunnel list
```

## Sensitive upstream rule

If the tunnel exposes an admin/agent surface (OpenClaw, dashboards, internal tools), Cloudflare Access is mandatory. See `references/cloudflare-access.md`.

## Safety

Never reveal tunnel tokens or credentials JSON contents. Keep credentials `chmod 600`. Keep the upstream service bound to `127.0.0.1` when possible. Keep Cloudflare DNS proxied, not DNS-only, for tunnel hostnames.

