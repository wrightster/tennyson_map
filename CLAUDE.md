# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an interactive real estate lot map for **Tennyson — Phase One**, a residential subdivision with 20 lots (lots 1–18 + Lot A + Lot B). The primary deliverable is `tennyson-map.html` — a standalone HTML file that requires no build step, bundler, or framework.

**To preview:** run `python dev-server.py [port]` (default 8765, opens at `/tennyson-map.html`) and use that — it also enables the in-page edit mode's `POST /save` back to disk (whitelist: `ALLOWED_FILES` in `dev-server.py`). A plain `python -m http.server` works for read-only previewing. The HTML fetches its companion files via `fetch()` so a server is always required; `file://` will not work.

**This repo is the authoring source, not the deploy.** As of 2026-07-16 the *running* map lives in the Tennyson site repo (`neighborhoods/Tennyson-Website`), served at **`tennyson.jwrgnc.com/map`** and embedded same-origin on its home page. It no longer loads from `tools.jwrgnc.com/tennyson`. Keep authoring here (CAD, plat tracing, the tree-shape editor, `dev-server.py`), then **copy the runtime files over** — `tennyson-map.html` → the site's `src/map/`, and `tennyson-map.svg` / `map-styles.css` / `tennyson-trees.json` / `plat_full.png` / `svg/*` → its `public/map/`. The site's copy of the HTML carries one site-specific edit (a `<base href="/map/">`); see that repo's `CLAUDE.md`. Lot **status/builder** come live from the office there via a merge endpoint, so `tennyson-lots.csv` only supplies the map's own columns (4-decimal acreage + poster colors).

**Embed mode:** append `?embed` (e.g. `tennyson-map.html?embed=1`) to render the map for iframe embedding on the Tennyson site. A `<script>` in `<head>` reads the param and adds an `embed` class to `<html>` before paint (no flash of chrome); the `html.embed …` CSS block plus the JS `IS_EMBED` flag drive the differences. In embed mode:

- **Hidden:** header, stats bar, zoom controls, JWRG watermark. The `html`/`body` background drops to transparent so the host page shows through (the embedding `<iframe>` must have no background of its own).
- **Kept:** the overlay toggles (Plat / Topo / Easements / Trees) and the builder/status legend — so viewers can toggle layers.
- **Pan & zoom disabled** — `initPanZoom()` no-ops under `IS_EMBED`, so no wheel/drag/touch handlers are attached; the map shows the fixed default view (lots stay clickable).
- **Lot info is a floating card** pinned top-right, shown only while a lot is selected (the `.has-selection` class on `.info-panel`, toggled in `openPanel()`/deselect paths). The lead-gen form is dropped in this card.
- **Double-click a lot → its page on the Tennyson site** (`https://tennyson.jwrgnc.com/lots/{lot_id}`, via `openLotPage()`). Lot ids (`1`–`18`, `a`, `b`) match the site's `/lots/[id]` route; when framed it navigates `window.top` to break out of the iframe.

None of this affects the standalone (non-embed) view, which keeps the full chrome, sidebar panel, and pan/zoom.

## Architecture

The project is split into four runtime files:

- **`tennyson-map.html`** — layout, page-chrome CSS, and all JavaScript
- **`tennyson-map.svg`** — static SVG geometry (roads, streams, easements, lot polygons, labels)
- **`tennyson-lots.csv`** — lot data (acreage, status, builder assignments, builder colors)
- **`map-styles.css`** — shared styles for SVG map elements (lots, labels, roads, overlays)

### `tennyson-map.html` structure:

1. **CSS**: page chrome (header, toolbar, info panel, stats bar, export modal), design tokens in `:root` (`--slate-*`, `--sand-*`, `--font-display` serif / `--font-body`), mobile layout
2. **HTML body**: Header, toolbar buttons, `#map-svg-container` placeholder, info panel, stats bar
3. **JavaScript**: All interactivity — `loadResources()`, `parseCSV()`, `rebuildFromCSV()`, `populateLabels()`, `rebuildBuilderLegend()`, `init()`, lot selection, status management, pan/zoom, builder view, plat/topo/tree overlay controls, tree placement system, edit mode, export/print
4. **External script**: JWRG contact form from `https://office.jwrgnc.com/js/forms.js`

### Load sequence:

1. `loadResources()` fetches `tennyson-map.svg`, `tennyson-lots.csv`, and `map-styles.css` in parallel via `fetch()`
2. SVG text is injected into `#map-svg-container` via `innerHTML`; CSS text into `#map-styles`
3. CSV rows are parsed → `LOT_DATA`, `BUILDER_DATA`, `statuses`, `builderByLot` are built; each lot polygon gets a per-element `--lot-fill` CSS variable (its builder's color)
4. Lot labels populated from CSV (ring badge + number + acreage + SOLD mark), builder legend HTML rebuilt
5. Status classes applied to lot polygons, `init()` called
6. Tree layer built: `injectTreeLayer()` fetches `svg/tree-{1..3}.svg` into `<symbol>`s and `rebuildTreeLayer()` scatters `<use>` elements from the user-drawn tree shapes (loaded from `tennyson-trees.json` / localStorage, most-recent-wins)

### Data files (served alongside HTML):

- `tennyson-lots.csv` — lot data in CSV format. **Edit this to update statuses, acreage, builders, and builder colors.**
- `tennyson-map.svg` — full SVG geometry. Edit this for geometry changes (roads, easements, lot polygons).
- `map-styles.css` — edit this for visual styling of map elements (also used by export, which strips edit-mode rules).
- `tennyson-trees.json` — user-drawn tree shapes (`{savedAt, polygons, lines}`), written by the shape editor via `POST /save`.

The in-page **Save to Files** button (`saveToFiles()`) serializes a *sanitized clone* of the SVG: runtime-generated layers (`#tree-layer`, `#lot-stroke-layer`, builder dots/masks, editor overlays), CSV-derived label text, per-lot `--lot-fill` styles, and the pan/zoom viewBox are stripped before writing, so `tennyson-map.svg` stays pure geometry. Keep it that way — baked runtime layers once bloated the file from 73KB to 336KB.

## Design system (poster-derived)

The map's look derives from `assets/Tennyson_MapPoster_barebones.svg` (the printed poster design — still the reference for lot fills, inks, and labels), but the on-screen background has diverged: the poster's paper canvas was dropped in favor of the site's dark slate.

- **Lot fills are builder colors** by default, set via per-lot `--lot-fill` in `rebuildFromCSV()` and consumed by `.lot` in `map-styles.css`. Lots with no builder get sage `#9fa995`. Palette (stored in the CSV `builder_color`/`builder_border` columns): Colebrooke tan `#b3a186`, Thadd Roberts dark green `#4b675e`, JW2 gray-sage `#899988`, Lilium rust `#885138`, Pinnacle taupe `#887b66`.
- **Ink**: lot boundaries, road edges, lot label text and ring badges use near-black `#231f20`. Roads are bone `#cfceca`. Area labels (`.area-label`: OPEN SPACE / RECREATION AREA) are white. Easements (`.esmt`) are red dashed strokes, no fill.
- **Background**: the map floats directly on the page's slate gradient (`body` background, `--slate-800`→`--slate-900`); `.map-wrap` is transparent. The old paper-white canvas and radial sage wash are gone from the screen view — the canvas rect (`rect.st4`, `fill: none` via the SVG's internal stylesheet) is kept only because export looks it up and inlines a white/cream fill (or removes it for transparent export). Don't delete it.
- **Builders and Lot Status are always on** — they are not toggleable overlays. The split is: **lot polygon fill = builder color** (`--lot-fill`), **lot-number badge fill = status color**. Both legend columns always show. (The `toggleLotStatusOverlay`/`toggleBuilderOverlay` functions still exist but have no callers.)
- **Labels**: serif (`minion-pro`/Georgia) lot number inside a `circle.lot-label-badge`, acreage as `X.XX AC` below (rounded to the hundredth for display; the CSV keeps 4 decimals), and a red (`#ed1c24`) serif `SOLD` text on sold lots — all created/synced by `populateLabels()` and `updateSoldLabel()`. The badge is **filled with the lot's status color** via a status class on the label group (`.lot-label-group.available/.sold/.reserved` → `#cce8c2` / `#e8c2c2` / `#f5e8b0`, matching the legend swatches). Current geometry: badge `r=18 @ cy=-8`, number `22.5px`, acreage `11.25px @ y=22`, SOLD `12px @ y=-33`. The group `transform` is rewritten as a pure `translate()` by `populateLabels()` and the label drag — so **never scale the label via that transform**; change the geometry/font sizes instead.
- Selected lots invert the badge (dark fill, white text); hover lightens the fill via `color-mix`.
- **Trees**: top-down canopy art in dark green, placed wherever tree shapes have been drawn in edit mode (`.tree-layer` in `map-styles.css`). Each tree gets a **subtle per-tree shade** from `TREE_TINTS` (a small palette around the base `#4a6b35`), picked deterministically from its position and applied as an **inline style** on the `<use>` (a `fill` attribute would lose to the `.tree-layer use { fill }` rule). The symbol art (`svg/tree-N.svg`) has its canopy path pre-scaled into the 14×14 tile with **no `<g transform>`** (some renderers don't apply it) and its shadow **baked in** — never reintroduce a CSS `drop-shadow`/`filter` on `.tree-layer use`; a per-`<use>` filter on ~1,700 trees cripples pan/zoom performance.
- **Chrome layout**: zoom top-left, JWRG watermark top-right, the two-column legend (Builders | Lot Status, bottom-justified) bottom-left, and the overlay toggles as a stacked "Overlays" list bottom-right. Easements are **off on load** — the SVG ships `#easement-overlay` visible, so `loadResources()` syncs it to `easementsVisible`.

## Tree placement system

All trees come from user-drawn shapes — there is no tree geometry in `tennyson-map.svg` and no automatic edge-based placement:

- `CUSTOM_TREE_POLYGONS` (interiors filled with trees on a jittered hex grid; `fillSpacing`/`fillScatter` in `TREE_CONFIG`) and `CUSTOM_TREE_LINES` (trees along every segment) are the only placement sources. They persist in **`tennyson-trees.json`** (written via `POST /save` whenever a shape is edited, so the design is shared and versioned) and in localStorage under `tennyson-custom-trees` (fallback for edits made without the dev server). Each save carries a `savedAt` timestamp; load picks the newer of the two.
- `TREE_CONFIG` (in the HTML JS) holds the tuning knobs: spacing, scatter, scale/variance, fill grid density, deterministic `seed`, road falloff distances, etc.
- `rebuildTreeLayer()` turns the shapes into seeded-random `<use href="#tree-symbol-N">` elements. Trees that land inside `.road-interior` polygons are pushed off (`pushOffRoad()`); trees near road segments (`.road-edge` polylines + the `lawrence-road-path`/`tennyson-court-path` centerlines) are scaled down.
- The three symbol variants are fetched from `svg/tree-{1..3}.svg` and injected into the hidden `<svg id="svg-filter-defs">` defs block at the bottom of the HTML (kept there, not in the geometry SVG, so they survive SVG re-exports). Export (`_buildExportSVG()`) copies them into the exported SVG's own defs and strips the shape-editor overlay.
- The **Trees** toolbar button (`toggleTreeOverlay()`) shows/hides the layer; trees are on by default and "Reset View" re-shows them.
- In edit mode, the **🌲 Tree Shapes** button toggles the shape editor: **Draw Tree Polygon** / **Draw Tree Line** start a drawing (click to place points, double-click/Enter finishes, Esc cancels; clicking a polygon's first point closes it). Click a shape to select it, drag its vertex anchors, double-click an edge to insert a vertex, double-right-click a vertex to remove it, Delete removes the whole shape.

## Key Data Structures

- **`LOT_DATA`**: Array of `{id, number, acres}` for all 20 lots (built from CSV)
- **`BUILDER_DATA`**: Array of builder objects with `{name, short, contact, address, phone, email, lots[], color, border}` (built from CSV; deduplicated by builder name)
- **`statuses`**: Object mapping lot ID → `'available'|'sold'|'reserved'` (built from CSV)
- **`builderByLot`**: Object mapping lot ID → builder object

## Assets

- `tennyson-map.svg` — standalone SVG with all map geometry; lot polygons have `data-lot` and `data-lot-id` attributes, label text is populated by JS from CSV
- `tennyson-lots.csv` — one row per lot; columns: `lot_id, lot_number, acres, status, builder_name, builder_short, builder_contact, builder_address, builder_phone, builder_email, builder_color, builder_border`
- `assets/Tennyson_MapPoster_barebones.svg` — printed poster design the map's styling replicates (reference only, not loaded at runtime)
- `plat_full.png` — 3600×2700px plat PDF scan, used as a togglable overlay
- `svg/Tennyson_TopoContour.svg` — topo contour overlay
- `svg/JWRG_Positive.svg` — JWRG watermark logo
- `svg/tree-1.svg` / `tree-2.svg` / `tree-3.svg` — tree symbol variants (14×14 viewBox) used by the tree placement system. Top-down canopy art baked from 64×64 source drawings (scale `0.1875` + offset `(1,1)`, no group transform); the fill is left off so each inherits the tree-layer color/tint.
- `tennysun.dwg` / `dxf_output_new/tennysun.dxf` — source CAD files (not used at runtime)
- `example.html` — standalone static map variant (Cannady Mill Road / Blackwell Builders)

## SVG Coordinate System

The map canvas rect is `0 0 1240 1000`; the viewBox is slightly larger (pan/zoom mutates it at runtime, so don't rely on its serialized value). All lot polygon `points` are in this coordinate space. The overlay alignment matrices map external image pixel coordinates to these SVG units:

- **Plat PNG**: `matrix(0.3768, 0, 0, 0.3768, -119, 26)` — constants `BASE_SX/SY/TX/TY` in JS
- **Topo SVG**: `matrix(1.8792, 0, 0, 1.8792, -70.62, -128.57)` — constants `TOPO_BASE_*` in JS

## Pan/Zoom Implementation

Zoom and pan modify the SVG `viewBox` attribute rather than CSS transforms. State: `scale`, `panX`, `panY` globals. Both mouse wheel+drag and touch pinch-zoom/pan are supported. The "Reset View" button resets all view state plus all filter/overlay toggles.

## Mobile Layout

At `max-width: 768px`, the info panel becomes a bottom sheet (slides up from bottom) instead of a right sidebar. Touch events handle panning and pinch-to-zoom.

## Modifying Lot Statuses

**The JWRG office is the source of truth for lot status.** As of the 2026-07-16 migration, the *deployed* map (in the Tennyson site) no longer reads status from this CSV at all — the site's `/map/tennyson-lots.csv` endpoint fetches the office live and overrides the `status` + builder columns, so **it cannot drift**. This CSV's status only affects a **local preview** here via `dev-server.py`; keep it roughly current, but the office is what ships. It drifted once already (seven lots wrong, corrected 2026-07-13), which is why the site merges live. The office serves the current statuses at `GET https://office.jwrgnc.com/api/v1/neighborhoods/tennyson/lots`; its `available`/`reserved`/`under_contract`/`not_released`/`sold` collapse to the three this map renders (`under_contract` and `not_released` → `reserved`, `common_area` lots are dropped).

Copy **status only**. The other columns are the map's own: the acreage here carries the plat's 4-decimal precision (the office rounds to 3), and `builder_color`/`builder_border` are the muted poster palette, deliberately different from the office's brand colors.

Mechanically: edit the `status` column in `tennyson-lots.csv` to `available`, `sold`, or `reserved`, then reload the page. (The old embedded `lots-data` CSV block in the HTML no longer exists — the external CSV is what the map reads.) Sold lots automatically get the red `SOLD` mark in their label group.

## Adding/Changing Builders

Edit the builder columns in `tennyson-lots.csv`. Builder info is repeated per lot row; JS deduplicates by `builder_name`. Lot fills, the builder legend, and the builder-view border bands (`createBuilderDots()`) are all generated dynamically from the rebuilt `BUILDER_DATA`, so a builder's `builder_color` drives everything.
