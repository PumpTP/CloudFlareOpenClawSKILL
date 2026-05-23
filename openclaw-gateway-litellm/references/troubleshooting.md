# OpenClaw troubleshooting

Deep dives on the four most common failure modes for the gateway layer. Cloudflare Tunnel failures live in the `cloudflare-tunnel` skill.

## Local gateway fails

Symptoms: `openclaw gateway status` shows not running, `curl http://127.0.0.1:<GATEWAY_PORT>` refused or times out.

```bash
openclaw gateway status
systemctl --user status openclaw-gateway.service
journalctl --user -u openclaw-gateway.service -n 100
ss -ltnp | grep <GATEWAY_PORT>
```

Likely causes:

- **Service crashed** — check `journalctl` for the stack trace. Look for the last few lines before the crash.
- **Port already taken** — `ss -ltnp` shows another process on `<GATEWAY_PORT>`. Kill it or change the port in `<CONFIG_PATH>` (then update the Cloudflare Tunnel route to match — see the `cloudflare-tunnel` skill).
- **Invalid JSON** — service won't start if it can't parse the config. Run `python3 -m json.tool <CONFIG_PATH>` and fix any errors.
- **Missing runtime dependency** — usually visible in journal logs as `ModuleNotFoundError` or similar. Reinstall the runtime or fix `PATH`.
- **Linger off** — service stopped after logout. Run `loginctl enable-linger $USER` (see `service-management.md`).

## LiteLLM auth error

Symptoms: gateway runs, but agent requests fail with `401 Unauthorized`, `403 Forbidden`, or `Invalid API key`. Visible in gateway logs.

Test the upstream directly. Use `read -s` so the key never enters shell history:

```bash
read -s LITELLM_API_KEY
curl LITELLM_BASE_URL/models \
  -H "Authorization: Bearer ${LITELLM_API_KEY}"
unset LITELLM_API_KEY
```

If this returns a valid models list, OpenClaw is using a different (stale) key — restart the gateway. If it returns an auth error, the key itself is the problem.

Likely causes:

- **Key expired, revoked, or over budget** → rotate via `api-key-rotation.md`
- **Wrong `baseUrl`** — missing `/v1`, extra trailing slash, wrong subdomain. Compare against the LiteLLM portal's example.
- **Key lacks access to the model** — admin restricts keys to certain models. Confirm the key's model allowlist in the portal.
- **Key copied wrong** — extra whitespace, partial paste, smart-quote characters. Re-copy carefully.
- **Different account/team** — the key belongs to a different namespace than the one hosting `MODEL_NAME`.

## JSON config error

Symptoms: gateway won't restart, journal shows `JSONDecodeError` or "config parse error", or `openclaw gateway status` reports config invalid.

```bash
python3 -m json.tool <CONFIG_PATH>
```

The output tells you exactly which line and character are broken. Common mistakes:

- **Missing comma** between two fields or array items
- **Trailing comma** — JSON doesn't allow them (unlike JavaScript or Python)
- **Unescaped quote** inside a string value (use `\"` for literal quote)
- **Duplicate keys** — JSON tolerates them but OpenClaw may use either, unpredictably
- **Smart quotes** — `"` and `"` are not the same as `"`. Editors sometimes auto-convert. Use plain quotes.
- **Missing closing brace or bracket** — count `{` vs `}` and `[` vs `]`

After fixing, validate again, then restart.

## Port conflict

Symptoms: service won't bind, error mentions "address in use", `ss -ltnp | grep <GATEWAY_PORT>` shows another process.

```bash
ss -ltnp | grep <GATEWAY_PORT>
```

This shows the PID and process name holding the port. Two paths:

**Option A — kill the conflicting process:**

```bash
kill <PID>
# If it doesn't terminate:
kill -9 <PID>
```

Then restart OpenClaw.

**Option B — move OpenClaw to a different port:**

1. Edit `<CONFIG_PATH>`, change the gateway port setting (location depends on OpenClaw version — search for the current `<GATEWAY_PORT>` value).
2. Update the Cloudflare Tunnel route to point at the new port — see the `cloudflare-tunnel` skill.
3. Validate JSON, restart OpenClaw, restart `cloudflared`.

**Option C — leftover OpenClaw process:**

Sometimes a previous `openclaw` direct-process invocation didn't clean up. `pgrep -af openclaw` reveals it. Kill it, then start the systemd service fresh.

## Quick health check after any fix

```bash
openclaw gateway status
openclaw models status
curl -I http://127.0.0.1:<GATEWAY_PORT>
openclaw agent --agent main -m "Health check after fix."
```

All four should succeed. If only the last fails but the others pass, the issue is model-level — see `model-management.md` or `api-key-rotation.md`.

