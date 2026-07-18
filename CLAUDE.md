# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained static page (`index.html`) — an interactive Elden Ring "Rune Level 1 / no level-up" playthrough guide (base game + Shadow of the Erdtree). There is **no build system, no dependencies, and no tests**. The entire site is one HTML file with an inline `<style>` block and one inline `<script>`; `README.md` and `.nojekyll` are the only other tracked files.

## Commands

- **Preview locally:** open `index.html` directly, or serve it — `python3 -m http.server` in the repo root, then visit `http://localhost:8000/`. Use a server (not `file://`) when testing checklist progress, since it relies on `localStorage`.
- **Deploy:** commit to `main` and push. GitHub Pages serves `main` / root and rebuilds automatically (live at https://marano.github.io/elden-ring-level-1-guide/). `.nojekyll` disables Jekyll so the raw file is served as-is.
- **Structural sanity check** (there's no test runner; this is the substitute after edits):
  ```sh
  grep -c '<section' index.html; grep -c '</section>' index.html        # must match
  grep -c '<details' index.html; grep -c '</details>' index.html        # must match
  # every map-pin key must point at a real step id:
  for k in $(grep -oE '"[a-z]+[0-9]+": FEX' index.html | grep -oE '[a-z]+[0-9]+'); do
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
2. **Map pins** — a `MAP_URLS` object maps `stepid → Fextralife URL` (via a `FEX` base const). The script injects a 📍 `.map-link` (`target="_blank"`) into that step's `.step-title`. The link sits inside the `<label>` but, being an `<a>`, navigates without toggling the checkbox. A step with no `MAP_URLS` entry simply gets no pin.

**Editing invariant:** a step's checkbox `id`, its `MAP_URLS` key, and the phase's `ph-badge` "0/N" count must stay in sync, and nav `href="#ph-*"` must match a `phase-group` id. When adding/removing/renumbering steps, update all of these together, then run the sanity check above.

### Fextralife URL convention (for new map pins)
Page path = the exact wiki title with spaces → `+`; apostrophes, hyphens, and parentheses kept literally. **Commas are inconsistent** on Fextralife (kept on some pages, dropped on others) and some titles have quirks (e.g. `Bayle+The+Dread` capitalizes "The"; the boss vs. spirit-ash pages differ). Verify any new URL against the live wiki rather than assuming.

## Content accuracy
Item locations, stat requirements, and boss tactics reflect the live game patch and are sourced from community wikis (primarily Fextralife). Treat gameplay claims as facts to verify against a current source, not to invent.
