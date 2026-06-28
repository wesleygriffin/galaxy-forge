# CLAUDE.md

Guidance for working in this repository with Claude Code.

## What this is

**Galaxy Forge** is a browser-based map generator for *Twilight Imperium 4th
Edition* (TI4). It is a **single self-hosted `index.html` file** with all CSS
and JavaScript inline — there is no build step, no bundler, no package manager,
and no server. Open `index.html` in a browser and it runs.

**Mobile (iOS Safari) is the primary design target.** Desktop is supported, but
when a trade-off is required, the mobile experience wins.

## Repository layout

```
index.html        The entire app: HTML + inline <style> + one inline <script>.
assets/           Original SVG primitives (circle, hexagon-flat, hexagon-pointy).
                  Geometric shapes only — safe to use and extend.
.gitignore        Just .DS_Store.
CLAUDE.md         This file.
```

There is intentionally no `tiles/` directory and no third-party art. See
**Art & licensing** below.

## Running and testing

There is no test suite and no build. To verify a change:

1. Open `index.html` in a browser (or serve the folder with any static server,
   e.g. `python3 -m http.server`).
2. **Validate JS syntax before committing.** The script is inline, so a syntax
   error silently breaks the whole app. Extract the inline `<script>` and run it
   through Node's parser:

   ```bash
   python3 - <<'PY'
   import re
   html = open('index.html').read()
   scripts = re.findall(r'<script(?![^>]*\bsrc=)[^>]*>(.*?)</script>', html, re.DOTALL)
   open('/tmp/extracted.js','w').write('\n'.join(scripts))
   PY
   node -e "new Function(require('fs').readFileSync('/tmp/extracted.js','utf8')); console.log('JS syntax OK');"
   ```

## Versioning — required on every code commit

`const APP_VERSION = "x.y.z";` lives near the top of the inline script (search
for `APP_VERSION`). **Bump the patch number (z) on every commit that changes
code.** Minor (y) and major (x) bumps happen only on explicit instruction. The
version renders in the app header and the help/settings sheet, so it is a quick
visual confirmation that a deploy actually took.

## Commit discipline

- **One logical change per commit.** Keep diffs focused.
- Include the `APP_VERSION` bump in the same commit as the change it describes.
- Write a real commit body explaining the *why*, not just the *what*.
- Commits go directly to `main`.

## Code architecture (inside `index.html`)

The inline script is organized into clearly commented sections (search for the
banner comments `/* ===`). In rough order:

- **TILE DATA** — `const SYSTEMS = {...}`: the full system/tile database keyed by
  tile ID. Each entry has type, planets (name/res/inf/tech), wormholes,
  anomalies, `back` color (blue/red), and `expansion`.
- **MAP LAYOUTS** — `const LAYOUTS = {...}` and `ring(n)`: axial `(q, r)`
  coordinates for each player count, including home positions and Mecatol Rex.
- **APP STATE** — `state` object: current map, locked hexes, selected hex,
  placement style, weighting, expansions toggles, view transform, etc.
- **UTILITIES** — hex math. **Axial coordinates, flat-top orientation.**
  `hexKey`, `parseKey`, `axialToPixel`, `hexPoints`, `shuffle`.
- **TILE POOL & WEIGHTING** — which tiles are eligible and how they're weighted
  (`random`, `resource`, `influence`, `custom`).
- **MAP GENERATION** — `generateMap`, `computePlacementOrder`,
  `placeWithConstraints`. Honors locked hexes and reserved home slots.
- **RENDERING** — `render`, `renderHex`, `renderTileContent`, `renderPlanets`,
  `getHexFill`. Everything is drawn as SVG with our own glyphs (no images).
- **TILE MENU**, **UNUSED TILES DRAWER**, **DRAG AND DROP**, **VIEW CONTROLS**
  (pan/zoom/pinch), **SHARE / EXPORT**, **UI SYNC**, **EVENT WIRING**,
  **SYSTEM DATA EXPORT**, **HIGH-RES MAP RENDER (PNG)**, **SIDEBAR RESIZE**,
  **INITIALIZATION** (`init`).

### Hex geometry

Axial coordinate system, **flat-top** hex orientation (vertices on left/right,
flat edges on top/bottom — matching the official TI4 board). `HEX_SIZE` is the
center-to-vertex radius. Don't reintroduce a pointy-top `-30°` vertex offset;
that was a past bug.

## Input handling — read before touching interaction code

All pointer interaction uses the **`PointerEvent`** API, not HTML5
drag-and-drop and not separate mouse/touch handlers.

- iOS Safari does **not** support HTML5 drag-and-drop. Use Pointer events for
  hex-to-hex swaps and the unused-tiles drawer.
- Mouse and touchpad both report `pointerType: "mouse"` and cannot be
  distinguished — unified handling is correct and intentional.
- **Double-tap-to-lock (mobile):** first tap on a tile is deferred via
  `setTimeout(showTileMenu, DOUBLE_TAP_MS + 20)` so a quick second tap can be
  detected as a lock toggle instead of opening the menu. `clearPendingMenu()` is
  centralized inside `hideTileMenu`, so every cancel path (drag, pinch, lock,
  regenerate) automatically cancels a pending menu open. Preserve this pattern.

## CSS gotchas (mobile)

- Use `100dvh`, **not** `100vh`, for full-height layout on iOS.
- Prefer **Flexbox** over CSS Grid for the main app layout — iOS Safari's
  auto-row grid sizing is unreliable. The sidebar is `flex: 0 0 auto`, the board
  area `flex: 1 1 auto`.
- **CSS source order matters.** Critical mobile overrides must be re-declared in
  a final `@media (max-width: ...)` block at the *end* of the stylesheet so they
  win the cascade against later-appearing desktop rules.
- Harden draggable/tappable elements with `-webkit-user-select: none`,
  `-webkit-touch-callout: none`, and `touch-action: none`.

## Icons

UI icons are inline SVGs in the **Tabler Icons** style (MIT licensed) — hammer
(the forge button), wrench (settings), padlock (lock badge), link, download,
etc. **Prefer flat SVG icons over emoji** for visual consistency. When adding an
icon, match the existing stroke-based Tabler aesthetic.

## Art & licensing — important

This project must stay **free of infringing artwork.**

- The official TI4 tile artwork is owned by Fantasy Flight Games / Asmodee.
  **Do not** mirror, hotlink, embed, or trace it. A previous version mirrored
  tile images from a fan repo and traced some SVGs from them; all of that was
  removed.
- Systems are rendered entirely from **our own SVG glyphs** in
  `renderTileContent` / `renderPlanets` (text labels, stat numbers, wormhole
  circles, anomaly markers, tech-color dots). This is the only rendering path.
- `assets/` may contain **original geometric primitives only** (plain hexagons,
  circles). Anything derived from copyrighted game art does not belong here.
- Tabler Icons (MIT) are fine for UI chrome.

If you're ever unsure whether an asset is safe to include, leave it out and ask.

## Git / tooling notes

- Before an edit session, orient yourself:
  `git status && git log origin/main..HEAD --oneline`.
- When a commit combines moves, renames, and new files, stage with
  `git add -A`. Use `git mv` to preserve history on renames/moves.
- After pushing, you can confirm the remote actually advanced by comparing
  `git ls-remote origin main` against your local `HEAD` hash (piped `git push`
  output can be misleading).

## Conventions summary

- Single-file app; keep CSS and JS inline in `index.html`.
- Mobile-first, iOS-primary.
- Pointer Events for all input.
- Original SVG rendering only — no third-party art.
- Bump `APP_VERSION` patch on every code commit; one logical change per commit.
