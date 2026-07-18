# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained static page (`index.html`) — an interactive Elden Ring "Rune Level 1 / no level-up" playthrough guide (base game + Shadow of the Erdtree). There is **no build system, no dependencies, and no tests**. The entire site is one HTML file with an inline `<style>` block and one inline `<script>`; `README.md` and `.nojekyll` are the only other tracked files.

## Workflow

**Commit and push after every change.** This repo is the single source of truth (the guide is no longer maintained as a claude.ai Artifact), and pushing to `main` is what publishes the live site. After any edit that leaves the guide in a working state, `git commit` and `git push origin main` — don't batch many edits into an unpushed working tree.

## Commands

- **Preview locally:** open `index.html` directly, or serve it — `python3 -m http.server` in the repo root, then visit `http://localhost:8000/`. Use a server (not `file://`) when testing checklist progress, since it relies on `localStorage`.
- **Deploy:** commit to `main` and push. GitHub Pages serves `main` / root and rebuilds automatically (live at https://marano.github.io/elden-ring-level-1-guide/). `.nojekyll` disables Jekyll so the raw file is served as-is.
- **Structural sanity check** (there's no test runner; this is the substitute after edits):
  ```sh
  grep -c '<section' index.html; grep -c '</section>' index.html        # must match
  grep -c '<details' index.html; grep -c '</details>' index.html        # must match
  # every WIKI_URLS key (value starts with FEX) and MAP_PINS key (value "id=…")
  # must point at a real step id:
  for k in $(grep -oE '"[a-z]+[0-9]+": FEX' index.html | grep -oE '[a-z]+[0-9]+'); do
    grep -q "id=\"$k\"" index.html || echo "ORPHAN WIKI KEY: $k"; done
  for k in $(grep -oE '"[a-z]+[0-9]+": "id=' index.html | grep -oE '[a-z]+[0-9]+'); do
    grep -q "id=\"$k\"" index.html || echo "ORPHAN MAP KEY: $k"; done
  ```

## Hard constraint: stay self-contained

`index.html` must load **zero external resources** — no CDN scripts, remote fonts, or remote images. Inline all CSS/JS; embed any asset as a `data:` URI (the favicon is an emoji SVG data URI). This originated as a claude.ai Artifact (whose CSP blocked external hosts); the standalone `<!DOCTYPE>/<head>` wrapper and minimal reset at the top were added so it works as a plain page, but the self-contained rule still holds. There is intentionally no webfont — typography uses system-font stacks.

## Architecture (the parts that span the file)

The page is a sidebar table-of-contents (`nav.toc`) plus these `<section>`s in order: `#doctrine`, `#loadout`, `#cheese-toolkit`, `#power`, `#checklist`, `#bosses`. A small IntersectionObserver in the script highlights the active TOC link and auto-opens a `<details>` phase when its nav link is clicked.

### Design tokens & theming
Colors/fonts are CSS custom properties on `:root`. The palette is theme-aware three ways: base `:root`, a `@media (prefers-color-scheme: dark)` override, and explicit `:root[data-theme="light"|"dark"]` overrides (the latter are a no-op on Pages but harmless — they were for the Artifact host's theme toggle). Style components through the tokens, never hard-code colors. A status-color code is load-bearing: `--bleed`/`--rot`/`--frost`/`--sleep` drive the `.chip c-*`, `.tactic--*`, and legend classes; `--gold` is the single accent.

### The interactive checklist (the one non-obvious system)
`#checklist` is the core. It is built from `<details class="phase-group" id="ph-X">` phases (`ph-a`…`ph-l` for the base game, `ph-m1`…`ph-m5` for the DLC). Each step is:
```html
<li><label class="step"><input type="checkbox" id="<stepid>"> … <div class="step-title">…</div> … </label></li>
```
Step ids encode the phase: `a1`,`a2`… / `b1`… / … / `m1`…`m28` (DLC). The trailing `<script>` wires up four things off those ids:
1. **Persistence** — checkbox state saves to `localStorage` under `er-rl1-checklist-v1`; `refresh()` recomputes the overall `done/total` counter, the `.cl-fill` progress bar, and each phase's `.ph-badge` (`done/total`, turns gold at 100%). "Reset progress" clears it.
2. **Step links** — two objects key off `stepid`: `WIKI_URLS` (`stepid → Fextralife entry-page URL`, via the `FEX` base const) and `MAP_PINS` (`stepid → interactive-map query string` — the part after `Interactive+Map?`, via the `MAP = FEX + "Interactive+Map?"` base const). For each step the script (`mkStepLink` helper) injects up to two `.step-link` anchors (`target="_blank"`) into `.step-title`: a 📖 **wiki** link (the entry page) and a 📍 **map** link (a pin dropped on the exact spot). Each sits inside the `<label>` but, being an `<a>`, navigates without toggling the checkbox. A step absent from an object just gets no such link — a handful have a wiki link but no map pin (the item/boss page has no interactive-map deep-link).

**Editing invariant:** a step's checkbox `id`, its `WIKI_URLS` / `MAP_PINS` keys, and the phase's `ph-badge` "0/N" count must stay in sync, and nav `href="#ph-*"` must match a `phase-group` id. When adding/removing/renumbering steps, update all of these together, then run the sanity check above.

### Fextralife URL convention (for new step links)
**Wiki entry (`WIKI_URLS`):** page path = the exact wiki title with spaces → `+`; apostrophes, hyphens, and parentheses kept literally. **Commas are inconsistent** on Fextralife (kept on some pages, dropped on others) and some titles have quirks (e.g. `Bayle+The+Dread` capitalizes "The"; the boss vs. spirit-ash pages differ). Verify any new URL against the live wiki rather than assuming.

**Map pin (`MAP_PINS`):** every item/boss/location page carries a "See Elden Ring Map" / "Map Link" pointing at `Interactive+Map?id=NNN&code=mapX` (sometimes with `&lat=…&lng=…` to center the view). Copy that query string from the live page — don't guess the `id`. Map layers seen in this guide: `mapA` overworld · `mapB` underground (Siofra/Nokron/Ainsel/Mohgwyn) · `mapC` Ashen Leyndell · `mapD` the DLC Realm of Shadow. Pages that list several locations give several links — pick the one matching the guide's route.

## Content accuracy
Item locations, stat requirements, and boss tactics reflect the live game patch and are sourced from community wikis (primarily Fextralife). Treat gameplay claims as facts to verify against a current source, not to invent.
