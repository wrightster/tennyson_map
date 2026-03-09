# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a single-file interactive real estate lot map for **Tennyson — Phase One**, a residential subdivision with 18 lots. The primary deliverable is `tennyson-map.html` — a standalone HTML file that requires no build step, bundler, or server.

To preview: open `tennyson-map.html` directly in a browser (or serve via any static file server).

## Architecture

Everything lives in `tennyson-map.html` — CSS, SVG map, and JavaScript are all inline. There is no separate JS or CSS file. The file structure:

1. **CSS** (lines 7–543): Styles for layout, lot states (available/sold/reserved), overlays, mobile responsive layout
2. **HTML body** (lines 545–777): Header, toolbar buttons, SVG map, info panel, stats bar
3. **Inline SVG map** (lines 616–750): Hand-placed polygon coordinates for roads, streams, easements, 18 lots, and labels — all in SVG user-space (~1200×960 units)
4. **JavaScript** (lines 778–1257): All interactivity — lot selection, status management, pan/zoom, builder view, plat/topo overlay controls
5. **External script** (line 1258–1262): JWRG contact form loaded from `https://office.jwrgnc.com/js/forms.js`

## Key Data Structures

- **`LOT_DATA`**: Array of `{id, acres}` for all 18 lots
- **`BUILDER_DATA`**: Array of builder objects with `{name, short, contact, address, phone, email, lots[], color, border}` — maps builders to their assigned lot IDs
- **`statuses`**: Object mapping lot ID → `'available'|'sold'|'reserved'` (initialized in JS; lots 1 and 9 are `sold`, lot 5 is `reserved` by default)

## Assets

- `plat_full.png` — 3600×2700px plat PDF scan, used as a togglable overlay
- `svg/Tennyson_TopoContour.svg` — topo contour overlay
- `svg/JWRG_Positive.svg` — JWRG watermark logo
- `tennysun.dwg` / `dxf_output_new/tennysun.dxf` — source CAD files (not used at runtime)
- `example.html` — standalone static map variant (Cannady Mill Road / Blackwell Builders)

## SVG Coordinate System

The SVG viewBox is `"-20 -20 1240 1000"`. All lot polygon `points` are in this coordinate space. The overlay alignment matrices map external image pixel coordinates to these SVG units:

- **Plat PNG**: `matrix(0.3768, 0, 0, 0.3768, -119, 26)` — constants `BASE_SX/SY/TX/TY` in JS
- **Topo SVG**: `matrix(1.8792, 0, 0, 1.8792, -70.62, -128.57)` — constants `TOPO_BASE_*` in JS

## Pan/Zoom Implementation

Zoom and pan modify the SVG `viewBox` attribute rather than CSS transforms. State: `scale`, `panX`, `panY` globals. Both mouse wheel+drag and touch pinch-zoom/pan are supported. The "Reset View" button resets all view state plus all filter/overlay toggles.

## Mobile Layout

At `max-width: 768px`, the info panel becomes a bottom sheet (slides up from bottom) instead of a right sidebar. Touch events handle panning and pinch-to-zoom.

## Modifying Lot Statuses

To change which lots start as sold/reserved, edit the `statuses` object initialization at lines 793–796. Status classes (`available`, `sold`, `reserved`) on the lot `<polygon>` elements are cosmetic initial values — JS overrides them on `init()`.

## Adding/Changing Builders

Edit the `BUILDER_DATA` array. Each builder entry needs a `lots` array with the lot IDs they own. Builder colored strokes are generated dynamically via `createBuilderDots()`.
