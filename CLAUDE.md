# On The New — Project Brief

A prototype/proof-of-concept website for an architecture proposal (Arch League) — a
speculative living archive of NYC buildings that fall outside the official landmarking
process, piloting in Washington Heights. Currently a single-page site plus one
standalone secondary page, all vanilla HTML/CSS/JS + Three.js (no build step, no
framework). Deployed via GitHub Pages.

This file exists to hand off context that isn't obvious from reading the code alone —
design decisions, conventions, and a couple of real bugs that took real iteration to
find, so they don't get reintroduced.

## File structure

```
index.html              — the main site (landing page + 3D grid + isolate view)
models/*.obj            — 3D model assets, referenced by relative path from index.html
map-test/index.html     — standalone map viewer (separate page, not linked from nav
                           except via the M button while browsing the grid)
map-test/map.svg        — the actual site map (vector line-drawing exported from Rhino)
```

**Critical**: `index.html` fetches files from `models/` and the map fetches `map.svg` at
runtime via `fetch()`/`OBJLoader`. This means:
- Opening `index.html` locally via `file://` will NOT load models or the map (browsers
  block `fetch` over `file://`). Test via GitHub Pages or a local server
  (`python3 -m http.server`).
- Any in-chat/preview rendering of `index.html` alone will show placeholder shapes
  instead of real models, for the same reason — this is expected, not a bug.

## The three pages / modes

### 1. Landing page
Full-screen splash, dismissed by clicking the word "New" in the title ("On The <New>").
Space Grotesk typeface, weight contrast (regular/bold) used for hierarchy instead of
color. Right-click anywhere on it toggles an inverted color scheme (yellow ⇄ black),
independent of the grid's own invert state. Fades back in automatically after 60s of
inactivity, and is reachable any time via the **N** button (top-left).

### 2. The grid
A 15×15 (225-cell) infinite-scrolling grid of 3D objects, built in Three.js on an
`OrthographicCamera`. Key mechanics:
- **Infinite wrap**: each cell's position is `wrap(base - offset, viewportSize)` — objects
  reposition modulo the viewport size as you pan, so panning never runs out of content
  without needing to duplicate objects.
- **Color scheme**: default is yellow background / black objects. Right-click toggles to
  black background / canary-yellow objects (`colorsInverted` flag, `applyColorScheme()`).
  A `body.color-inverted` class is toggled in sync so CSS-based UI (nav marks, isolate
  panels) can track the same contrast via `currentColor` / class selectors.
- **Placeholder geometries**: `[Box, Icosahedron, Torus, Cone, Sphere]`, randomly assigned
  per cell with an adjacency constraint (no two touching cells — including diagonals —
  share the same shape).
- **Hover**: a single settle-in jitter nudge (not continuous), scale bump, eased back on
  hover-out.
- **Drag**: pans the grid with momentum/inertia after release (friction-based decay).
- **Click** → isolates that object (see below). **Click-vs-drag** is distinguished by
  total movement distance (<6px = click), used consistently everywhere in this codebase
  where click and drag share a pointer target.

### 3. Isolation / orbit view
Clicking a cell triggers `isolate()`: the object grows in place (no camera movement) while
every other cell fades out — deliberately *not* a camera zoom, after an earlier version
that panned/zoomed the camera looked "sloppy." Once fully grown, camera control hands off
to a `PerspectiveCamera` + `OrbitControls` (rotate only — zoom and pan are disabled,
`enableZoom = false`, `enablePan = false`).

**Load-bearing fact**: because OrbitControls always keeps its `target` centered in frame,
the isolated object is *always* exactly at the screen's geometric center, for any camera
angle. This is relied upon elsewhere (see Leader lines below) — don't reposition the
object itself without updating that logic too. A request was made to move the object
visually to one side of the screen; this was deliberately *not* done (would require
reworking the camera/canvas setup) — instead, the four info panels are arranged in a
cross pattern *around* the fixed center point. That tradeoff was flagged to the user and
accepted.

**No camera jump**: the perspective camera's starting pose exactly matches the orthographic
grid camera's framing (same 1-world-unit-per-pixel scale, computed from FOV + viewport
height), then glides to the centered orbit framing over ~500ms, rather than cutting
directly to it.

**Exit**: only the **N** button returns to the grid (calls `returnToGrid()`, which
properly unwinds state — shrink + fade back in — before showing the landing page if N is
hit from the grid; from isolation, N goes only to the grid, not the landing page).
Clicking the object, clicking empty space, and Escape do **not** exit isolation
(explicitly removed after being confusing/redundant).

## Isolation-view info panels

Three draggable panels + one fixed label, default arranged in a cross around the
(centered) object:

- **Photos** (`#isolate-photos`) — right of the object. 3×3 grid of placeholder tiles;
  clicking one spawns an independent, larger, separately-draggable "enlarged" card with
  its own close button.
- **Address** (`#isolate-address`) — not draggable, just floats below-ish. Placeholder text.
- **Site plan** (`#isolate-blockmap`) — below the object by design intent, but **currently
  positioned bottom-right next to the M button** per a later request ("start the map
  directly next to M"). A hand-drawn SVG line diagram: a full block (closed perimeter,
  corner lots included) + a half-block continuing in all four cardinal directions +
  small corner fragments at the four diagonal corners. Lot proportions are deliberately
  narrow/elongated (~20×45 units) to read as real NYC lots, not generic squares. One lot
  is highlighted as "this building." No card background — floats with just a
  `drop-shadow` filter, like a transparent PNG line-drawing would, not a bordered panel.
  Toggled *only* by the **M** button while isolated (M's behavior is mode-dependent —
  see below) — no separate close button.
- **Facade/elevation** (`#isolate-facade`) — above the object. A simple line-drawing
  building front elevation (cornice w/ brackets, floors of windows, arched parlor
  windows, stoop — same visual language as the site plan and the very first hero
  illustration from early in the project). Opened by **double-clicking the isolated
  object**, or by **double-clicking the highlighted lot** inside the site plan (both call
  the same `toggleFacade()`).

### Leader lines
Every draggable panel gets a thin dashed line back to the object, drawn in a dedicated
`<svg id="leader-svg">` overlay (z-index between the canvas and the panels). Because the
object is always at the exact viewport center (see above), the line's object-side anchor
is just `window.innerWidth/2, innerHeight/2` — no 3D→2D projection needed.

Both ends of each line are **clipped to the edge of what they're connecting to**, not the
center: the near end stops at the object's approximate on-screen radius
(`isolatedObjectRadius`, computed from the actual grown scale in `enterInspectMode`), and
the far end stops at the edge of the panel's bounding box (ray from panel center back
toward the object, clipped against the box's half-width/half-height). This was a
deliberate fix — the line used to visibly cross over the object and penetrate into panel
content before this clipping was added.

### The "M" button (top-right area, next to N)
Context-dependent:
- **From the grid**: normal link to `map-test/index.html`.
- **From isolation**: toggles the site-plan panel instead, and does **not** navigate.
  This is enforced at the source — the `<a>` tag's `href` attribute is actually removed
  the instant isolation begins and restored on return, not just intercepted with
  `preventDefault()` — an earlier version relied on `preventDefault()` alone and it
  wasn't reliable.

### Off-screen drag behavior (important, was buggy, now fixed)
Every draggable panel, when dragged fully off the viewport, does something different by
design:
- **Photo grid**: snaps back to its true rendered "home" position (captured via
  `getBoundingClientRect()` the moment it's first shown, not just "clear inline styles
  and hope CSS reapplies" — that approach was the actual bug in an earlier version and
  produced wildly wrong positions).
- **Individual enlarged photos**: deleted entirely (same as clicking their × — can be
  pulled back out of the grid again).
- **Site plan**: closed entirely (same as toggling M off — reopens fresh via M).
- **Facade**: same as site plan — closes rather than repositions.

**Pattern to reuse**: for anything positioned via CSS defaults that needs a reliable
reset, prefer setting explicit pixel/computed values in JS over clearing inline styles
and trusting the stylesheet to reapply — the latter was unreliable in practice here.

## 3D model pipeline

Real OBJ models replace specific placeholder shapes (currently: every cell with
`shapeIndex === 0`, i.e. what used to be the cube, now loads `models/grid_test_1_centered.obj`).

**Loader**: `THREE.OBJLoader` (r128, loaded via CDN alongside `OrbitControls`). Each cell
is a `THREE.Group` (not a bare `Mesh`) so it can hold either one placeholder mesh or
however many meshes a loaded OBJ actually contains — no assumption that a model is a
single merged shape. `fetchModelObject()` caches the parsed object per URL so repeats
don't re-fetch, and clones it per use. Materials are always replaced with the site's own
`makeMaterial()` (flat color matching the current scheme) — real textures/materials
from the source file are intentionally ignored for now.

**Preprocessing every OBJ needs before use** (done in Python, not in-browser):
1. **Center** on its own bounding-box centroid (uploaded files are often sitting at
   arbitrary coordinates from whatever scene they were exported from).
2. **Normalize scale** so its longest bounding-box dimension is ~2.6 units (matches the
   placeholder geometries' rough size).
3. **Triangulate** any quad/n-gon faces (some exports use quads; don't rely on the
   loader's own n-gon handling — a real bug was traced to exactly this once).

**Known constraints while doing this**:
- No internet access in the code-execution sandbox — can't `pip install` conversion
  tools (trimesh, pygltflib, rsvg-convert, etc.). All preprocessing above is done with
  plain Python (numpy + text parsing), not a mesh library.
- GitHub's web upload UI caps at 25MB/file even though the actual git/repo limit is
  100MB — GitHub Desktop or CLI push is needed for anything 25–100MB.
- **Face count matters a lot.** A 205MB/2.8M-vertex test file was unusably heavy; workable
  single-object range is roughly hundreds to a few thousand triangles for grid cells, up
  to tens of thousands for a hero object shown alone in isolation.

**Two-tier LOD plan (discussed, not yet built)**: show a low-poly/faceted version in the
grid (flat-shaded, deliberately low-poly for an "8-bit" look) and swap to a separate
high-res file only when that object is isolated, since isolation carries the entire
render budget alone vs. sharing it with 224 others in the grid.

**Rendering fixes already applied**:
- `side: THREE.DoubleSide` on the shared material — fixes "missing" backfaces on models
  with inconsistent/inward normals (classic backface-culling symptom).
- `depthWrite: false` on all cell materials — transparent objects were incorrectly
  occluding things behind them at opacity 0 before this was set; this was the actual
  cause of a "ghosting"/shadow bug during the isolate transition.

**SSAO (experimental, isolation view only)**: an `EffectComposer` + `RenderPass` +
`SSAOPass` pipeline, wrapped in `try/catch` so a CDN/version failure falls back to plain
rendering rather than breaking the page. Scoped to the perspective/isolate camera only
(not the grid) since 225 objects don't benefit from it and it's meaningfully more
expensive. Tuning (`kernelRadius`, `minDistance`, `maxDistance`) was estimated for this
project's object scale (tens of world units, not real-world meters) but **not visually
confirmed live** — treat current values as a starting point, not final.

## Map viewer (`map-test/index.html`)

Separate standalone page. The source (`map.svg`) is a genuinely vector line-drawing (10K+
`<path>` elements, ~9MB, no embedded rasters) exported from Rhino — confirmed suitable
for a vector-zoom approach rather than needing a tiled-image pyramid.

**Current approach**: canvas-based rendering, not a scaled `<img>` or scaled DOM `<svg>`.
Both of those were tried first and both went blurry under zoom — browsers can silently
treat a scaled `<img src="*.svg">` or a CSS-transformed live SVG element as a cached
bitmap layer rather than genuinely re-vectorizing at the new size. The current version:
1. On pan/zoom, does a cheap instant bitmap blit of the last good render for immediate
   feedback (some softness while actively dragging is expected/fine).
2. ~160ms after input settles, rewrites the SVG's own `viewBox` to exactly the visible
   region and sets `width`/`height` to the actual destination pixel size, then decodes
   that as a brand-new image — bounding cost/memory to screen resolution regardless of
   zoom level, and forcing a genuinely fresh vector rasterization rather than reusing a
   cached one.

This was iterated through three different technical approaches before landing here; if
zoom quality regresses again, this is the mechanism to revisit, not `<img>`/CSS-transform.

**Not yet built**: the site plan panel is currently a hand-drawn generic SVG placeholder,
not an actual crop of this real map. The intended eventual version: crop `map.svg`'s
`viewBox` to the specific coordinates of whichever building is isolated (reusing the
exact cropping technique above) and inject that instead. This requires each grid
artifact to know its real-world location/coordinate on the map — that per-artifact data
doesn't exist yet. The panel's shell (drag, leader line, show/hide, M-toggle) doesn't
need to change for this swap — only its content.

## Outstanding / not yet built

- **Map's second interaction mode**: clicking a building footprint on the *full* map
  viewer should surface a circle of related objects around that footprint, leading into
  the same isolation experience as the grid. Not started.
- **Real per-artifact content**: photos, addresses, and site-plan/facade data are all
  placeholders everywhere. No data model yet connecting a grid cell to a real building.
- **Two-tier LOD** (low-poly grid / high-res isolation) — discussed, not implemented.
- Only one real model is wired in so far (replacing the cube placeholder across all
  225 cells that would show it).

## Working conventions worth preserving

- Every new model gets centered/normalized/triangulated via the Python pipeline above
  before being handed back for the `models/` folder.
- Prefer additive, defensive changes for anything risky/unverifiable (the SSAO pass, the
  inline-model-data preview fallback) — wrap in `try/catch` or a feature-detection
  fallback so a failure degrades gracefully instead of breaking the page.
- When fixing "goes to the wrong place" bugs on draggable/toggleable UI, prefer explicit
  computed values over clearing inline styles and trusting CSS defaults to reapply.
- Console logging has been deliberately stripped from both `index.html` and the map
  viewer per an explicit request — don't reintroduce `console.log`/`console.error` calls
  as a debugging default; if temporary logging is needed while diagnosing something, say
  so and remove it again afterward.
