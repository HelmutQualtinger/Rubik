# Prompt: Reconstruct this exact Rubik's Cube app

Feed this to an LLM (with no other context) to regenerate this project's `index.html` as closely as possible: a single self-contained file, no build step, no dependencies to install.

---

Build a single `index.html` file — a fully interactive, visually polished 3×3×3 Rubik's Cube. One file, no build tooling, no `package.json`. Three.js and its addons load from a CDN via an `importmap`; everything else is inline `<style>` and one `<script type="module">`.

## Page identity & copy

- `<title>Cube — a study in motion</title>`
- Google Fonts, preconnected: **Fraunces** (weights/styles `0,9..144,300`, `0,9..144,500`, `0,9..144,600`, `1,9..144,400`, `1,9..144,500` — variable optical-size serif) for display type, and **Space Mono** (400, 700) for everything else (UI labels, body).
- Eyebrow label above the title: `No. 87 — Object Study`, with a small rotated-square (`::before`, 0.5em, 1px brass border, rotate 45°) bullet before it.
- Title: `Cube<em>,</em><br>turning<em>.</em>` — Fraunces, italic, weight 500, the `em` tags de-italicized to weight 300 in a dimmer color so the commas/periods read as punctuation, not emphasis.
- Subline under the title (max 22ch): `A third-order puzzle, rendered in glass and plastic. Handle it.`
- Footer instructions (bottom-left, hidden under 680px): three lines — `Drag a facelet — turn that layer`, `Drag empty space — orbit the view`, `Scroll — zoom`, with the leading phrase in brass color via a `<span class="k">`.
- Number plate (bottom-center, hidden under 680px): `Rubik's Cube · Three.js Rendering`.
- Solved banner (center, hidden until solved): eyebrow `State achieved` above an italic Fraunces `Solved.` in brass with a soft text-shadow glow.
- Buttons (bottom-right): `Reset`, `Solve` (disabled until at least one move exists), `Scramble` (styled as the primary/emphasized action).
- Sound toggle (top-right, under the stat panel): a small dot indicator + `Sound On`/`Sound Off` label, toggled via `aria-pressed`.
- Stat panel (top-right): two cells, `Moves` (integer counter) and `Time` (`MM:SS`, tabular-nums), separated by a hairline border.

## Visual design system

Dark, editorial "object study" aesthetic — like a museum catalog card for a mechanical object, not a game UI.

CSS custom properties:
```
--bg-0:#0a0908;  --bg-1:#151210;
--brass:#c9a24b; --brass-dim:#8a7238;
--paper:#ece7df; --paper-dim:#8f897c;
--line:rgba(236,231,223,0.14); --line-strong:rgba(236,231,223,0.28);
```

- Full-bleed fixed canvas (`#scene`) behind everything, `touch-action:none`.
- `#backdrop`: two radial gradients — a faint warm brass glow centered high-mid, and a dark vignette pooling toward the bottom — layered behind the canvas.
- `#vignette`: a fixed full-screen inset `box-shadow` (large soft dark ring) over everything for a lens/frame effect.
- `#grain`: an inline SVG full-screen overlay using `feTurbulence` (`fractalNoise`, `baseFrequency=0.9`, 2 octaves, stitched) at ~5% opacity with `mix-blend-mode:overlay`, for film grain.
- All HUD elements (`.hud`) are `position:fixed`, `pointer-events:none` at the container level, with `pointer-events:auto` re-enabled only on the actual `<a>`/`<button>` inside — so the 3D scene stays draggable everywhere else.
- Buttons: transparent background, `1px solid` border, uppercase Space Mono label with wide (`0.24em`) letter-spacing, brass border/text + faint brass-tinted background on hover, 1px downward translate on active, disabled state at 32% opacity with hover effects suppressed.
- Every HUD block fades in on load with staggered `fade-up`/`fade-down` keyframe animations (translate 10px + opacity), each with a slightly different `animation-delay` (roughly 0–320ms) so elements arrive in a loose cascade rather than all at once.
- A responsive breakpoint at 680px shrinks paddings/positions and hides the instructions and number plate to keep the mobile view uncluttered.
- Default cursor is `grab`; a `dragging` body class swaps it to `grabbing`, an `on-piece` class swaps it to `pointer` while hovering a grabbable cubie face.

## Three.js setup

- Import map pinning `three@0.169.0` from `unpkg.com`, both `three` and `three/addons/` (the `examples/jsm/` tree) — core and addons must stay on the same version.
- Addons used: `OrbitControls`, `RectAreaLightUniformsLib` (call `.init()` once at module load — required for RectAreaLight to render at all), `RoomEnvironment`.
- `WebGLRenderer({ antialias:true, alpha:true })`, capped pixel ratio at 2, soft PCF shadow maps, `ACESFilmicToneMapping` at exposure 1.05, `SRGBColorSpace` output.
- Environment: bake a `RoomEnvironment` through `PMREMGenerator` into `scene.environment` — needed because the brushed-metal materials are metalness-heavy with no diffuse response, so they need something to reflect.
- Camera: 38° FOV perspective, positioned around `(5.4, 4.6, 6.6)`.
- `OrbitControls`: damping on (0.08), panning disabled, distance clamped `[5.5, 13]`, gentle `autoRotate` at speed 0.6 that stops permanently the first time the user grabs a cubie.

### Lighting rig
- One `HemisphereLight` (warm sky `0xfdf6e8`, dark ground `0x1b1712`, intensity 1.0) for base fill.
- Three `RectAreaLight`s forming a soft-box: a bright warm one from above (`0xfff5e2`, intensity 8, 9×9), a dimmer brass-toned one from below (`0xc9a24b`, intensity 4.5, 9×9) for bounce/rim, and a cool, weak side kick (`0x7fa0e0`, intensity 1.2, 6×6) for separation.
- One low-intensity `DirectionalLight` (`0xfff3df`, 0.5) purely to cast a soft, biased, 2048² PCF shadow — not for illumination.
- A shadow-catcher "plinth": a `CircleGeometry` disc with a canvas-generated radial-gradient alpha texture (dark center fading to transparent) plus a large invisible `ShadowMaterial` ground plane (opacity 0.28) receiving the directional light's shadow, both positioned just below the cube.

### Procedural textures (no image assets, ever)
- **Brushed-metal bump map**: a 512×512 canvas filled mid-gray, then ~9000 near-horizontal streaks of random shade/opacity/thickness drawn across it with slight wobble, wrapped/repeated 3×3, used as a shared `bumpMap` (with anisotropy + `anisotropyRotation`) on both the plastic body material and every sticker material.
- **Sticker texture**: a 256×256 canvas rounded-rect (radius 34, padding 10) filled with the face's color, plus a top-to-bottom white gradient sheen (18%→0% opacity) for a subtle plastic highlight; sRGB color space, anisotropy 4.

### Materials
- `stickerMaterial(colorName)`, memoized per color: `MeshPhysicalMaterial` with the sticker texture as `map` (transparent), metalness 0.9, roughness 0.38, the brushed bump map at scale 0.012, anisotropy 0.6 rotated 90°, `envMapIntensity` 1.15, light clearcoat (0.15, roughness 0.5).
- `plasticMat` (shared by every cubie body): near-black (`0x232323`), metalness 0.95, roughness 0.42, same bump map at scale 0.018, anisotropy 0.8, `envMapIntensity` 1.1.

### Sticker colors (exact)
```
U (up)    #f4f1e8  off-white
D (down)  #f6c60a  yellow
R (right) #d7361e  red
L (left)  #e8791c  orange
F (front) #2a6fd6  blue
B (back)  #2f9e52  green
```

## Cube data model

- Constants: `SPACING = 1.06` (gap between cubie centers), `CUBIE = 0.98` (body size, leaving a visible gap), `STICKER_INSET = 0.82` (sticker plane size), `TURN_MS = 460` (base manual-turn duration).
- `FACES`: six entries, each `{ key, axis, sign, color }` mapping U/D/R/L/F/B to an axis (`x`/`y`/`z`), a sign (±1), and its color key.
- `buildCube()`: nested loop `x,y,z ∈ {-1,0,1}` builds 27 `THREE.Group`s, each holding one shared-geometry `BoxGeometry(CUBIE,CUBIE,CUBIE)` body plus one `PlaneGeometry(STICKER_INSET,STICKER_INSET)` sticker per exterior face (only where the cubie's coordinate matches that face's sign), positioned/rotated to sit flush on the box surface with a tiny 0.012 offset to avoid z-fighting. Each sticker gets `userData.face` set to its color key. Callable again wholesale (rebuilds from scratch) for Reset. Cubies are tracked in a module-level array as `{ group, pos:{x,y,z} }`.
- Immediately after building, every sticker's fixed **local** outward normal is cached once as `userData.localNormal` (derived from `new THREE.Vector3(0,0,1).applyEuler(child.rotation)`) — this is what solve-detection reads later, rotated by the *cubie group's* current world quaternion, not the sticker mesh's own.

## Layer turning

- `rotateLayer(axis, layerValue, dir, {countMove, duration, record})`: filters cubies whose `pos[axis] === layerValue`, creates a temporary pivot `Group`, `pivot.attach()`s each affected cubie (preserves world transform while reparenting), animates `pivot.rotation[axis]` from 0 to `dir * 90°` over `duration` ms with a cubic ease-out (`1 - (1-t)^3`) via `requestAnimationFrame`, plays a click sound at animation start, then on completion: reparents cubies back onto the root with `cubeRoot.attach()`, snaps each transform to the nearest exact grid position/90° rotation (`snapTransform`, preventing float drift from accumulating over many turns), updates each cubie's integer `.pos` via a hand-rolled `integerRotate(pos, axis, dir)` (a 90°-rotation permutation of the two non-turning coordinates, sign depending on `dir`), and — unless `record:false` — pushes `{axis, layerValue, dir}` onto a `history` array and refreshes the Solve button's disabled state. Returns a Promise so callers can `await` sequences.
- `solveCube()`: pure undo — pops the entire `history` array, reverses it, and replays each move with `dir` negated and `record:false` (shorter 230ms duration), landing back at solved regardless of how the cube got scrambled. No actual solving algorithm.
- `scramble(n=22)`: fires `n` random `{axis, layerValue∈{-1,0,1}, dir∈{-1,1}}` turns sequentially through `rotateLayer` (210ms each, `countMove:false` but still recorded to history so Solve can undo a scramble too).
- A single `animating` boolean lock prevents any new grab/scramble/solve from starting while one is in flight.

## Solve detection

- Orientation-agnostic: for each of the 6 global faces, for every cubie currently in that face's layer, find whichever of its stickers' **current world-space normal** (cached local normal rotated by the cubie group's live world quaternion) points closest to that face's axis — not whichever sticker was originally assigned there — and require all 9 to report the same color key. Fires `onSolved()` (shows the banner, stops the timer, plays the chime, latched via a `solvedShown` flag so it only fires once) the instant every face matches, however the cube got scrambled or however it's currently oriented in view.

## Pointer interaction — the turn-vs-orbit disambiguation

This is the trickiest part; implement it exactly:

- `pointerdown` on the canvas: `ensureAudio()`, stop `autoRotate` permanently, raycast against cubie **body** meshes only. If nothing hit, let `OrbitControls` handle it natively (camera orbit). If a body is hit, derive which face was grabbed from the hit's local face normal transformed to world space and rounded to the nearest axis, disable `OrbitControls`, and start a `drag` record (grabbed cubie, normal axis/sign, exact 3D grab point, starting screen coords) — but don't commit to a turn yet.
- On `pointermove`, before the drag is "resolved" (a small ~8px screen threshold): a grabbed face can only turn about the two axes lying in its own plane (the two axes other than its normal) — 4 candidate turns total. For each of the 2 candidate axes, compute the 3D tangent direction a point at the grab location would move under a small *positive* rotation about that axis (`tangent = axis × normal`, i.e. `v = ω × r`), project both the grab point and grab-point-plus-tangent through the camera to get that motion as a 2D screen-space direction, and dot it against the actual on-screen drag vector. The axis whose tangent best aligns (largest `|dot|`) wins; lock in `drag.axis`. Also calibrate `pixelsPerQuarterTurn` by projecting an *actual* 90°-rotated grab point through the camera and measuring the screen distance — accurate at large angles, unlike the small linear nudge used just for picking the axis. Then build the real pivot group and reparent the affected layer onto it, claim the shared `animating` lock, and add the `dragging` cursor class.
- On every subsequent `pointermove` once resolved: project the live mouse-drag vector onto the locked screen-space tangent direction, convert "pixels along the tangent" to an angle via the calibrated `pixelsPerQuarterTurn`, clamp to `±90°`, and drive `pivot.rotation[axis]` directly from that angle every frame — the layer visually follows the cursor in real time, not just an animated end state.
- On `pointerup`: if the drag was resolved, commit forward to the nearest quarter turn if `|angle| ≥ 45°`, else snap back to 0 (a cancelled/undone drag). `finishDrag(drag, dir)` animates the remainder with the same ease-out curve, then performs the same bake/snap/history/stats/solve-check steps as `rotateLayer`'s completion, releases the `animating` lock, and clears `drag`.

## Audio

Synthesized entirely with the Web Audio API — no audio files anywhere.

- `ensureAudio()`: lazily creates (or resumes, if suspended) a single shared `AudioContext`, gated by a `soundEnabled` flag. Must be called synchronously inside a real user-gesture handler (`pointerdown`, every control-button `click`) — never from inside a `.then()` or animation callback, or the context stays suspended and sound silently never plays.
- `playClick(pitch=1)`: layers a ~35ms bandpassed (2200Hz center × pitch, Q 1.1) white-noise burst with a cubic decay envelope (the transient "click") on top of a short sine oscillator sweeping 190→95Hz × pitch with its own fast exponential decay (the low "thock" body). Called once per completed turn, with a small `0.92 + random()*0.16` pitch jitter per call so a rapid scramble/solve doesn't sound perfectly mechanical/repetitive.
- `playSolvedChime()`: a 4-note triangle-wave arpeggio, C5→E5→G5→C6 (523.25/659.25/783.99/1046.5 Hz), each note starting 0.1s after the last with its own quick linear attack + exponential decay envelope.
- A sound-toggle button flips `soundEnabled`, updates its `aria-pressed` attribute and label text/dot styling, and calls `ensureAudio()` if re-enabling.

## Stats

- Move counter increments only on moves where `countMove` is true (manual drags), not on scramble/solve moves.
- Timer starts lazily on the first counted move (`maybeStartTimer`), updates a `MM:SS` display every 250ms via `setInterval`, and stops when the cube is solved. Reset zeroes and stops it.

## Buttons wiring

- **Scramble**: guarded by the `animating` lock; calls `ensureAudio()`, runs `scramble()`, releases the lock in a `finally`.
- **Solve**: guarded by `animating` and an empty-history check; disabled attribute is kept in sync via `updateSolveButtonState()` after every move that touches `history`.
- **Reset**: guarded by `animating`; calls `buildCube()` again (full rebuild, not a reverse-animation), re-patches sticker local normals, clears the solved banner, resets stats, and empties `history` — instant, not animated.

## Verification

Open `index.html` directly (`file://` works fine — everything loads from the CDN import map, no server needed). Confirm: dragging a facelet turns only that layer and follows the cursor live; dragging empty space orbits; scroll zooms; Scramble randomizes; Solve animates back to solved from any state including after manual moves mixed with a scramble; Reset is instant and clears move count/timer; the Solved banner and chime fire exactly once per solve and reset correctly on the next scramble; the sound toggle mutes both click and chime. Check the browser console for errors (or drive it headlessly with Playwright, watching for `pageerror`/console output, and take a screenshot).
