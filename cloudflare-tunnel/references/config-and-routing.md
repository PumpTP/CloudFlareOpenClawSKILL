# `config.yml` and DNS routing

Two pieces wire a tunnel to a public hostname:

1. **Ingress rules** in `CONFIG_YML` — tell `cloudflared` how to route incoming requests to local services.
2. **DNS record** in the Cloudflare dashboard — points `PUBLIC_HOSTNAME` at the tunnel.

Both must agree. Mismatch = 530 or "DNS not found".

## Write `config.yml`

```bash
sudo nano CONFIG_YML
```

Minimal template for a single hostname:

```yaml
tunnel: TUNNEL_UUID
credentials-file: CREDENTIALS_FILE

ingress:
  - hostname: PUBLIC_HOSTNAME
    service: http://127.0.0.1:LOCAL_PORT
  - service: http_status:404
```

Notes:
- `tunnel:` is the UUID from `cloudflared tunnel create`, not `TUNNEL_NAME`.
- `credentials-file:` is the absolute path to the credentials JSON.
- The last `- service: http_status:404` is a catch-all required by `cloudflared` — every other rule above it is matched in order.
- `service:` must use `http://` (not `https://`) since the upstream is local plaintext.

## Multiple hostnames

Add more rules above the catch-all:

```yaml
ingress:
  - hostname: app.CLOUDFLARE_DOMAIN
    service: http://127.0.0.1:18789
  - hostname: api.CLOUDFLARE_DOMAIN
    service: http://127.0.0.1:18790
  - service: http_status:404
```

## Path-based routing

Within a single hostname, route by path:

```yaml
ingress:
  - hostname: PUBLIC_HOSTNAME
    path: /api/.*
    service: http://127.0.0.1:18790
  - hostname: PUBLIC_HOSTNAME
    service: http://127.0.0.1:18789
  - service: http_status:404
```

Order matters. More specific rules go first.

## Permissions

```bash
sudo chown root:root CONFIG_YML
sudo chmod 644 CONFIG_YML
```

The credentials file referenced inside should be `chmod 600` (see `install-and-auth.md`).

## Validate the config

```bash
sudo cloudflared tunnel ingress validate
```

Tests the YAML structure and ingress rule semantics. Fix any errors before continuing.

## Set up DNS routing

The tunnel can run, but until DNS points `PUBLIC_HOSTNAME` at it, no traffic arrives.

### Option A — let `cloudflared` write the DNS record

```bash
cloudflared tunnel route dns TUNNEL_NAME PUBLIC_HOSTNAME
```

This creates a `CNAME` record `PUBLIC_HOSTNAME` → `TUNNEL_UUID.cfargotunnel.com`, automatically Cloudflare-proxied.

### Option B — manually in the Cloudflare dashboard

DNS → Records → Add record:
- Type: `CNAME`
- Name: subdomain part of `PUBLIC_HOSTNAME` (e.g., `app` if hostname is `app.example.com`)
- Target: `TUNNEL_UUID.cfargotunnel.com`
- Proxy status: **Proxied (orange cloud)** — required. DNS-only (grey cloud) breaks routing.

### Verify DNS

```bash
dig +short PUBLIC_HOSTNAME
```

Should return Cloudflare proxy IPs (e.g., `104.x.x.x` or `172.x.x.x`), not your VM's actual IP. If it returns your VM's IP, the record isn't proxied — fix in dashboard.

## Test end-to-end

Start `cloudflared` (manually for first test, or via systemd — see `systemd-service.md`):

```bash
cloudflared tunnel run TUNNEL_NAME
```

In another shell:

```bash
curl -I https://PUBLIC_HOSTNAME
```

Expected: `HTTP/2 200`. If you get anything else, see `troubleshooting.md`.

When the manual test passes, stop the foreground process (`Ctrl+C`) and set it up as a service.

## Next step

Install as a systemd service → `systemd-service.md`.


