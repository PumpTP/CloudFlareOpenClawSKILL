# WhatsApp plugin reference

Use this reference when the user asks about installing, checking, auditing, or securing the OpenClaw WhatsApp plugin.

## Install

```bash
npm install --prefix ~/.openclaw/npm @openclaw/whatsapp@2026.5.7
```

## Verify

```bash
npm list @openclaw/whatsapp --prefix ~/.openclaw/npm
```

## Audit

```bash
npm audit --prefix ~/.openclaw/npm
```

## Known audit issue

Known issue: `@openclaw/whatsapp` depends on Baileys/libsignal/protobufjs with a critical npm audit warning and no fix available.

This does not automatically mean the demo is unusable, but it means the plugin must be treated as higher risk.

## Mitigation

- Pin the plugin version instead of installing floating latest.
- Only allow trusted WhatsApp numbers.
- Keep the OpenClaw gateway behind Cloudflare Access.
- Disable or remove the plugin when it is not needed.
- Do not expose OpenClaw publicly without authentication.

