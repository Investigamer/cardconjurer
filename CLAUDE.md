# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Card Conjurer — a browser-based Magic: The Gathering custom card creator. It is a **static site**: plain HTML/CSS/vanilla JS with no build step, no bundler, no package.json, and no test suite. Everything runs client-side; there is no backend/API of its own (art can be fetched from Scryfall or uploaded locally).

## Running it locally

No build step is required — just serve the repo root as static files.

```bash
make start
```
Runs the Docker/nginx target and serves on http://localhost:4242/ (see `Makefile`, `Dockerfile`, `app.conf`).

Alternatives (see `README.md`):
- Run the prebuilt `launcher-macos` / `launcher-linux` / `launcher.exe` binary in the repo root (built from `launcher.py`, a trivial Python `http.server` on port 8080).
- `python3 launcher.py` directly.
- Any static file server (WAMP/XAMPP/`python -m http.server`) pointed at the repo root also works, since routing is just plain files (see the nginx `try_files` rules in `app.conf`).

There is no lint/test/build command — verify changes by loading the page in a browser and exercising the UI directly.

## High-level architecture

### Multi-page app stitched together with htmx

`index.html` is the shell: header, hamburger nav menu, and an empty `#content` div. Each nav entry (`creator/index.html`, `print/index.html`, `askurza/index.html`, `phyrexian/index.html`, `about/index.html`, `gallery/index.html`, `theme/index.html`, `legal/index.html`) is a **standalone HTML page** that also gets htmx-swapped into `#content` (`hx-get="..." hx-target="#content"`) when navigated to from the menu. Because each of these pages is fully standalone, every one can also be loaded directly by URL (e.g. visiting `/creator/` works on its own, not just via the nav swap). Keep this dual-use in mind when editing shared markup/scripts inside a page: it must work both injected into `#content` and as a top-level document.

`js/main-1.js` holds the shell-level behavior: hamburger menu toggle, notifications, drag-and-drop file upload wiring, and the `doCreate` event that htmx fires after swapping in the creator.

### The card creator (the core of the app)

- `creator/index.html` — the creator UI: all the tab panels (Frame/Text/Art/Set Symbol/Watermark/Bottom Info/Save-Load/etc.) as one large form-heavy document.
- `js/creator-23.js` (~200KB, versioned filename — bump the suffix, e.g. `creator-24.js`, when making a breaking change and update the `<script>` reference in `creator/index.html`) — the entire creator engine in one file. It is organized as a large set of top-level functions operating on shared global state rather than modules/classes. Key globals:
  - `card` — the in-memory model of the card being edited (art position/zoom, set symbol, watermark, frame list, mana symbols, version, etc.).
  - `canvasList`, `previewCanvas`/`previewContext` — the layered `<canvas>` stack that gets composited and drawn to the visible preview.
  - `availableFrames`, `selectedFrame*` — the currently loaded frame pack and selection state.
  - Card rendering is a layered canvas pipeline: frames/masks are loaded as images onto individual canvases (`drawFrames`, `addFrame`), text is measured/wrapped and painted manually (`writeText`, `renderTextJustified`, no DOM text), and `drawCard()` composites everything to the final canvas for preview/export (`downloadCard`).
  - Auto-frame selection logic (`autoFrame`, `autoM15Frame`, `autoBorderlessFrame`, `autoUBFrame`, etc.) picks appropriate frame pieces per-letter (color pips W/U/B/R/G/multicolor/artifact/land) based on mana cost/color/type — see the `make*FrameByLetter` functions.
  - Scryfall integration (`fetchScryfallData`, `fetchScryfallCardByID`, `scryfallCardFromText`, `processScryfallCard`) imports real card data/art to seed a custom card.
  - Save/load is entirely client-side via `localStorage` (`saveCard`, `loadCard`, `deleteCard`, `downloadSavedCards`/`uploadSavedCards` for JSON export/import of saved cards). There is no server persistence.
- `js/frameSearch.js` — autocomplete search box for jumping to a frame by name.
- `js/frames/*.js` — **data files**, not general code. Each `group*.js` file is a call to `loadFramePacks([...])` / `loadFramePack([...])` describing one dropdown's worth of frame-pack options (name → asset key), and `manaSymbols*.js` / `pack*.js` files similarly register mana symbol sets and frame packs. Adding a new frame set/style means adding a data file here (following the existing naming/registration pattern) and the corresponding image assets under `img/frames/`, rather than writing new logic in `creator-23.js`.
- `img/frames/` — the actual frame/mask PNG/SVG assets referenced by the frame pack data files, organized in deeply nested subfolders mirroring set/style names.

### Other standalone tools (each self-contained, own JS file, no shared framework)

- `print/` — lays out saved cards onto printable sheets with cutting guides (`print.js`, uses `jspdf` for PDF export).
- `askurza/` — a planeswalker-ability list generator (`askUrza.js`).
- `phyrexian/` — English↔Phyrexian text transliterator (`phyrexian.js`), backed by `data/fonts/phyrexian.ttf`.
- `converter/` — a card image converter.
- `theme/` — `theme/index.html` + `js/themeEditor.js` + `js/themes.js`, for customizing the site's own color theme (stored client-side, applied via CSS variables — see `css/`).
- `gallery/` — static showcase page.
- `data/scripts/` (`localCardStorage.js`, `localStorage.js`, `animations.js`, etc.) is legacy/unreferenced code kept in the repo but not loaded by any current page — don't assume it runs; check for an actual `<script src="data/scripts/...">` reference before relying on it.

### Local art override

To avoid bloating localStorage-saved cards with base64 image data, users can drop image files into `local_art/` and reference them by filename in the Art tab's "Via URL" field instead of uploading (see `README.md`).

## Deployment

`.github/workflows/publish.yaml` syncs the repo straight to an S3 bucket (`cardconjurer.app`) on every push to `master` via `aws s3 sync` — there is no build/compile step in CI, so whatever is committed is what ships. Be careful with changes on `master`.
