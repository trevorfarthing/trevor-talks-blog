# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pnpm dev          # Dev server on port 5000 (falls back to 5001)
pnpm build        # Production build
pnpm preview      # Preview production build locally
pnpm sync-images  # Sync images from Obsidian vault to public/
pnpm check-images # Validate image references in content
```

Note: This project uses **pnpm**, not npm.

## Architecture Overview

This is an **Astro 7 static site** functioning as a personal blog/portfolio. Content is authored in **Obsidian** and treated as the CMS (using Vault CMS with Astro Modular theme) — the site supports Obsidian-native syntax (wikilinks, callouts, embeds, image sizing) via custom remark/rehype plugins.
### Content System

Content lives in `src/content/` across five collection types: `posts`, `pages`, `projects`, `docs`, and `special`. All use glob loaders (Astro v6+ content layer) with Zod schemas defined in `src/content.config.ts`. Frontmatter is validated at build time.

### Custom Markdown Pipeline

The richest part of this codebase. `astro.config.mjs` chains a set of custom plugins in `src/utils/`:

- `remark-obsidian-embeds` — Obsidian `![[embed]]` syntax
- `remark-internal-links` — wikilink resolution
- `remark-callouts` — Obsidian callout blocks
- `remark-image-grids` — multi-image gallery layout
- `remark-mermaid` — Mermaid diagram rendering
- `rehype-mark` — `==highlight==` syntax
- Plus standard plugins: math (KaTeX), TOC, reading time, Shiki syntax highlighting

When something in markdown rendering breaks, start in `src/utils/` with the relevant plugin.

### Site Configuration

Nearly everything is toggled through `src/config.ts` (30KB): theme selection, feature flags (dark mode, search, comments, graph view, etc.), navigation, social links, SEO, and deployment platform. The `pnpm generate-deployment-config` script reads this and writes the appropriate `netlify.toml` / `vercel.json` / etc.

### Routing

Standard Astro file-based routing under `src/pages/`. Key patterns:
- `/posts/[...slug].astro` — individual posts
- `/posts/[page].astro` — paginated listing
- `/api/` — JSON endpoints (search index, OG image generation)
- `[...slug].astro` at root — catch-all for `pages` content

Redirects for URL migrations are configured directly in `astro.config.mjs`.

### Layouts

Five layouts in `src/layouts/`: `BaseLayout`, `PostLayout`, `PageLayout`, `ProjectLayout`, `DocumentationLayout`. `BaseLayout` owns navigation, footer, theme switching, and Swup page transitions. Post layout includes reading time, TOC, and Giscus comments.

### Build Scripts

`src/scripts/` contains pre-build Node scripts invoked via `pnpm` commands: image syncing from the Obsidian vault, alias processing for URL customization, and graph data generation (D3-powered knowledge graph visualization).
