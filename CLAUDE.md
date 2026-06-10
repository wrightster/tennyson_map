# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an interactive real estate lot map for **Tennyson — Phase One**, a residential subdivision with 20 lots (lots 1–18 + Lot A + Lot B). The primary deliverable is `tennyson-map.html` — a standalone HTML file that requires no build step, bundler, or framework.

**To preview:** run `python dev-server.py [port]` (default 8765, opens at `/tennyson-map.html`) and use that — it also enables the in-page edit mode's `POST /save` back to disk (whitelist: `ALLOWED_FILES` in `dev-server.py`). A plain `python -m http.server` works for read-only previewing. The HTML fetches its companion files via `fetch()` so a server is always required; `file://` will not work.

## Architecture

The project is split into four runtime files:

- **`tennyson-map.html`** — layout, page-chrome CSS, and all JavaScript
- **`tennyson-map.svg`** — static SVG geometry (roads, streams, easements, lot polygons, labels, background wash)
- **`tennyson-lots.csv`** — lot data (acreage, status, builder assignments, builder colors)
- **`map-styles.css`** — shared styles for SVG map elements (lots, labels, roads, overlays)

### `tennyson-map.html` structure:

1. **CSS**: page chrome (header, toolbar, info panel, stats bar, export modal), design tokens in `:root` (`--slate-*`, `--sand-*`, `--font-display` serif / `--font-body`), mobile layout
2. **HTML body**: Header, toolbar buttons, `#map-svg-container` placeholder, info panel, stats bar
3. **JavaScript**: All interactivity — `loadResources()`, `parseCSV()`, `rebuildFromCSV()`, `populateLabels()`, `rebuildBuilderLegend()`, `init()`, lot selection, status management, pan/zoom, builder view, plat/topo overlay controls, edit mode, export/print
4. **External script**: JWRG contact form from `https://office.jwrgnc.com/js/forms.js`

### Load sequence:

1. `loadResources()` fetches `tennyson-map.svg`, `tennyson-lots.csv`, and `map-styles.css` in parallel via `fetch()`
2. SVG text is injected into `#map-svg-container` via `innerHTML`; CSS text into `#map-styles`
3. CSV rows are parsed → `LOT_DATA`, `BUILDER_DATA`, `statuses`, `builderByLot` are built; each lot polygon gets a per-element `--lot-fill` CSS variable (its builder's color)
4. Lot labels populated from CSV (ring badge + number + acreage + SOLD mark), builder legend HTML rebuilt
5. Status classes applied to lot polygons, `init()` called

### Data files (served alongside HTML):

- `tennyson-lots.csv` — lot data in CSV format. **Edit this to update statuses, acreage, builders, and builder colors.**
- `tennyson-map.svg` — full SVG geometry. Edit this for geometry changes (roads, easements, lot polygons).
- `map-styles.css` — edit this for visual styling of map elements (also used by export, which strips edit-mode rules).

## Design system (poster-derived)

The map's look replicates `assets/Tennyson_MapPoster_barebones.svg` (the printed poster design — keep it as the visual reference):

- **Lot fills are builder colors** by default, set via per-lot `--lot-fill` in `rebuildFromCSV()` and consumed by `.lot` in `map-styles.css`. Lots with no builder get sage `#9fa995`. Palette (stored in the CSV `builder_color`/`builder_border` columns): Colebrooke tan `#b3a186`, Thadd Roberts dark green `#4b675e`, JW2 gray-sage `#899988`, Lilium rust `#885138`, Pinnacle taupe `#887b66`.
- **Ink**: lot boundaries, road edges, label text and ring badges use near-black `#231f20`. Roads are bone `#cfceca`. Canvas is paper white `#fbfaf6` with a radial sage wash (`#bg-wash` circle + `#bg-wash-gradient` in the SVG) under the subdivision.
- **Labels**: serif (`minion-pro`/Georgia) lot number inside a `circle.lot-label-badge` ring, acreage as `X.XXXX AC` below, and a red (`#ed1c24`) serif `SOLD` text on sold lots — all created/synced by `populateLabels()` and `updateSoldLabel()`.
- Selected lots invert the badge (dark fill, white text); hover lightens the fill via `color-mix`.

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

Edit the `status` column in `tennyson-lots.csv` to `available`, `sold`, or `reserved`, then reload the page. (The old embedded `lots-data` CSV block in the HTML no longer exists — the external CSV is the single source of truth.) Sold lots automatically get the red `SOLD` mark in their label group.

## Adding/Changing Builders

Edit the builder columns in `tennyson-lots.csv`. Builder info is repeated per lot row; JS deduplicates by `builder_name`. Lot fills, the builder legend, and the builder-view border bands (`createBuilderDots()`) are all generated dynamically from the rebuilt `BUILDER_DATA`, so a builder's `builder_color` drives everything.
