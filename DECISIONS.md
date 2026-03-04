# Decisions

Architectural decisions made during the initial scaffold (2026-02-27).

## Content Collections: Astro glob loader over astro-loader-obsidian

`astro-loader-obsidian` is installed as a dependency but not used in the initial content config. The two content collections (`writing` and `building`) use Astro's built-in `glob` loader pointed at local markdown directories. This keeps the initial setup simple and self-contained.

The Obsidian loader is available for a future "garden" or "notes" collection that could pull directly from the vault. Adding it later is a one-line change in `src/content.config.ts`.

## Two collections, not one

Writing (essays, observations) and Building (technical posts, tools) are separate content collections rather than a single "posts" collection with a category field. This gives each section its own directory, its own URL namespace (`/writing/slug` vs `/building/slug`), and its own listing page. It also makes it easy to add collection-specific frontmatter later (e.g., `repo` for building posts, `thread` for writing).

## Pure CSS, no framework

No Tailwind, no CSS framework, no JavaScript runtime. Styles are in a single `<style is:global>` block in `Base.astro`. This keeps the output as small as possible and the codebase trivially readable. The aesthetic is terminal-inspired: dark background (#0a0a0a), monospace font stack, muted colors, green accent.

## Static output only

`output: 'static'` in `astro.config.mjs`. The site is 100% pre-rendered HTML. No SSR, no server adapter, no client-side hydration. Every page is a plain HTML file.

## Existing site preserved

The previous single-file site (`index.html`) is backed up as `index.html.bak` and excluded from git via `.gitignore`. The git history from the old site is preserved in the same repo.

## Draft system

Posts with `draft: true` in frontmatter are filtered out of listing pages but still technically built by Astro's static route generation. They are unreachable unless someone guesses the URL. This is intentional -- it avoids the complexity of conditional route generation and matches the behavior of most static blog generators.

## Font stack

`'SF Mono', 'Fira Code', 'Cascadia Code', 'JetBrains Mono', monospace` -- covers macOS (SF Mono), Linux (Fira Code), Windows (Cascadia Code), and JetBrains IDEs. Falls back to the system monospace font.

## No RSS/sitemap yet

These are easy to add via `@astrojs/rss` and `@astrojs/sitemap` when there is actual content to syndicate. Not worth the dependency for a scaffold.
