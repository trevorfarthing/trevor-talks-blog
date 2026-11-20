# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pnpm dev          # Runs all pre-build scripts, then dev server on port 5000 (falls back to 5001)
pnpm build        # Runs all pre-build scripts, then astro build -> dist/
pnpm preview      # Build, then serve dist/ locally
pnpm sync-images  # Copy src/content/*/attachments -> public/*/attachments (sharp-optimized)
pnpm check-images # Validate image references in content
pnpm generate-deployment-config:dry-run  # Preview redirect/deploy config output without writing
pnpm update       # Pull upstream Astro Modular theme updates
```

Note: This project uses **pnpm**, not npm (overrides the global npm preference). There is no test suite or linter. Use `pnpm build` as the correctness check, since it validates frontmatter and type-checks content.

## Architecture Overview

This is an **Astro 7 (alpha) static site** functioning as a personal blog/portfolio, built on the upstream **Astro Modular** theme (by David V. Kimball). Content is authored in **Obsidian** and treated as the CMS via Vault CMS. `src/content/` *is* the Obsidian vault, so there is no external vault path. Commits titled "Blog update ..." come from the Obsidian git workflow. Theme internals may be overwritten by `pnpm update`, so prefer changes in `src/config.ts` and `src/themes/custom/` where possible.

### Content System

Content lives in `src/content/` across five collections: `posts`, `pages`, `projects`, `docs`, and `special`. All use glob loaders with Zod schemas in `src/content.config.ts`. Frontmatter is validated at build time. Each collection keeps images in an `attachments/` folder that `sync-images` mirrors into `public/`. `src/content/Blog/` and `src/content/bases/` are vault-only folders and not collections.

### Custom Markdown Pipeline

The richest part of this codebase. `astro.config.mjs` chains custom plugins from `src/utils/`, and **order matters** (see inline comments there):

- `internallinks.ts` — wikilink resolution, plus `remarkFolderImages` and `remarkImageCaptions`
- `remark-obsidian-image-size` (runs first), `remark-obsidian-comments` (strips `%%...%%`), `remark-inline-tags`
- `remark-obsidian-embeds` — `![[embed]]` syntax
- `remark-bases` — Obsidian Bases (`.base`) rendered as tables
- `remark-callouts`, `remark-image-grids`, `remark-mermaid`
- `rehype-mark` (`==highlight==`), `rehype-image-attributes`, `rehype-normalize-anchors` (must stay last)
- Plus standard plugins: math (KaTeX), TOC, reading time, `remark-breaks`, Shiki (`github-dark`)

When something in markdown rendering breaks, start in `src/utils/` with the relevant plugin.

### Site Configuration

Nearly everything is toggled through `src/config.ts` (30KB): theme selection, feature flags (dark mode, search, comments, graph view, etc.), navigation, social links, SEO, and deployment platform. The deployment platform is currently **`cloudflare-workers`**: `wrangler.toml` serves `dist/` as static assets. `netlify.toml` is a leftover from the theme. Path aliases (`@/components`, `@/utils`, `@/config`, etc.) are defined in `astro.config.mjs`.

### Routing

Standard Astro file-based routing under `src/pages/`. Key patterns:
- `/posts/[...slug].astro` — individual posts
- `/posts/[page].astro` — paginated listing
- `/api/` — JSON endpoints (search index, OG image generation)
- `[...slug].astro` at root — catch-all for `pages` content
- Root-level `rss.xml`, `feed.xml`, `sitemap.xml`, `robots.txt`, `llms.txt` endpoints

**Redirects are generated, so don't hand-edit them.** Add `aliases:` to a note's frontmatter instead. `scripts/process-aliases.js` rewrites the `redirects` block in `astro.config.mjs`, which only applies in dev. Then `scripts/generate-deployment-config.js` writes `public/_redirects` for production.

### Layouts

Five layouts in `src/layouts/`: `BaseLayout`, `PostLayout`, `PageLayout`, `ProjectLayout`, `DocumentationLayout`. `BaseLayout` owns navigation, footer, theme switching, and Swup page transitions. Post layout includes reading time, TOC, and Giscus comments.

### Build Scripts

Root `scripts/` holds the Node scripts that `dev` and `build` run in sequence: `sync-images` → `process-aliases` → `generate-deployment-config` → `generate-graph-data` (D3 knowledge graph data). These scripts modify tracked files (`astro.config.mjs`, `public/`, and content frontmatter), so expect diffs after running dev or build. `src/scripts/` is different: it holds client-side TS (Mermaid, Bases embeds). `src/integrations/refresh-content-on-change.ts` handles dev hot-reload of content.
