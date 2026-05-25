# CloudFlareOpenClawSKILL

Two paired Claude skills. Run OpenClaw locally. Expose via Cloudflare Tunnel.

## Skills

### `openclaw-gateway-litellm`

Owns local gateway layer. Covers `openclaw.json`, LiteLLM provider, model selection, key rotation, systemd service, logs, port conflicts, JSON errors.

### `cloudflare-tunnel`

Owns public hostname layer. Covers `cloudflared` install, DNS, `config.yml` ingress, systemd, 502/530 fixes, Cloudflare Access.

## Layer split

| Layer | Skill |
|---|---|
| Local `127.0.0.1:18789` | `openclaw-gateway-litellm` |
| Public `https://host` | `cloudflare-tunnel` |

Use both together. Diagnose local first. Then tunnel.

## Fast diagnosis

```bash
curl -I http://127.0.0.1:18789
curl -I https://PUBLIC_HOSTNAME
```

| Local | Public | Layer |
|---|---|---|
| ok | ok | healthy |
| ok | fail | tunnel |
| fail | fail | gateway |

## Layout

```
cloudflare-tunnel/
  SKILL.md
  references/
    install-and-auth.md
    config-and-routing.md
    systemd-service.md
    troubleshooting.md
    cloudflare-access.md

openclaw-gateway-litellm/
  SKILL.md
  references/
    openclaw-config.md
    model-management.md
    api-key-rotation.md
    service-management.md
    hard-restart.md
    troubleshooting.md
    whatsapp-plugin.md
```

## Install

Drop folders into Claude skills directory:

```bash
cp -r cloudflare-tunnel openclaw-gateway-litellm ~/.claude/skills/
```

Claude loads `SKILL.md` automatically. References load on demand.

## Safety

Never print API keys. Bind gateway to `127.0.0.1`. Keep DNS proxied. Require Cloudflare Access on public hostname. `chmod 600` on configs and credentials.
