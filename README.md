# Cube, turning.

An interactive 3×3×3 Rubik's Cube built with [Three.js](https://threejs.org/) — a single self-contained `index.html`, no build step, no dependencies to install.

**Play it live: [helmutqualtinger.github.io/Rubik](https://helmutqualtinger.github.io/Rubik/)**

![Screenshot of the Rubik's Cube app](screenshot.png)

## Run it

Open the [live version](https://helmutqualtinger.github.io/Rubik/), or open `index.html` locally in a modern browser. That's it — Three.js and its addons load from a CDN via an import map, so there's nothing to install or build.

## Controls

| Action | Effect |
|---|---|
| Drag a facelet | Turns that layer 90° |
| Drag empty space | Orbits the camera around the cube |
| Scroll | Zooms in/out |
| **Scramble** button | Randomly mixes the cube |
| **Solve** button | Animates every move you've made back out, in reverse, until the cube is solved |
| **Reset** button | Instantly returns to a solved cube and clears history |

The HUD tracks your move count and a timer (starts on your first turn), and shows a "Solved." banner the moment every face matches — orientation-agnostic, so it works correctly no matter how the cube has been scrambled or rotated in view.

## How it works

- **27 independent cubies** sit on an integer grid; each tracks its own logical `(x, y, z)` position separately from its Three.js transform.
- **Face turns** are done by reparenting the affected layer's cubies onto a temporary pivot group, animating the pivot's rotation, then baking the result back into each cubie's position and grid coordinate.
- **Solve** doesn't run a cube-solving algorithm — it replays every recorded move (scramble included) in reverse with the direction flipped, an exact undo stack, so it always lands on solved regardless of how the cube got scrambled.
- **Turn detection** works by comparing your on-screen drag direction against the two possible rotation axes for the face you grabbed (each projected into screen space), picking whichever matches best.
- **Materials** are brushed metal: `MeshPhysicalMaterial` with high metalness, an anisotropic highlight, and a procedurally generated brush-grain bump map, lit by a soft top/bottom `RectAreaLight` rig plus a procedural studio environment map for reflections.

See [`CLAUDE.md`](CLAUDE.md) for a fuller architecture breakdown if you're digging into the code.
