# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

The companion website for the **Affinity Network Workshop** (a four-day workshop on network design, Bolzano, July 6–9, 2026). It's a small Jekyll site with three pages, and everything analytical runs **client-side in the browser** — nothing is precomputed, committed, or uploaded:

- **Home** (`index.html`, permalink `/`) — workshop description, learning outcomes, approach, and a day-by-day schedule.
- **Text** (`text.html`, permalink `/text/`) — the "microscope": drop in **one** document and read it in layers (size, vocabulary, grammar, entities, readability, keywords, key phrases, and where terms appear). English only.
- **Network** (`network.html`, permalink `/network/`) — the "telescope": drop in a **set** of text files and watch them arrange into a similarity network. Embedding, similarity, and force layout all run in the browser.

> History: the site previously rendered a *pre-baked* publication graph computed offline in Python. That is gone, along with its `_publications/` abstracts and `scripts/`. If you find references to `_data/network.json`, a `network` layout, or "baked positions", they're stale — nothing in the current site produces or consumes them.

## Commands

```bash
bundle install
bundle exec jekyll serve            # http://localhost:4000/
bundle exec jekyll build --quiet    # sanity-check a change builds
```

There are no tests. Deployment is automatic: pushing to `main` triggers `.github/workflows/deploy.yml`, which builds with Jekyll and publishes to GitHub Pages. The Pages `--baseurl` is derived from the repo name in CI (`steps.pages.outputs.base_path`), so renaming the repo needs no config change. Locally `baseurl` is empty (serves at root).

## Architecture

Standard Jekyll. One layout, `_layouts/default.html`, wraps every page: it builds `<head>`, inlines the CSS (`{% include %}` of `nunito.css`, `main.css`, then each sheet named in the page's `styles:` front matter), renders `{% include site-nav.html %}`, and holds the shared dark-mode toggle.

**Each tool page is a single inline `<script type="module">`** — there is no build step and no shared JS file. Styling splits across `_includes/`: `network.css` (graph visuals: nodes, links, labels, stage — `#network-view`, `.node`, `.link`), `studio.css` (the builder chrome shared by both tools: control columns, sliders, toggles, overlays, dropzone), and `text.css` (the Text page's analysis layout). Note the `studio.*` filename/class names are a leftover from when the builder page was called "Studio"; the name stuck.

### The Network builder (`network.html`)

On **Build network**:

1. **Read** dropped `.txt`/`.md` files. Each is a node; its name is the file's first Markdown heading (`# …`) via `titleFromText()`, else the filename.
2. **Embed** each doc with `Xenova/multilingual-e5-small` through Transformers.js (`@huggingface/transformers@3` from the CDN), mean-pooled and normalized. Multilingual model → no translation step needed. The model (~110 MB) is fetched from the HF hub, browser-cached, and **preloaded** as soon as files are staged (`warmModel()`), so the load overlaps with the user reviewing files.
3. **Similarity** — pairwise cosine (`computeSimilarity()` builds a dense N×N matrix).
4. **Links** — `buildLinks(threshold)`: each node links to its single strongest neighbour, only if that score clears the live **Connection threshold**.
5. **Layout** — a live `d3.forceSimulation` the user can drag; forces map to the sliders.

**Live controls** (all mutate the running simulation, no rebuild):

- **Threshold** → rebuilds links; **Repulsion** → `charge` strength; **Gravity** → `forceX`/`forceY` strength toward centre; **Node spacing** → `collide` radius; **Link distance** → `forceLink` distance.
- **Node labels** / **Shorten titles** toggles; hover always shows the full title.
- **Stats overlay** (top-right of the graph): files, edges, clusters (union-find over links), unconnected, strongest/avg similarity, and "built in Ns". Recomputes live on threshold change.
- **PNG export** at 1×/2×/4×: `exportPNG(scale)` clones the SVG, inlines colours/fonts (external CSS doesn't apply to an SVG-in-`<img>`), and rasterises at the scaled size for crisp output. Label fonts may fall back to a system sans in the export.
- A built-in `SAMPLE_DOCS` corpus lets the page be tried without user files.

### The Text analyzer (`text.html`)

Input is a dropped `.txt`/`.md` file, text typed/pasted into the box beside the dropzone, or the bundled sample — all three feed the same staged text (files and the sample also fill the paste box). On **Analyze** it's broken into eight sections. Each has a right-hand aside — one short paragraph explaining the left column, with a single reference/library link, top-aligned to the figure rather than the heading — and each stat card carries a plain-language definition shown as a styled tooltip on hover/focus (`STAT_DESC` + the generic `[data-tip]` CSS, also used by the grade bars and formula rows):

1. **Measurements** — word/sentence/paragraph/character counts, average sentence length, longest sentence, average word length, and a ~200 wpm reading-time estimate, plus a *sentence-length* histogram (5-word buckets).
2. **Vocabulary** — unique words, lexical diversity (type–token ratio), lexical density (content words ÷ all words), and average uses per word, plus a *word-frequency spectrum* histogram (how many words occur 1×, 2×, … 11+×).
3. **Grammar** — noun : verb, adjectives : noun, and adverbs : verb ratios, plus active/passive **voice** (sentence-level via `#Passive`), a stacked *grammatical composition* bar (nouns/verbs/adjectives/adverbs/other, with a note that "other" is function words + punctuation).
4. **Entities** — distinct people/places/organizations counts, plus chip lists of the actual named entities per type (deduped, with a mention count when a name recurs; heuristic, so expect stray words).
5. **Readability** — Flesch reading-ease, syllables/word, and complex-word (3+ syllable) and long-word (>6 letter) shares, plotted on a labelled ease scale, plus a *grade-level by formula* bar chart comparing Flesch–Kincaid, Gunning fog, SMOG and Coleman–Liau on a fixed 0–24 axis (syllables estimated from vowel groups; assumes English).
6. **Keywords** — content-word frequency: nouns (singularised) and verbs (infinitive), so inflected forms merge.
7. **Key phrases** — noun phrases (head noun singularised), split on stopwords into runs of ≥2 content words, kept if they occur ≥2×.
8. **Where terms appear** — positional dispersion of the top keywords, matched against a lemma-aligned token stream so inflected forms register.

**compromise** (`compromise@14` from esm.sh, imported by the inline module) is created once and drives most of the page: it provides the shared **word/sentence token base** for Measurements and Vocabulary (via `doc.terms()` and `doc.sentences()`, so every section counts the same way — only paragraphs and characters stay plain, since compromise has no concept of them), plus part-of-speech tagging, voice detection, named-entity recognition, and lemmatisation for Grammar, Entities, Keywords, Key phrases and the dispersion stream. So the whole page needs a one-time CDN fetch (~200 KB, browser-cached), then runs instantly. Readability's syllable heuristic is the only non-compromise analysis. The whole page assumes **English**. Key helpers: `renderStats` (stat cards + tooltips), `renderSentHist`/`renderFreqSpectrum`/`renderGradeBars` (histograms and bar charts), `renderGrammar` (composition bar), `renderEntities`/`entityList` (entity chips), `buildLemmatizer` (surface→lemma map — nouns singularised in one whole-doc pass, verbs infinitivised per unique form because the whole-view `toInfinitive()` is O(n²)-slow), and `keywordCounts`/`keyphraseCounts`/`lemmatizedTerms` (all take the compromise `doc` plus that lemmatizer).

## Conventions

- Vanilla ES5-style JS in the inline modules (function expressions, no build step). Keep it dependency-free beyond D3 (vendored, `js/d3.v7.min.js`), Transformers.js (CDN, Network page only), and compromise (CDN, Text page only).
- CSS uses design tokens from `main.css` (`--fg`, `--bg`, `--muted`, `--border`, `--text-xs`, `--tracking-caps`, …) and supports light/dark via `html.dark`.
- After a JS change, a quick check is `node --check` on the extracted module plus `jekyll build`.
