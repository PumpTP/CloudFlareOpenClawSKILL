# LiteLLM API key rotation

Rotate when the key expires, gets revoked, hits its budget cap, or leaks. Also rotate proactively on a schedule.

## Procedure

1. **Get a fresh key** from the LiteLLM portal. Do not paste it into chat, terminal commands, or commits — only into the config file (Step 3).

2. **Back up the current config:**

   ```bash
   cp <CONFIG_PATH> <CONFIG_PATH>.bak.$(date +%Y%m%d-%H%M%S)
   ```

3. **Edit the config:**

   ```bash
   nano <CONFIG_PATH>
   ```

   Find the provider block:

   ```json
   "<PROVIDER_KEY>": {
     "baseUrl": "LITELLM_BASE_URL",
     "apiKey": "<OLD_KEY_HERE>",
     ...
   }
   ```

   Replace the `"apiKey"` value with the new key. Save and close.

4. **Lock down permissions and validate JSON:**

   ```bash
   chmod 600 <CONFIG_PATH>
   python3 -m json.tool <CONFIG_PATH>
   ```

   `chmod 600` ensures only the owner can read it. `json.tool` confirms the edit didn't break syntax.

5. **Test the new key against the upstream** before restarting OpenClaw. Use `read -s` so the key never enters shell history or `ps`:

   ```bash
   read -s LITELLM_API_KEY
   curl LITELLM_BASE_URL/models \
     -H "Authorization: Bearer ${LITELLM_API_KEY}"
   unset LITELLM_API_KEY
   ```

   Expected: JSON response listing available models. If you get `401`, `403`, or `Invalid API key`, the new key isn't ready — don't restart yet.

6. **Restart the gateway:**

   ```bash
   systemctl --user restart openclaw-gateway.service
   openclaw gateway status
   openclaw models status
   ```

7. **End-to-end test:**

   ```bash
   openclaw agent --agent main -m "Quick health check."
   ```

   Agent should reply. If it errors with auth, check `journalctl --user -u openclaw-gateway.service -n 50` — the key may have a typo or the wrong format.

## If the old key leaked

If the old key appeared in chat, git, logs, shell history, a screen recording, or any other shared surface:

1. **Revoke the old key in the LiteLLM portal immediately.** Do not wait for the new key to be deployed.
2. Then run the rotation procedure above.
3. Audit recent git commits: `git log -p -S "sk-" --all` to find any commits that touched a key. Use `git filter-repo` or BFG to scrub history if needed.
4. If the key landed in a public repository or paste site, treat it as already compromised — revoke first, panic never.

## Schedule

Even without a leak, rotate keys periodically (every 90 days is a reasonable default). This limits the blast radius if a leak happens unnoticed.

## Rollback

If the new key fails after restart and the old key is still valid:

```bash
cp <CONFIG_PATH>.bak.<TIMESTAMP> <CONFIG_PATH>
systemctl --user restart openclaw-gateway.service
```

Then investigate why the new key didn't work (wrong key copied, missing permissions on the portal side, wrong `LITELLM_BASE_URL`).

