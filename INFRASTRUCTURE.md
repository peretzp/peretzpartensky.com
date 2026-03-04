# Infrastructure — peretzpartensky.com

## Hosting Architecture

```
Build (Mac Studio)          Deploy (SSH)           Serve (NAS)              Edge (Cloudflare)
┌─────────────────┐    ┌──────────────────┐    ┌──────────────────────┐    ┌────────────────────┐
│ Astro build     │───>│ tar pipe via SSH  │───>│ Caddy v2.11.1        │───>│ Cloudflare Tunnel  │
│ npm run build   │    │ deploy-site cmd   │    │ ARM64 binary         │    │ CDN + auto-HTTPS   │
│ ~2 seconds      │    │ ~3 seconds        │    │ :8080 static files   │    │ Free tier          │
└─────────────────┘    └──────────────────┘    └──────────────────────┘    └────────────────────┘
```

## Quick Reference

| What | Where |
|------|-------|
| Site source | `~/peretzpartensky.com/` (this repo) |
| Build output | `~/peretzpartensky.com/dist/` |
| NAS web root | `peretz@192.168.1.57:/volume1/homes/peretz/www/peretzpartensky.com/` |
| NAS apps dir | `peretz@192.168.1.57:/volume1/homes/peretz/apps/` |
| Caddy config | NAS: `/volume1/homes/peretz/apps/Caddyfile` |
| Tunnel config | NAS: `/volume1/homes/peretz/apps/cloudflared-config.yml` |
| Boot script | NAS: `/volume1/homes/peretz/apps/start-all.sh` |
| Deploy script | Mac: `~/.local/bin/deploy-site` |
| Setup guide | `~/.claude/NAS-HOSTING-SETUP.md` |

## Deploy

```bash
# Deploy peretzpartensky.com (default)
deploy-site

# Deploy tolerableinsanity.com
deploy-site ti
```

What `deploy-site` does:
1. `npm run build` — generates static HTML in `dist/`
2. `tar czf | ssh tar xzf` — pipes build to NAS over SSH (port 2222)
3. Verifies HTTP 200 from NAS

## Access Points

| Network | URL | Use |
|---------|-----|-----|
| Public | `https://peretzpartensky.com` | Production (after domain transfer) |
| LAN | `http://192.168.1.57:8080` | Local testing |
| Tailscale | `http://100.93.227.12:8080` | Remote testing (iPad, iPhone, anywhere) |
| Quick tunnel | `https://*.trycloudflare.com` | Temporary public (for sharing/testing) |

## NAS Services

| Service | Binary | Port | RAM | Purpose |
|---------|--------|------|-----|---------|
| Caddy v2.11.1 | ARM64 standalone | 8080, 8081 | ~15MB | Static file server |
| cloudflared 2026.2.0 | ARM64 standalone | — | ~20MB | Cloudflare Tunnel |
| **Total** | | | **~35MB** | of 968MB available |

Binaries at: `/volume1/homes/peretz/apps/{caddy,cloudflared}`

## NAS Hardware

- **Model**: Synology DS223j
- **CPU**: Realtek RTD1619B (ARMv8/aarch64), 2 cores
- **RAM**: 1GB (non-upgradeable) — NO Docker support
- **Storage**: 16TB (2x 8TB), Btrfs, 14TB free
- **Network**: 1GbE LAN + Tailscale mesh VPN
- **Power**: ~15W, always-on
- **SSH**: Port 2222, key auth, user `peretz`

## Adding Content

### New essay:
```bash
# Create the markdown file
cat > src/content/writing/my-new-essay.md << 'EOF'
---
title: "My New Essay"
date: 2026-02-27
description: "What this essay is about"
---

Content here...
EOF

# Deploy
deploy-site
```

### New technical post:
```bash
cat > src/content/building/my-new-post.md << 'EOF'
---
title: "What I Built"
date: 2026-02-27
description: "Technical writeup"
---

Content here...
EOF

deploy-site
```

### Draft (hidden from listings):
Add `draft: true` to frontmatter. The page exists at its URL but doesn't appear in listing pages.

## Troubleshooting

```bash
# Check if Caddy is running on NAS
ssh -p 2222 peretz@192.168.1.57 "pgrep -la caddy"

# Check Caddy logs
ssh -p 2222 peretz@192.168.1.57 "tail -20 /volume1/homes/peretz/apps/caddy-access.log"

# Restart Caddy
ssh -p 2222 peretz@192.168.1.57 "/volume1/homes/peretz/apps/start-all.sh"

# Check cloudflared tunnel status
ssh -p 2222 peretz@192.168.1.57 "pgrep -la cloudflared"

# Quick tunnel for temporary public URL
ssh -p 2222 peretz@192.168.1.57 "/volume1/homes/peretz/apps/cloudflared tunnel --url http://localhost:8080"

# Check site from Mac Studio
curl -s http://192.168.1.57:8080 | grep '<title>'
curl -s http://100.93.227.12:8080 | grep '<title>'  # via Tailscale
```

## Future Additions

- **Umami** (self-hosted analytics) — needs Docker, will run on Mac Studio or Anvil (M3 Ultra)
- **Uptime Kuma** (monitoring) — same, needs more RAM than NAS has
- **Public API** — Node.js on NAS (v20 installed), endpoints for daily reconnect feed, life stream data
- **Media CDN** — large files (voice memo excerpts, images) served from NAS, cached on Cloudflare edge
