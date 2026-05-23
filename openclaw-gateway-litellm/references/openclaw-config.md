# OpenClaw config (`openclaw.json`)

Full schema for the LiteLLM provider block plus the agent defaults. Use this when first configuring the gateway or when restructuring providers.

## Always back up first

```bash
cp <CONFIG_PATH> <CONFIG_PATH>.bak.$(date +%Y%m%d-%H%M%S)
```

## Edit

```bash
nano <CONFIG_PATH>
```

## Required structure

Substitute placeholders from the Defaults table in SKILL.md (or whatever values the user specified).

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "<PROVIDER_KEY>": {
        "baseUrl": "LITELLM_BASE_URL",
        "apiKey": "LITELLM_API_KEY",
        "api": "openai-completions",
        "models": [
          {
            "id": "MODEL_NAME",
            "name": "<MODEL_DISPLAY_NAME>",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 128000,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "<MODEL_ID>"
      }
    }
  }
}
```

## Naming rules

The model identifier splits across three places. Get this wrong and OpenClaw can't resolve the agent's primary model.

- `<PROVIDER_KEY>` — first half of `<MODEL_ID>`, before the slash. Also the key under `providers`.
- `MODEL_NAME` — second half of `<MODEL_ID>`, after the slash. Goes in the `id` field of the provider's model entry.
- `<MODEL_ID>` — full `<PROVIDER_KEY>/MODEL_NAME` reference. Goes in `agents.defaults.model.primary`.

Example: for `litellm/gpt-4o-mini`:
- Provider key in `providers`: `"litellm"`
- Model `id` inside the array: `"gpt-4o-mini"`
- Agent reference: `"litellm/gpt-4o-mini"`

## Field reference

| Field | Purpose |
|-------|---------|
| `models.mode` | `"merge"` — combine these models with any built-in defaults. Use `"replace"` to fully override. |
| `providers.<PROVIDER_KEY>.baseUrl` | Upstream API root. Include `/v1` if the provider follows OpenAI conventions. |
| `providers.<PROVIDER_KEY>.apiKey` | Secret credential. Lives only here. |
| `providers.<PROVIDER_KEY>.api` | API dialect. `"openai-completions"` for LiteLLM and OpenAI. |
| `providers.<PROVIDER_KEY>.models[].id` | The exact model name the upstream expects. |
| `providers.<PROVIDER_KEY>.models[].name` | Display label only — shown in dashboard. |
| `providers.<PROVIDER_KEY>.models[].reasoning` | `true` for reasoning models (o1, o3 family). `false` for everything else. |
| `providers.<PROVIDER_KEY>.models[].input` | Modalities accepted. `["text"]` for text-only. |
| `providers.<PROVIDER_KEY>.models[].contextWindow` | Max input tokens the upstream supports. |
| `providers.<PROVIDER_KEY>.models[].maxTokens` | Max output tokens per response. |
| `agents.defaults.model.primary` | Full `<PROVIDER_KEY>/MODEL_NAME` reference used by default. |

## Validate and secure

After saving, always:

```bash
chmod 600 <CONFIG_PATH>
python3 -m json.tool <CONFIG_PATH>
```

If `json.tool` errors, the file is invalid. Common mistakes: missing comma, trailing comma, unescaped quote, duplicate keys. Fix before restarting the gateway.

## Restart

```bash
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

Then verify per the "Test the gateway" section in SKILL.md.

