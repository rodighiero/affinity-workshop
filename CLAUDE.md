# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

The companion website for the **Affinity Network Workshop** (a four-day workshop on network design, Bolzano, July 6–9, 2026). It's a small Jekyll site with two pages:

- **Home** (`index.html`) — workshop description, schedule, teachers, readings.
- **Network** (`network.html`, permalink `/network/`) — an **in-browser** builder that turns a set of dropped text files into a text/affinity similarity network. Embedding, similarity, and force layout all run **client-side** in the browser; nothing is precomputed or committed, and nothing leaves the machine.

> History: the site previously rendered a *pre-baked* publication graph computed offline in Python. That is gone. If you find references to `_data/network.json`, a `network` layout, or "baked positions", they're stale — see **Legacy** below.

## Commands

```bash
bundle install
bundle exec jekyll serve            # http://localhost:4000/
bundle exec jekyll build --quiet    # sanity-check a change builds
```

There are no tests. Deployment is automatic: pushing to `main` triggers `.github/workflows/deploy.yml`, which builds with Jekyll and publishes to GitHub Pages. The Pages `--baseurl` is derived from the repo name in CI (`steps.pages.outputs.base_path`), so renaming the repo needs no config change. Locally `baseurl` is empty (serves at root).

## Architecture

Standard Jekyll. One layout, `_layouts/default.html`, wraps every page: it builds `<head>`, inlines the CSS (`{% include %}` of `nunito.css`, `main.css`, then each sheet named in the page's `styles:` front matter), renders `{% include site-nav.html %}`, and holds the shared dark-mode toggle. Pages are `index.html` and `network.html`.

**All the interesting logic lives in `network.html`** — its inline `<script type="module">` is the entire builder. Styling splits across `_includes/network.css` (graph visuals: nodes, links, labels, stage — shared class names like `#network-view`, `.node`, `.link`) and `_includes/studio.css` (the builder chrome: the two control columns, sliders, toggles, overlays). Note the CSS filename is still `studio.css` and classes are still `studio-*` — the page was renamed from "Studio" to "Network" but the internal names were left alone.

### The builder pipeline (`network.html`)

On **Build network**:

1. **Read** dropped `.txt`/`.md` files. Each is a node; its name is the file's first Markdown heading (`# …`) via `titleFromText()`, else the filename.
2. **Embed** each doc with `Xenova/multilingual-e5-small` through Transformers.js (`@huggingface/transformers@3` from the CDN), mean-pooled and normalized. Multilingual model → no translation step needed. The model (~110 MB) is fetched from the HF hub, browser-cached, and **preloaded** as soon as files are staged (`warmModel()`), so the load overlaps with the user reviewing files.
3. **Similarity** — pairwise cosine (`computeSimilarity()` builds a dense N×N matrix).
4. **Links** — `buildLinks(threshold)`: each node links to its single strongest neighbour, only if that score clears the live **Connection threshold**.
5. **Layout** — a live `d3.forceSimulation` the user can drag; forces map to the sliders.

### Live controls (all mutate the running simulation, no rebuild)

- **Threshold** → rebuilds links; **Repulsion** → `charge` strength; **Gravity** → `forceX`/`forceY` strength toward centre; **Node spacing** → `collide` radius; **Link distance** → `forceLink` distance.
- **Node labels** / **Shorten titles** toggles; hover always shows the full title.
- **Stats overlay** (top-right of the graph): files, edges, clusters (union-find over links), unconnected, strongest/avg similarity, and "built in Ns". Recomputes live on threshold change.
- **PNG export** at 1×/2×/4×: `exportPNG(scale)` clones the SVG, inlines colours/fonts (external CSS doesn't apply to an SVG-in-`<img>`), and rasterises at the scaled size for crisp output. Label fonts may fall back to a system sans in the export.
- A built-in `SAMPLE_DOCS` corpus lets the page be tried without user files.

## Conventions

- Vanilla ES5-style JS in the inline module (function expressions, no build step). Keep it dependency-free beyond D3 (vendored, `js/d3.v7.min.js`) and Transformers.js (CDN).
- CSS uses design tokens from `main.css` (`--fg`, `--bg`, `--muted`, `--border`, `--text-xs`, `--tracking-caps`, …) and supports light/dark via `html.dark`.
- After a JS change, a quick check is `node --check` on the extracted module plus `jekyll build`.

## Legacy files (unused by the site)

Leftovers from the old baked-graph version, safe to delete: `scripts/build-network.py`, `scripts/layout-network.js`, `_publications/*.md`, `requirements.txt`, `_data/translations-cache.json`. The current site references none of them.
