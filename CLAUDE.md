# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained `index.html` implementing a fully interactive 3x3x3 Rubik's Cube with Three.js (loaded via CDN + import map — no build step, no package.json, no dependencies to install). Open the file directly in a browser to run it.

## Development

- Run: open `index.html` in any modern browser (or `python3 -m http.server` and visit it — needed only if testing module/CORS behavior, plain `file://` works fine here since Three.js is loaded from a CDN import map).
- There is no build, lint, or test tooling in this repo. Verify changes by loading the page and checking the browser console, or by driving it headlessly with Playwright (`chromium.launch()` → `page.goto('file://.../index.html')`) to check for `pageerror`/console errors and take screenshots.
- Three.js version is pinned in the `<script type="importmap">` block (currently `three@0.169.0`, resolved from `unpkg.com`). Bump the version there — and in the `three/addons/` path — together; addon modules (OrbitControls, RectAreaLightUniformsLib, RoomEnvironment) come from the same `examples/jsm/` tree and must stay in lockstep with core.

## Architecture

Everything lives in one `<script type="module">` block at the bottom of `index.html`, structured top to bottom as:

1. **Constants & color table** — `SPACING`/`CUBIE`/`STICKER_INSET` control cubie geometry and gaps; `COLORS` maps face keys (U/D/R/L/F/B) to sticker hex colors.
2. **Renderer/scene/camera/OrbitControls** — standard Three.js boilerplate. A procedural `RoomEnvironment` is baked via `PMREMGenerator` into `scene.environment` specifically so the brushed-metal materials (metalness-heavy, no diffuse response) have something to reflect.
3. **Lighting rig** — a hemisphere light plus `RectAreaLight`s positioned above/below/side for soft, diffuse illumination, and one dim `DirectionalLight` kept only to cast a soft ground-contact shadow (`sun`).
4. **Materials** — `stickerTexture()`/`brushedTexture()` generate canvas textures at runtime (rounded-rect sticker color, directional brush-noise bump map); `stickerMaterial()` is memoized per color in `stickerMatCache` and `plasticMat` is shared by every cubie body.
5. **Cube data model** (`cubies` array + `buildCube()`): 27 `THREE.Group`s on an integer grid `x,y,z ∈ {-1,0,1}`, each holding one black body mesh plus one sticker `Mesh` per exterior face. Each cubie tracks its logical grid position in `.pos`, independent of its Three.js transform.
6. **Layer rotation** (`rotateLayer`, `integerRotate`, `snapTransform`): to turn a layer, matching cubies are reparented onto a temporary pivot `Group` via `pivot.attach()` (preserves world transform), the pivot is animated one axis by ±90°, then cubies are reparented back with `cubeRoot.attach()` and their `.pos` updated via a hand-rolled integer rotation matrix. Positions/quaternions are snapped to exact grid values afterward to prevent floating-point drift from compounding over many turns. `queueMove()` serializes turns so overlapping animations (e.g. during scramble) never race on `.pos`.
7. **Solve detection** (`checkSolved`): for each of the 6 global faces, finds — per cubie in that layer — whichever sticker's *current world-space normal* points closest to that face's axis (not which sticker was originally assigned there), then checks all 9 are the same color. This makes it orientation-agnostic and correct after arbitrary scrambling.
8. **Pointer interaction**: `pointerdown` raycasts against cubie bodies only; if it hits one, `OrbitControls` is disabled and a drag session starts. On `pointermove` past a small pixel threshold, the two candidate rotation axes (perpendicular to the clicked face's normal) are each projected to screen space as tangent vectors and compared against the actual mouse-drag vector — whichever axis's projected tangent best matches the drag wins, and its sign gives turn direction. This resolves the face-turn on the *first* threshold crossing (discrete "flick" turns, not continuous drag-following rotation). If `pointerdown` misses all cubies, `OrbitControls` handles it natively for camera orbit.
9. **Scramble/Reset/stats**: `scramble()` fires a sequence of un-counted, faster `rotateLayer` calls through the same queue; `buildCube()` is called again wholesale on Reset rather than reversing moves. Move counter and timer are plain module-level state updated from `rotateLayer`'s completion callback.

## Key invariants to preserve when editing

- A cubie's `.pos` (integer grid coordinate) must always be updated *after* its animation finishes and its transform is snapped — never optimistically.
- Any new geometry that needs correct outward-facing normals for `checkSolved()` must have `userData.face` and `userData.localNormal` set the same way `buildCube()`/`patchAllNormals()` do it now.
- Keep face-turn animations going through `queueMove()`, not `rotateLayer()` directly, whenever more than one turn might be in flight (scramble, programmatic sequences) — `rotateLayer()` alone has no race protection.
