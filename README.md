# Affinity Workshop

The companion website for a four-day workshop on network design — *turning a
corpus of texts into a visual network as a way of knowing* (Bolzano, July 6–9,
2026). Deployed with GitHub Pages at
[rodighiero.github.io/affinity-workshop](https://rodighiero.github.io/affinity-workshop/).

It is a small Jekyll site with two pages:

- **Home** (`index.html`) — the workshop description, learning outcomes,
  schedule, teachers, and readings.
- **Network** (`network.html`, at `/network/`) — an **in-browser** builder that
  turns a set of text files into a **text/affinity similarity network**. Drop in
  `.txt`/`.md` files and it embeds them, computes pairwise similarity, and lays
  them out as a force-directed graph — all client-side. Nothing is precomputed,
  and nothing leaves the browser.

## Run the site

```bash
bundle install
bundle exec jekyll serve   # http://localhost:4000/
```

Locally `baseurl` is empty, so the site serves at the root. In CI the GitHub
Pages workflow builds with the project subpath (`--baseurl "/<repo-name>"`),
derived automatically from the repo — no manual change needed if the repo is
renamed.

The Network page pulls two things from the internet at runtime: the
[Transformers.js](https://github.com/huggingface/transformers.js) library and
the embedding model (`Xenova/multilingual-e5-small`, ~110 MB, downloaded once
and then browser-cached). D3 (`js/d3.v7.min.js`) and the Nunito fonts
(`fonts/`) are vendored.

## How the Network page works

Everything runs in `network.html` (markup + inline ES-module script); styling is
in `_includes/network.css` (shared graph visuals) and `_includes/studio.css`
(the builder UI). The pipeline, on **Build network**:

1. **Read** the dropped files. Each becomes a node; its name is the file's first
   Markdown heading (`# …`) if present, otherwise the filename.
2. **Embed** each document with `Xenova/multilingual-e5-small` via
   Transformers.js (mean-pooled, normalized). Mixed-language corpora work
   directly — the multilingual model needs no translation step.
3. **Similarity** — pairwise cosine similarity across the embeddings.
4. **Links** — each node connects to its single strongest neighbour, but only if
   that similarity clears the **Connection threshold**.
5. **Layout** — a live D3 force simulation (`d3.v7`) the user can drag and tune.

The panel exposes live controls (threshold, repulsion, gravity, node spacing,
link distance), label toggles, a top-right stats overlay (files, edges,
clusters, unconnected, strongest/avg similarity, build time), and PNG export at
1×/2×/4×. A built-in sample set is available for a quick try without your own
files.

## Layout

```
_config.yml                  # site config (title, baseurl note)
_layouts/default.html        # the only layout — head, nav, mode toggle, styles
index.html                   # Home (workshop info + schedule)
network.html                 # in-browser network builder (/network/)
_includes/
  site-nav.html              # shared header nav
  head-init.html             # early <head> script (theme, etc.)
  main.css                   # base styles
  network.css                # graph visuals (nodes, links, labels, stage)
  studio.css                 # builder UI (columns, controls, overlays)
  nunito.css                 # @font-face for the vendored fonts
fonts/                       # self-hosted Nunito
js/d3.v7.min.js              # vendored D3 v7
```

## Legacy files (unused)

Earlier the site rendered a *pre-baked* publication graph computed offline. That
version is gone, but some of its inputs are still in the tree and are **no longer
used by the site**:

- `scripts/build-network.py`, `scripts/layout-network.js` — the old offline
  embed/layout pipeline.
- `_publications/*.md` — the source abstracts it consumed.
- `requirements.txt` — its Python dependencies.
- `_data/translations-cache.json` — its translation cache.

They can be deleted whenever you like; the current site doesn't reference them.
