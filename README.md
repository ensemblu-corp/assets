# Ensemblu Assets & Documentation Hub

Official distribution centre for Ensemblu's public assets, branding and hosted micro-sites.

> Current release: **Project Axiom 3.0.0** · Java 26 · 10 modules · native drivers for PostgreSQL, MySQL, Oracle, SQLite and SQL Server

## Hosted pages

| Page | What it is |
|---|---|
| [Project Axiom homepage](https://ensemblu-corp.github.io/assets/index.html) | The 3.0 homepage: architecture explorer, module catalog, driver explorer, code lab, 2.x → 3.0 migration, articles, licensing and FAQ. Press `Ctrl K` / `⌘ K` to search. |
| [SchemaGuard documentation](https://ensemblu-corp.github.io/assets/schemaguard.html) | Perimeter validation with `axiom-language` |
| [Enterprise services](https://ensemblu-corp.github.io/assets/services.html) | Licensing, private deployments and custom drivers |

## Structure

| Path | Contents |
|---|---|
| `index.html` | The Axiom homepage: a single self-contained file, no build step, no framework, no CDN scripts |
| `blogs/` | Architectural notes, release changelogs and deep dives ([index](blogs/README.md)) |
| `images/` | Ecosystem covers, logos and visual assets |

## Editing the homepage

`index.html` is hand-written HTML, CSS and vanilla JavaScript:

- **Content data** (modules, drivers, code-lab recipes, migration steps, articles) lives in plain arrays near the bottom of the file (`MODULES`, `DRIVERS`, `LAB`, `MIG`, `POSTS`). Edit them, and the cards, tabs and search index update automatically.
- **Theme tokens** live in `:root` / `[data-theme="light"]` at the top of the `<style>` block.
- The theme choice is stored in `localStorage` (`axiom-theme`, plus the legacy `theme` key, so the older pages stay in sync).
- Animations respect `prefers-reduced-motion`.
