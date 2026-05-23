---
name: openclaw-gateway-litellm
description: Use for OpenClaw gateway tasks on Ubuntu/Linux, openclaw.json, LiteLLM or OpenAI-compatible provider config, internal LiteLLM, model selection, default model changes, API key rotation, openclaw-gateway systemd service, gateway logs, local 127.0.0.1:18789 checks, port conflicts, JSON errors, and model/auth failures. For cloudflared, DNS, ingress, 502/530, or public hostname tunnel routing, use cloudflare-tunnel too.
---

# OpenClaw Gateway with LiteLLM

Purpose: run OpenClaw locally and connect it to a LiteLLM/OpenAI-compatible provider. This skill owns the local gateway and model provider layer. Cloudflare Tunnel owns the public hostname layer.

## Defaults

| Placeholder | Default |
|---|---|
| `<OS>` | Ubuntu 22.04 VM |
| `PUBLIC_HOSTNAME` | `app.example.com` |
| `<GATEWAY_PORT>` | `18789` |
| `<CONFIG_PATH>` | `~/.openclaw/openclaw.json` |
| `LITELLM_BASE_URL` | `https://gpt.example.co.th/litellm/v1` |
| `<MODEL_ID>` | `litellm/gpt-4o-mini` |
| `<PROVIDER_KEY>` | `litellm` |
| `MODEL_NAME` | `gpt-4o-mini` |

Never print `LITELLM_API_KEY`. Read it from config or prompt securely when needed.

## Routing logic

Before opening references, classify the task:

| User needs | Open |
|---|---|
| First OpenClaw provider setup or full config structure | `references/openclaw-config.md` |
| Add/change/list models, change default model, fallback model | `references/model-management.md` |
| Rotate expired/leaked LiteLLM API key | `references/api-key-rotation.md` |
| Restart, status, logs, switch direct/systemd, reboot survival — service responds normally | `references/service-management.md` |
| Hard kill — gateway stuck, port held, normal restart failed, `openclaw gateway stop` not stopping | `references/hard-restart.md` |
| WhatsApp plugin install, verify, audit warning, mitigation | `references/whatsapp-plugin.md` |
| Fix local gateway down, auth, JSON, port conflicts | `references/troubleshooting.md` |
| Public URL, Cloudflare DNS, tunnel, 502/530 | Use `cloudflare-tunnel` |

Load only the smallest matching reference. Do not read all references for basic restart/status questions.

## Fast diagnosis

```bash
openclaw gateway status
curl -I http://127.0.0.1:<GATEWAY_PORT>
curl -I https://PUBLIC_HOSTNAME
```

| Local curl | Public curl | Likely layer |
|---|---|---|
| works | works | healthy; test model next |
| works | fails | Cloudflare Tunnel layer |
| fails | fails | OpenClaw local gateway layer |
| fails | works | inconsistent test; rerun checks |

## Common commands

```bash
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
journalctl --user -u openclaw-gateway.service -n 100 --no-pager
python3 -m json.tool <CONFIG_PATH>
openclaw models status
openclaw agent --agent main -m "Health check. What model are you using?"
```

## Public exposure security rule

If exposing OpenClaw publicly, the tunnel must require auth. See `cloudflare-tunnel` skill → `references/cloudflare-access.md`.

## Safety

Back up config before edits. Validate JSON before restarting. Set `chmod 600 <CONFIG_PATH>`. Keep gateway bound to `127.0.0.1`. Never put API keys in shell history, logs, chat, or commits.



