# peretzpartensky.com

Personal website built with Astro. Static site with writing and building sections.

## Setup

```bash
npm install
```

## Usage

```bash
npm run dev       # Local dev server
npm run build     # Build static site to dist/
npm run preview   # Preview production build
```

## Structure

```
src/
  pages/
    index.astro         # Landing page
    writing/            # Essays and observations
    building/           # Technical work and tools
  content/
    writing/*.md        # Markdown content for writing section
    building/*.md       # Markdown content for building section
  layouts/
    Base.astro          # Base layout (dark theme)
```

## Deployment

Static output deployed to NAS via Cloudflare Tunnel:

```bash
deploy-site pp    # Build + deploy to peretzpartensky.com
```

Hosted on Synology NAS (Caddy + cloudflared).

## Dependencies

- `astro` -- Static site generator
- `astro-loader-obsidian` -- Obsidian vault content loader

## Status

Live at [peretzpartensky.com](https://peretzpartensky.com). Early stage -- two content sections seeded.
