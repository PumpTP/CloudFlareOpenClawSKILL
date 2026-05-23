# Model management

How to add, change, or list models in `<CONFIG_PATH>`. Use this when the user wants a new model, a different default, or a model audit.

## List what the upstream offers

Before adding anything, confirm the upstream actually serves the model name you plan to use. Use `read -s` so the key never lands in shell history:

```bash
read -s LITELLM_API_KEY
curl LITELLM_BASE_URL/models \
  -H "Authorization: Bearer ${LITELLM_API_KEY}"
unset LITELLM_API_KEY
```

The response is JSON with `data: [...]` — each entry's `id` is what you put in `MODEL_NAME`.

## Add a new model alongside existing ones

Back up first:

```bash
cp <CONFIG_PATH> <CONFIG_PATH>.bak.$(date +%Y%m%d-%H%M%S)
nano <CONFIG_PATH>
```

Inside `providers.<PROVIDER_KEY>.models[]`, append a new object:

```json
{
  "id": "<NEW_MODEL_NAME>",
  "name": "<NEW_MODEL_DISPLAY_NAME>",
  "reasoning": false,
  "input": ["text"],
  "contextWindow": 128000,
  "maxTokens": 8192
}
```

Set `reasoning: true` only for reasoning models (o1, o3 family). Match `contextWindow` and `maxTokens` to what the upstream actually supports — over-claiming causes silent truncation, under-claiming wastes context.

## Change the default model

Edit `agents.defaults.model.primary`:

```json
"agents": { "defaults": { "model": { "primary": "<PROVIDER_KEY>/<NEW_MODEL_NAME>" } } }
```

The reference must use the full `<PROVIDER_KEY>/<NEW_MODEL_NAME>` form, even if there's only one provider configured.

## Switch to a different provider entirely

If the new model lives under a different provider (e.g., switching from `litellm/...` to `anthropic/...`):

1. Add a new provider block under `providers` (mirror the existing one's structure — see `openclaw-config.md`).
2. Add the model entry inside that provider's `models[]`.
3. Update `agents.defaults.model.primary` to `<NEW_PROVIDER_KEY>/<NEW_MODEL_NAME>`.

Keep the old provider block in place during the transition so you can fall back if the new one fails.

## Multi-model setup (primary + fallback)

Some OpenClaw versions support a fallback chain in `agents.defaults.model`:

```json
"agents": {
  "defaults": {
    "model": {
      "primary": "<MODEL_ID>",
      "fallback": ["<OTHER_PROVIDER_KEY>/<OTHER_MODEL_NAME>"]
    }
  }
}
```

Check `openclaw --help` or release notes for whether your version honors `fallback` — older versions ignore it silently.

## Validate, secure, restart

```bash
chmod 600 <CONFIG_PATH>
python3 -m json.tool <CONFIG_PATH>
systemctl --user restart openclaw-gateway.service
openclaw models status
```

`openclaw models status` should list the new model and show the correct default.

## Test the new model

```bash
openclaw agent --agent main -m "Hello. What model are you using?"
```

Model self-ID isn't always reliable — trust the config and `openclaw models status` over the agent's reply.

## Rollback

If the new model breaks the gateway, restore the backup:

```bash
cp <CONFIG_PATH>.bak.<TIMESTAMP> <CONFIG_PATH>
systemctl --user restart openclaw-gateway.service
```

