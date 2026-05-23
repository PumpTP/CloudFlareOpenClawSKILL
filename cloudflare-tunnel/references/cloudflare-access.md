# Cloudflare Access for sensitive upstreams

Use this reference when the tunnel exposes anything more sensitive than a static site — admin dashboards, agent gateways (e.g., OpenClaw), database UIs, internal tools.

## Rule

Public tunnels to admin or agent surfaces must sit behind Cloudflare Access.

A Cloudflare Tunnel by itself only hides the origin IP. It does **not** authenticate visitors. Anyone who knows the hostname can reach the upstream. For surfaces that control tools, browser sessions, plugins, local files, or third-party integrations, that's not acceptable — treat the tunnel as an admin door and require auth.

## Minimum policy

Create an Access application for the public hostname and allow only trusted identities.

- Application hostname: `PUBLIC_HOSTNAME`
- Policy action: Allow
- Identity rule: specific emails, or a trusted email domain
- Session duration: short enough to fit the use case (e.g., 1–8h for demos)
- DNS record stays proxied through Cloudflare (orange cloud)

## Verify before going live

Open `https://PUBLIC_HOSTNAME` in a private/incognito window.

Expected: Cloudflare Access login screen **before** the upstream loads.

If the upstream loads directly without login, the policy isn't enforcing. Stop the tunnel or remove the DNS route until Access is fixed.

## OpenClaw note

OpenClaw is an agent gateway with tool, browser, plugin, and integration access (e.g., WhatsApp). It is a sensitive upstream under this rule — never expose it publicly without an Access policy.

