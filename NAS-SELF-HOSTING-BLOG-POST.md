# I Self-Host My Publishing Stack on a $200 NAS

My entire web presence — personal site, essay archive, deployment pipeline — runs on a Synology DS223j. It's a $200 plastic box with 1GB of RAM and an ARM processor slower than my phone. It serves my sites through Cloudflare's global CDN, costs $24/year total, and I deployed the whole thing in under an hour.

This isn't a flex. This is what happens when you stop treating infrastructure as something you rent and start treating it as something you own.

## The Setup

I have three domains:

- **peretzpartensky.com** — personal site (Astro static site)
- **tolerableinsanity.com** — essay archive (years of travel writing)
- **jalalagood.com** — collaborative project (Afghanistan blog)

Previously, all three sat on DreamHost shared hosting. I was paying $60-180/year depending on the plan, plus $16-20/year per domain. The sites were WordPress installs I hadn't updated in months. One had been hacked twice.

Now they run on a NAS in my living room. Here's the architecture:

## The Architecture

```
Public Internet
    │
    ▼
Cloudflare Edge (CDN, DDoS protection, auto-HTTPS, caching)
    │
    ▼ (Cloudflare Tunnel — encrypted, zero port forwarding)
    │
Synology DS223j NAS ($200, 16TB, 1GB RAM, always-on)
    ├── Caddy v2.11.1 (reverse proxy + static file server)
    │   ├── peretzpartensky.com → Astro static build (:8080)
    │   └── tolerableinsanity.com → archived travel writing (:8081)
    └── cloudflared (Cloudflare Tunnel daemon)

Private access (for development):
    Tailscale mesh VPN → NAS directly
    Works from: Mac, iPhone, iPad — anywhere in the world
```

The key insight: **the NAS is the origin server, Cloudflare is the CDN**. Cloudflare Tunnel creates an encrypted connection from my NAS to Cloudflare's edge — no port forwarding, no dynamic DNS, no exposing my home IP. Cloudflare caches static assets globally and handles HTTPS automatically.

From the outside, it looks like a professionally hosted site. From the inside, it's a plastic box next to my router.

## Why Not Docker?

I planned to run everything in Docker containers. Caddy, cloudflared, analytics, monitoring — all neatly containerized. Then I SSH'd into my NAS:

```
$ free -m
              total        used        free
Mem:            968         241         111
```

968MB total RAM. The DS223j is Synology's budget model — ARM processor, 1GB non-upgradeable RAM, no Container Manager support. Docker was dead on arrival.

So I adapted. Instead of containers, I downloaded standalone ARM64 binaries:

- **Caddy** — 44MB binary, serves static files with gzip and security headers
- **cloudflared** — 37MB binary, maintains the Cloudflare Tunnel

Together they use ~35MB of RAM. The NAS doesn't even notice them.

```bash
# The entire "deployment infrastructure"
$ ls -lh /volume1/homes/peretz/apps/
total 78M
-rwx 43M caddy
-rwx 36M cloudflared
-rw-  1K Caddyfile
-rw-  1K cloudflared-config.yml
-rwx  1K start-all.sh
```

Five files. 78MB. That's the whole stack.

## The Deployment Pipeline

My site is built with [Astro](https://astro.build/) — a static site generator that outputs plain HTML. No JavaScript framework, no server-side rendering. Just HTML, CSS, and content.

Deploy is one command:

```bash
deploy-site
```

Here's what it does:

```bash
#!/bin/bash
# 1. Build the Astro site locally (takes ~2 seconds)
cd ~/peretzpartensky.com && npm run build

# 2. Pipe the build output directly to the NAS over SSH
cd dist && tar czf - . | ssh -p 2222 peretz@192.168.1.57 \
  "cd /volume1/homes/peretz/www/peretzpartensky.com && tar xzf -"

# 3. Verify
curl -s http://192.168.1.57:8080 | grep '<title>'
```

Build locally, tar pipe over SSH, done. No CI/CD service. No GitHub Actions. No build minutes to pay for. The build runs on my Mac (fast), the files land on the NAS (always-on), and Cloudflare caches them globally (free).

Adding a new essay:

```bash
# Write the essay
vim ~/peretzpartensky.com/src/content/writing/new-essay.md

# Deploy
deploy-site

# Live in 5 seconds
```

## The Caddyfile

```
:8080 {
    root * /volume1/homes/peretz/www/peretzpartensky.com
    file_server
    encode gzip

    header {
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        Referrer-Policy strict-origin-when-cross-origin
        Cache-Control "public, max-age=3600"
    }
}

:8081 {
    root * /volume1/homes/peretz/www/tolerableinsanity.com
    file_server
    encode gzip
}
```

That's the entire web server configuration. Caddy's defaults are sane — it handles everything else.

## The Cloudflare Tunnel

```yaml
tunnel: nas-homelab
credentials-file: ~/.cloudflared/TUNNEL_ID.json

ingress:
  - hostname: peretzpartensky.com
    service: http://localhost:8080
  - hostname: tolerableinsanity.com
    service: http://localhost:8081
  - service: http_status:404
```

The tunnel runs as a background process on the NAS. It maintains a persistent connection to Cloudflare's edge. When someone visits `peretzpartensky.com`, Cloudflare routes the request through the tunnel to Caddy on the NAS. No port 80 or 443 open on my network. No firewall rules. No dynamic DNS updates.

The NAS connects to Cloudflare. Cloudflare connects to the world. My home network stays invisible.

## Boot Resilience

Both services start on NAS boot via Synology's Task Scheduler:

```bash
#!/bin/sh
# /volume1/homes/peretz/apps/start-all.sh
cd /volume1/homes/peretz/apps
./caddy run --config Caddyfile &
./cloudflared tunnel --config cloudflared-config.yml run &
echo "NAS services started at $(date)" >> boot.log
```

The NAS draws ~15W and runs 24/7. It's backed by a UPS. After a power outage, it boots, starts Tailscale (VPN mesh), starts Caddy, starts cloudflared, and the sites are back online — no human intervention.

## Private Access via Tailscale

I don't just serve public sites from the NAS — I also access it privately from everywhere. Tailscale creates a mesh VPN across all my devices:

| Device | Tailscale IP |
|--------|-------------|
| Mac Studio | 100.100.182.4 |
| NAS | 100.93.227.12 |
| iPad Pro | 100.111.181.14 |
| iPhone | 100.64.121.40 |

From my iPad at a coffee shop: `http://100.93.227.12:8080` — instant access to the site for review. No public URL needed. No VPN server to maintain.

## Cost Comparison

| | Before (DreamHost) | After (NAS) |
|---|---|---|
| Hosting | $60-180/yr | $0 (NAS already running) |
| Domains (3) | $48-60/yr | $24/yr (Cloudflare at-cost) |
| SSL | included | $0 (Cloudflare auto-HTTPS) |
| CDN | none | $0 (Cloudflare free tier) |
| Analytics | none | $0 (self-hosted, planned) |
| DDoS protection | none | $0 (Cloudflare free tier) |
| **Total** | **$108-240/yr** | **$24/yr** |

Plus I get: global CDN, DDoS protection, automatic HTTPS, private mesh access from all devices, and full data sovereignty. Every byte lives on hardware I own.

## What I Learned

**1GB of RAM is enough.** Caddy and cloudflared together use 35MB. Static file serving doesn't need Docker, Kubernetes, or even a proper web framework. A binary and a config file will do.

**The binary fallback is underrated.** When Docker isn't available, standalone binaries are the next best thing. Caddy and cloudflared both ship single-file binaries for every architecture. Download, chmod +x, run. No package manager, no dependency hell, no container runtime.

**Cloudflare Tunnel is the missing piece.** I've self-hosted things before. The pain was always: dynamic DNS, port forwarding, SSL certificates, DDoS exposure. Cloudflare Tunnel eliminates all of it. One outbound connection from your NAS, and you're globally accessible with auto-HTTPS and CDN caching. Free tier.

**The NAS was already running.** This is the part people miss. I didn't buy a NAS to host websites. I bought it for Time Machine backups and file storage. It was already on, already drawing power, already on my network. Adding a web server was 78MB of disk space and 35MB of RAM. The marginal cost of self-hosting was effectively zero.

**Deploy scripts beat CI/CD for personal sites.** GitHub Actions, Vercel, Netlify — they're all fine. But for a personal site updated weekly, a 3-line shell script that tar-pipes over SSH is simpler, faster, and has zero vendor lock-in. I can move my deploy target by changing one IP address.

## The Stack

For the curious, here's everything:

| Layer | Tool | Why |
|-------|------|-----|
| Static site generator | [Astro](https://astro.build/) | Fast, zero JS by default, markdown content collections |
| Web server | [Caddy](https://caddyserver.com/) | Single binary, great defaults, automatic HTTPS |
| Tunnel | [cloudflared](https://github.com/cloudflare/cloudflared) | Zero port forwarding, free, auto-HTTPS via Cloudflare |
| CDN | [Cloudflare](https://cloudflare.com/) (free tier) | Global edge caching, DDoS protection |
| VPN mesh | [Tailscale](https://tailscale.com/) (free tier) | Private access from all devices |
| NAS | [Synology DS223j](https://www.synology.com/en-us/products/DS223j) | $200, 16TB, ARM64, always-on, 15W |
| Deploy | `deploy-site` shell script | tar pipe over SSH, 3 lines |

Total recurring cost: **$24/year** (three domain renewals at Cloudflare at-cost pricing).

## What's Next

This is the second open-source infrastructure tool I'm extracting from my personal setup. The first was [download-router](https://github.com/peretzp/download-router) — a zero-dependency file organizer with daemon mode that the most-starred file organizer on GitHub still can't do.

Next up: crash-resilient session logging for AI agents — SQLite + FTS5 for searchable, crash-proof conversation history.

The pattern is the same each time: build something for yourself, use it until it's proven, extract and publish. Infrastructure as autobiography.

---

*I was born in the Soviet Union in 1981 and have lived in four countries. I build infrastructure for self-knowledge — 929 voice memos indexed into a searchable archive, a self-organizing filesystem, a personal dashboard that probes 15 data sources. More at [peretzpartensky.com](https://peretzpartensky.com).*
