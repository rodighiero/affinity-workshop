# Affinity Workshop

The companion website for a four-day workshop on network design — *turning a
corpus of texts into a visual network as a way of knowing* (Bolzano, July 6–9,
2026). Deployed with GitHub Pages at
[rodighiero.github.io/affinity-workshop](https://rodighiero.github.io/affinity-workshop/).

It is a small Jekyll site whose analytical tools all run **client-side in the
browser** — nothing is precomputed, committed, or uploaded:

- **Home** (`index.html`, `/`) — the workshop description, learning outcomes,
  approach, and a day-by-day schedule.
- **Text** (`text.html`, `/text/`) — the *microscope*: drop in **one** document
  and read it in layers (measurements, vocabulary, grammar, entities,
  readability, sentiment, keywords, recurring phrases, and a paragraph-by-
  paragraph narrative arc). English only.
- **Network** (`network.html`, `/network/`) — the *telescope*: drop in a **set**
  of text files and watch them arrange into a similarity network. Embedding,
  similarity, and force layout all run in the browser.
- **Audio** (`audio.html`, `/audio/`) — the telescope for **sound**: drop in a
  **set of short audio clips** and watch them arrange into a similarity network
  by how they *sound*. Embedding (CLAP), similarity, and force layout run in the
  browser; click a node to hear its clip. Reuses the Network page's
  similarity/force-layout engine.
- **Compare** (`compare.html`, `/compare/`) — two documents side by side: drop
  **one file in each of two slots**, and each *paragraph* becomes a node in a
  similarity network, coloured by its source document. Cross-document links are
  drawn as prominent *bridges* (shared themes = common language); within-document
  links stay faint (each text's own texture = specific language). A three-column
  panel below the graph lists each document's own vocabulary and the words they
  share.

## Run the site

```bash
bundle install
bundle exec jekyll serve            # http://localhost:4000/
bundle exec jekyll build --quiet    # sanity-check a change builds
```

There are no tests. Locally `baseurl` is empty, so the site serves at the root.
In CI the GitHub Pages workflow (`.github/workflows/deploy.yml`) builds with the
project subpath, derived automatically from the repo name — no manual change
needed if the repo is renamed. Pushing to `main` deploys.

## Runtime dependencies

The analytical pages fetch a few things from the internet at runtime; everything
else (D3 in `js/d3.v7.min.js`, the Nunito fonts in `fonts/`, and the bundled
`js/afinn.js` / `js/en-freq.js` data) is vendored.

- **Text** — [compromise](https://github.com/spencermwoodland/compromise)
  (`compromise@14`, ~200 KB) for tokenising, POS tagging, entities, and
  lemmatisation.
- **Network / Compare** — [Transformers.js](https://github.com/huggingface/transformers.js)
  plus the embedding model `Xenova/multilingual-e5-small` (~110 MB, downloaded
  once and then browser-cached). The multilingual model means mixed-language
  corpora work directly, with no translation step.
- **Audio** — Transformers.js plus **CLAP** (`Xenova/clap-htsat-unfused`, the
  ~34 MB q8 audio tower only) to embed sound.

## How the network pages work

Each tool page is a single inline `<script type="module">` — there is no build
step and no shared JS file. The Network, Audio, and Compare pages share the same
graph engine; on **Build network** the Network page:

1. **Reads** the dropped `.txt`/`.md` files. Each becomes a node; its name is the
   file's first Markdown heading (`# …`) if present, otherwise the filename.
2. **Embeds** each document with `Xenova/multilingual-e5-small` via
   Transformers.js (mean-pooled, normalized).
3. **Similarity** — pairwise cosine similarity across the embeddings.
4. **Links** — each node connects to its single strongest neighbour, but only if
   that similarity clears the **Connection threshold**.
5. **Layout** — a live D3 force simulation (`d3.v7`) the user can drag and tune.

The panel exposes live controls (threshold, repulsion, gravity, node spacing,
link distance), label toggles, a top-right stats overlay (files, edges,
clusters, unconnected, strongest/avg similarity, build time), and PNG export at
1×/2×/4×. Each of these pages ships a built-in sample set for a quick try without
your own files. Audio adds click-to-play and an *embedding layout* toggle
(classical MDS/PCA projection); Compare adds cross-document *bridges* and a
shared/exclusive vocabulary panel.

## Layout

```
_config.yml                  # site config
_layouts/default.html        # the only layout — head, nav, mode toggle, styles
index.html                   # Home (workshop info + schedule)
text.html                    # Text analyzer (/text/)
network.html                 # text similarity network builder (/network/)
audio.html                   # audio similarity network builder (/audio/)
compare.html                 # two-document paragraph comparison (/compare/)
_includes/
  site-nav.html              # shared header nav
  head-init.html             # early <head> script (theme, etc.)
  main.css                   # base styles + design tokens
  nunito.css                 # @font-face for the vendored fonts
  text.css                   # Text page analysis layout
  network.css                # graph visuals (nodes, links, labels, stage)
  studio.css                 # builder UI (columns, controls, overlays)
  audio.css                  # Audio page extras (playing node, etc.)
  compare.css                # Compare page extras (doc colours, bridges, vocab)
js/
  d3.v7.min.js               # vendored D3 v7
  afinn.js                   # AFINN-165 sentiment lexicon (Text)
  en-freq.js                 # top-5000 English word frequencies (Text)
fonts/                       # self-hosted Nunito
```
