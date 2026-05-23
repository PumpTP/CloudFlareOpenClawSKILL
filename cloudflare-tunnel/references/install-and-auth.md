# Install and authenticate `cloudflared`

First-time setup on a Linux VM. Skip ahead if `cloudflared --version` already works.

## Install on Ubuntu/Debian

```bash
# Add Cloudflare's apt repo
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt-get update
sudo apt-get install -y cloudflared
```

Verify:

```bash
cloudflared --version
```

## Authenticate

This opens a browser-based flow that ties your VM to your Cloudflare account.

```bash
cloudflared tunnel login
```

The command prints a URL — open it in any browser, log into Cloudflare, and select the domain `CLOUDFLARE_DOMAIN`. The browser writes a certificate to `~/.cloudflared/cert.pem`. This certificate authorizes the VM to create and manage tunnels in your account.

On a headless server, run the `tunnel login` command, copy the URL it prints, open it from a desktop browser, and authenticate there — the VM polls Cloudflare and picks up the cert automatically.

## Create the tunnel

```bash
cloudflared tunnel create TUNNEL_NAME
```

Output includes:
- A tunnel UUID
- Path to a credentials JSON file (e.g., `~/.cloudflared/<UUID>.json`)

This credentials file is sensitive — treat it like an API key. Move it to a system location with strict permissions:

```bash
sudo mkdir -p /etc/cloudflared
sudo mv ~/.cloudflared/<UUID>.json /etc/cloudflared/
sudo chown root:root /etc/cloudflared/<UUID>.json
sudo chmod 600 /etc/cloudflared/<UUID>.json
```

Record the UUID — you'll need it for `config.yml`.

## List tunnels

To confirm or list existing tunnels:

```bash
cloudflared tunnel list
```

## Next step

Wire the tunnel to a hostname and local port → see `config-and-routing.md`.

