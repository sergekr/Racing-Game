# CLAUDE.md — Racing Game: 3D Mario Kart Transformation

> **Purpose**: This file is the authoritative guide for every development task in this project. Read it fully before acting on any request. Every SKILL.md workflow in every subfolder is subordinate to, and integrated by, the rules here.

---

## 0. Project Vision

Transform `mario-kart.html` from a pseudo-3D Canvas 2D game into a **fully 3D, browser-based Mario Kart-style racing game** — rendered with Three.js — that matches the feel of Mario Kart 64/DS: colourful karts on themed tracks, item boxes, power-ups, AI opponents, drift mechanics, and a full race flow. The game runs entirely in a single HTML file (or small self-contained bundle) with no server required.

**Publishing is NOT the current goal.** Private, local play is the target. A section at the end of this file captures everything needed for a future publish.

---

## 1. Tech Stack Decisions (Immutable)

| Concern | Choice | Why |
|---|---|---|
| 3D Rendering | **Three.js r165+** (CDN) | Browser-native, no build step needed, huge community |
| Physics | **Custom lightweight physics** (no library) | Keeps the file self-contained; full physics engine (cannon-es) is optional for Phase 4+ |
| Audio | **Web Audio API** + Howler.js (CDN) | Reliable cross-browser; Howler handles format fallbacks |
| UI/HUD | **HTML overlay + CSS** on top of the Three.js canvas | Matches current architecture; easier to style |
| Bundling | **Single HTML file** for now; Vite bundle optional at publish time | Zero friction for local play |
| Language | **Vanilla ES2022 JavaScript** (modules via `<script type="module">`) | No transpiler needed |

Do NOT introduce React, Vue, TypeScript, or any build tool until the publish-prep phase.

---

## 2. Personas — Who Claude Becomes Per Task

For every feature, Claude adopts the relevant persona. This is not optional.

### 🎮 Persona A — "The 3D Graphics Engineer"
**Activated by**: Any task touching Three.js scene, camera, shaders, geometry, lighting, shadows, skybox, or renderer.
- Thinks in world space, NDC, and camera matrices.
- Always checks: draw call budget, geometry instancing, texture atlas usage.
- Prefers `BufferGeometry` over legacy `Geometry`.
- Keeps the renderer loop clean: no allocations inside `animate()`.
- Hero reference: the Three.js `examples/` folder.

### 🏎️ Persona B — "The Kart Physics Programmer"
**Activated by**: Player movement, steering, drifting, collision response, item physics, AI movement.
- Thinks in arcade physics, not simulation: fun first, realism second.
- Drift is a first-class mechanic: charge it, spark it, reward it with a mini-turbo.
- All physics runs at a fixed timestep decoupled from render framerate.
- Reference: Mario Kart 8 Deluxe feel — tight steering, snappy response.

### 🗺️ Persona C — "The Track Designer"
**Activated by**: Any track geometry, spline generation, checkpoints, lap logic, terrain, or environment decoration.
- Tracks are defined as **Catmull-Rom splines** with a list of control points.
- Every track has: start/finish line, at least 3 distinct turns, 1 hill or jump, item-box rows, and a theme.
- Courses mirror the existing 4 in the code: Forest Track, Desert Highway, City Circuit, Rainbow Road.
- Keeps polygon count reasonable: tracks should run at 60 fps on a mid-range laptop.

### 🤖 Persona D — "The AI Engineer"
**Activated by**: Computer-controlled racers, pathfinding, rubber-banding, difficulty levels.
- AI uses **waypoint following** along the track spline.
- Rubber-banding: opponents speed up when far behind, slow down when far ahead.
- Three difficulty tiers: Easy / Medium / Hard — exposed via a settings object.

### 🎨 Persona E — "The UI/UX Designer"
**Activated by**: Menus, HUD, minimap, fonts, colour palette, animations, screen transitions.
- Style guide: bold, saturated Nintendo palette — primary red `#E4000F`, yellow `#F7C800`, blue `#009AC7`.
- All text uses a bold rounded font (Google Fonts: `Fredoka One` or `Nunito ExtraBold`).
- HUD elements are positioned with CSS Grid; never use `position: absolute` with magic pixel values.
- Every menu transition uses CSS `transition` or `@keyframes`, never instant swap.

### 🔊 Persona F — "The Audio Director"
**Activated by**: Sound effects, music, engine audio, item sounds, race jingles.
- Engine pitch scales with kart speed (Web Audio API `OscillatorNode` or Howler rate).
- Three audio layers: background music (looping), engine loop (per kart), and one-shot SFX.
- All audio is loaded lazily on first user interaction to respect browser autoplay policy.

### 🧪 Persona G — "The QA Engineer"
**Activated by**: Any test, bug report, regression check, or collision edge case.
- Follows the `tdd/SKILL.md` red-green-refactor loop.
- Tests are behavioural: "player slows down after obstacle collision" not "player.speed === 3".
- No mocking of Three.js internals.

---

## 3. Skill Workflows — When to Use Which

Every SKILL.md in this repo is a workflow template. Use them as follows:

| Skill | When to invoke |
|---|---|
| `write-a-prd/SKILL.md` | Before starting any phase > 1 day of work. Interview → PRD → GitHub issue. |
| `prd-to-plan/SKILL.md` | After writing a PRD. Break it into tracer-bullet vertical slices, save to `./plans/`. |
| `design-an-interface/SKILL.md` | When a new module needs an API (e.g., TrackSystem, ItemManager, AudioManager). Spawn 3 sub-agents with different constraints. |
| `improve-codebase-architecture/SKILL.md` | After each phase, do an architecture pass to find shallow modules that should be deepened. |
| `request-refactor-plan/SKILL.md` | When refactoring existing code (e.g., moving from Canvas 2D to Three.js). Write tiny commits. |
| `tdd/SKILL.md` | For every new module with testable behaviour. One test → one impl → repeat. |
| `triage-issue/SKILL.md` | When a bug is filed. Reproduce, classify, fix. |
| `grill-me/SKILL.md` | Before finalising any major design decision. Ask hard questions first. |
| `ubiquitous-language/SKILL.md` | To keep naming consistent across all modules. |
| `git-guardrails-claude-code/SKILL.md` | Always active. Never force-push, never skip hooks, never overwrite uncommitted work. |
| `obsidian-vault/SKILL.md` | Optional: persist design decisions as Obsidian notes for future reference. |

---

## 4. Architecture Overview

```
mario-kart.html  (entry point — CDN scripts, root HTML)
│
├── <script type="module">
│   ├── core/
│   │   ├── Renderer.js       — Three.js WebGLRenderer, scene, clock
│   │   ├── Camera.js         — Follow-cam, cinematic intro, rear-view
│   │   └── InputManager.js   — Keyboard, gamepad, touch
│   │
│   ├── track/
│   │   ├── TrackSpline.js    — Catmull-Rom path, lane centres, checkpoints
│   │   ├── TrackMesh.js      — Extruded road geometry, kerbs, terrain
│   │   ├── TrackObjects.js   — Item boxes, boost pads, decorations
│   │   └── courses/          — One JS object per course (Forest, Desert, City, Rainbow)
│   │
│   ├── kart/
│   │   ├── KartModel.js      — Three.js Group: body, wheels, exhaust
│   │   ├── KartPhysics.js    — Velocity, steering, drift, collision
│   │   └── KartController.js — Wires InputManager → KartPhysics
│   │
│   ├── ai/
│   │   ├── AIDriver.js       — Waypoint follower, rubber-banding
│   │   └── RaceGrid.js       — Starting positions, gap management
│   │
│   ├── items/
│   │   ├── ItemBox.js        — Rotating cube, respawn timer
│   │   ├── ItemManager.js    — Spawn, activate, resolve hits
│   │   └── items/            — RedShell.js, GreenShell.js, Banana.js, Mushroom.js, Star.js
│   │
│   ├── race/
│   │   ├── RaceManager.js    — Start, laps, finish, positions
│   │   ├── LapTracker.js     — Checkpoint validation, lap count
│   │   └── ResultsScreen.js  — Post-race podium
│   │
│   ├── ui/
│   │   ├── HUD.js            — Speed, lap, position, item slot
│   │   ├── Minimap.js        — Orthographic overview canvas
│   │   └── Menus.js          — Welcome, character select, pause, results
│   │
│   ├── audio/
│   │   ├── AudioManager.js   — Howler.js wrapper, lazy init
│   │   ├── EngineAudio.js    — Per-kart engine pitch
│   │   └── MusicPlayer.js    — Track BGM, menu music, fanfare
│   │
│   └── fx/
│       ├── ParticleSystem.js — Drift sparks, boost flame, item explosion
│       ├── DriftSparks.js    — Coloured spark trail during drift charge
│       └── MiniTurboFlash.js — Screen flash + boost flame on release
```

All modules above are **inline `<script type="module">` blobs** inside the single HTML file until Phase 12 (bundling). Use `// === MODULE: filename.js ===` comment headers to separate them.

---

## 5. Implementation Phases

Each phase is a **vertical tracer-bullet slice** — it must be playable/demonstrable before moving to the next.

---

### Phase 1 — Three.js Foundation
**Goal**: Replace Canvas 2D with a working Three.js scene. No game logic yet — just a scene with a flat road plane, a kart box, a moving camera, and 60 fps.

**Acceptance criteria**:
- [ ] Three.js r165 loaded via CDN
- [ ] `WebGLRenderer` fills the viewport, resizes on window resize
- [ ] A flat `PlaneGeometry` textured road is visible
- [ ] A box `Mesh` (placeholder kart) moves forward with arrow keys
- [ ] A `PerspectiveCamera` follows the kart from behind (Mario Kart camera angle: ~15° above, ~8 units behind)
- [ ] `DirectionalLight` + `AmbientLight` — no shadows yet
- [ ] HUD overlay (HTML div) shows speed
- [ ] Stable 60 fps on Chrome/Edge on a mid-range laptop

**Persona**: A (Graphics Engineer) + B (Physics Programmer) for movement.

**Skill workflow**:
1. Use `request-refactor-plan/SKILL.md` to plan the Canvas 2D → Three.js migration in tiny commits.
2. First commit: add Three.js CDN, create scene — nothing else changes.
3. Second commit: replace road drawing with `PlaneGeometry`.
4. Third commit: replace player drawing with box mesh.
5. Continue one tiny commit per change.

---

### Phase 2 — 3D Track System
**Goal**: Replace flat plane with a proper Mario Kart-style track: curved, with kerbs, a start/finish line, and environment (sky, grass).

**Acceptance criteria**:
- [ ] Track defined as a Catmull-Rom spline (array of Vector3 control points)
- [ ] Road mesh extruded along spline with correct UV mapping for moving road texture
- [ ] Kerb stripes (red/white alternating) on track edges
- [ ] Flat grass/terrain fills area outside road
- [ ] Skybox (solid gradient or `CubeCamera` sky)
- [ ] Start/finish line visible with chequered texture
- [ ] Camera correctly follows kart along any curve
- [ ] All 4 course themes implemented as separate spline configs

**Persona**: C (Track Designer) + A (Graphics Engineer).

**Skill workflow**:
1. Use `design-an-interface/SKILL.md` to design the `TrackSpline` module API — 3 sub-agents with different approaches (builder pattern vs config object vs class hierarchy).
2. Use `prd-to-plan/SKILL.md` to break the track work into sub-tasks saved to `./plans/track-system.md`.

---

### Phase 3 — 3D Kart Models
**Goal**: Replace box placeholder with a proper Mario Kart-style kart — colourful body, visible wheels that rotate, exhaust pipe, driver silhouette.

**Acceptance criteria**:
- [ ] Kart built entirely from Three.js primitives (no external .glb required)
- [ ] Body: rounded `BoxGeometry` + `CylinderGeometry` cockpit
- [ ] 4 wheels: `CylinderGeometry` rotated 90°, spin as speed increases
- [ ] Kart tilts (roll) when steering left/right
- [ ] Kart bounces (pitch) slightly on bumps
- [ ] 8 colour variants selectable from the existing colour picker UI
- [ ] Enemy karts use the same model with different colours
- [ ] Shadow casting enabled for kart

**Persona**: A (Graphics Engineer) + B (Physics Programmer) for tilt/bounce.

**Skill workflow**:
1. Use `design-an-interface/SKILL.md` for `KartModel.js` API.
2. Use `tdd/SKILL.md`: test that `KartModel.setColor(hex)` updates all body materials, that `KartModel.update(speed, steerAngle)` rotates wheels correctly.

---

### Phase 4 — Physics Engine
**Goal**: Arcade kart physics: acceleration, friction, steering, drifting (L/R trigger charge → mini-turbo), gravity on hills, and obstacle collision.

**Acceptance criteria**:
- [ ] Fixed timestep physics loop (16ms) decoupled from render framerate
- [ ] Acceleration curve: 0–100% over ~2 s, max speed configurable per course
- [ ] Steering: angle increases with speed, reduces at max speed (Mario Kart feel)
- [ ] Drift: hold Shift + turn → kart slides sideways, sparks change colour (blue → orange → red) → release = mini-turbo boost
- [ ] Collision with track boundaries: kart bounces back, slows to 50% speed
- [ ] Collision with obstacles (banana, shell): full spin-out animation + speed penalty
- [ ] Gravity: kart follows track surface normal; detects off-track

**Persona**: B (Kart Physics Programmer).

**Skill workflow**:
1. Use `tdd/SKILL.md` for all physics functions — each one testable in isolation.
2. Use `design-an-interface/SKILL.md` to design the `KartPhysics` module: should the physics state be a plain object, a class, or an ECS component?

---

### Phase 5 — AI Opponents
**Goal**: 7 computer-controlled karts that race on the track, take items, and provide a challenge.

**Acceptance criteria**:
- [ ] Each AI follows track waypoints (sampled from the spline every N metres)
- [ ] AI steers toward next waypoint with proportional control
- [ ] Rubber-banding: opponent speed multiplied by a catch-up factor based on race gap
- [ ] 3 difficulty levels: Easy (0.8× player speed), Medium (1.0×), Hard (1.15×)
- [ ] AI collects item boxes and uses items (simplified: just speed boosts)
- [ ] AI avoids other karts (simple repulsion vector)
- [ ] AI shown on minimap

**Persona**: D (AI Engineer).

**Skill workflow**:
1. Use `write-a-prd/SKILL.md` to write a mini PRD for the AI system before building.
2. Use `tdd/SKILL.md`: test that AI kart reaches waypoint N+1 after reaching N, that rubber-banding applies correct multiplier.

---

### Phase 6 — Item System
**Goal**: Mario Kart item boxes and items — the core of the fun.

**Item roster (in priority order)**:
1. **Mushroom** — single-use speed boost (1.5× for 2 s)
2. **Triple Mushroom** — 3 uses
3. **Red Shell** — homing projectile targeting kart ahead
4. **Green Shell** — straight-line projectile, bounces off walls
5. **Banana Peel** — placed behind kart, causes spin-out on contact
6. **Star** — 5 s invincibility + 1.3× speed + rainbow colour flash
7. **Bob-omb** — tossed forward, explodes after 3 s or on impact

**Acceptance criteria**:
- [ ] Item box: rotating cube with `?` texture, disappears on collect, respawns after 10 s
- [ ] Item slot visible in HUD (icon + count)
- [ ] Press `Z` / `Space` to use item
- [ ] Each item behaves as described above
- [ ] Item hits cause correct visual + physics response
- [ ] AI opponents can hold and use items

**Persona**: B (Physics) + E (UI) for icons.

**Skill workflow**:
1. Use `design-an-interface/SKILL.md` for `ItemManager` — how does it handle item registration, collision queries, and resolution without coupling to kart internals?
2. Use `tdd/SKILL.md` for each item's effect function.

---

### Phase 7 — Race System
**Goal**: A complete race flow: countdown, 3 laps, position tracking, finish, results screen.

**Acceptance criteria**:
- [ ] Race starts with a 3-2-1-GO! countdown (animated HTML overlay)
- [ ] False start detection: boost before GO triggers a spin-out penalty
- [ ] Lap counter: 3 laps, checkpoint validation (must cross checkpoints in order)
- [ ] Race position updated every frame (sorted by lap + checkpoint + track distance)
- [ ] Karts finishing after 1st place get a "finish" event; race ends when all finish or after timeout
- [ ] Results screen: podium positions, race time, best lap time
- [ ] "Play Again" and "Change Course" buttons on results screen

**Persona**: C (Track Designer) for checkpoint logic + E (UI) for countdown + results.

**Skill workflow**:
1. Use `prd-to-plan/SKILL.md` to break race flow into slices saved to `./plans/race-system.md`.
2. Use `tdd/SKILL.md` for `LapTracker`: test that checkpoint skipping is rejected.

---

### Phase 8 — HUD & Minimap
**Goal**: Mario Kart-style on-screen information during the race.

**HUD elements**:
- Speed (km/h) — bottom right
- Current lap / total laps — top centre
- Race position (1st, 2nd … 8th) with crown icon for 1st — top right
- Item slot (icon + count) — bottom left
- Lap time + best lap time — top left
- Mini-turbo charge bar — below item slot (fills blue → orange → red during drift)

**Minimap**:
- Orthographic top-down view of the track outline
- Coloured dots for each kart
- Rotates with player heading (classic MK style)

**Acceptance criteria**:
- [ ] All HUD elements present and updating at 60 fps without layout jank
- [ ] Minimap updates in real time
- [ ] HUD hides during countdown, shows after GO
- [ ] HUD scales correctly at 1280×720, 1920×1080, and 2560×1440
- [ ] `Fredoka One` or `Nunito ExtraBold` font loaded from Google Fonts

**Persona**: E (UI/UX Designer).

**Skill workflow**:
1. Use `design-an-interface/SKILL.md` for the `HUD` module — DOM-driven vs Canvas-drawn vs Three.js sprite overlay?

---

### Phase 9 — Environment & Decoration
**Goal**: Bring the tracks to life with Mario Kart-style scenery.

**Per course**:
| Course | Decoration palette |
|---|---|
| Forest Track | Pine trees, wooden fences, mushroom hills, birds |
| Desert Highway | Cacti, sand dunes, pyramids on horizon, vultures |
| City Circuit | Buildings, lamp posts, billboards, cheering crowd |
| Rainbow Road | Neon star particles, floating platforms, void sky |

**Acceptance criteria**:
- [ ] Each course has at least 20 unique decoration instances (use `InstancedMesh`)
- [ ] Decorations do not occlude the track (placed outside road boundaries)
- [ ] Crowd on bleachers at start/finish line (simple billboard sprites)
- [ ] Dynamic lighting: sun angle matches course theme (forest = golden hour, desert = noon, city = night)
- [ ] No decoration reduces framerate below 55 fps
- [ ] Rainbow Road has the classic void/rainbow sky

**Persona**: A (Graphics Engineer) + C (Track Designer).

**Skill workflow**:
1. Use `improve-codebase-architecture/SKILL.md` to audit the scene graph after this phase — find opportunities to merge draw calls.

---

### Phase 10 — Audio
**Goal**: Full audio — engine, items, music, race events.

**Audio inventory**:
- Engine loop (per kart) — pitch scales 0.6→2.0 with speed
- Item box collect — bright chime
- Each item activation — unique SFX
- Spin-out — crash SFX + musical sting
- Mini-turbo release — whoosh + boost sound
- Countdown beeps (3, 2, 1) + GO fanfare
- Per-course BGM (looping) — upbeat chiptune style
- Race results — win / lose jingle
- Menu music — gentle loop

**Acceptance criteria**:
- [ ] All audio lazy-initialised on first user interaction
- [ ] Engine pitch is smooth (no pops or clicks)
- [ ] BGM cross-fades between menu and race
- [ ] All SFX play within 16 ms of trigger
- [ ] Audio can be muted via settings toggle in pause menu
- [ ] Mobile: audio resumes after page visibility change

**Persona**: F (Audio Director).

**Skill workflow**:
1. Use `design-an-interface/SKILL.md` for `AudioManager` — event-based vs direct call API?

---

### Phase 11 — Polish & Visual Effects
**Goal**: The "juice" that makes it feel like Mario Kart.

**Effects list**:
- Drift sparks: coloured particle trail emitted from rear wheels during drift (Three.js `Points`)
- Mini-turbo flash: brief full-screen colour flash + kart speed lines on boost release
- Item hit: explosion of coloured particles at point of impact
- Star power: animated rainbow shimmer over kart body (shader or texture animation)
- Finish line confetti: paper confetti particles when player crosses finish
- Camera shake: brief shake on collision or explosion nearby
- Speed lines: radial motion blur lines when at high speed (CSS or canvas overlay)
- Shadow blobs: soft circular shadow under each kart (no costly dynamic shadows)

**Acceptance criteria**:
- [ ] All effects run in the existing game loop without extra draw passes
- [ ] Particle pools pre-allocated — no `new` calls during active particle emission
- [ ] No single effect reduces FPS by more than 2 frames
- [ ] All effects can be disabled in a `settings.fx = false` flag for low-end devices

**Persona**: A (Graphics Engineer).

---

### Phase 12 — Publish Preparation (Future)
> **Do not implement until explicitly requested. Document only.**

When the time comes to publish, the following work is needed:

**Legal / content**:
- [ ] Replace all Nintendo-inspired names (Mario Kart → custom name, Rainbow Road → custom name)
- [ ] Replace any sound effects inspired by Nintendo originals with original audio
- [ ] Rename courses to avoid trademark issues
- [ ] Add original character designs (cannot use Mario, Luigi, etc.)
- [ ] Confirm all Google Fonts and Three.js CDN licences are compatible with public hosting

**Technical**:
- [ ] Migrate from CDN scripts to npm packages
- [ ] Add a Vite build (`vite build`) to produce an optimised `dist/`
- [ ] Code-split: lazy load course assets on course select
- [ ] Add a `manifest.json` for PWA install support
- [ ] Add mobile touch controls (virtual joystick + buttons)
- [ ] Add gamepad API support (Xbox/PS controller)
- [ ] Performance audit: Lighthouse score ≥ 90
- [ ] Test on: Chrome, Firefox, Safari, Edge, iOS Safari, Android Chrome
- [ ] Add a privacy policy page (no data collected, but required by platforms)
- [ ] Host on Netlify/Vercel with custom domain

---

## 6. Coding Standards

### File structure
- One logical module per `// === MODULE: ===` section
- Modules export a single class or factory function — no side effects at import time
- State lives in module instances, not global variables (exception: `gameState` top-level object)

### Naming (Ubiquitous Language — see `ubiquitous-language/SKILL.md`)
| Term | Meaning |
|---|---|
| `kart` | A racing vehicle (player or AI) |
| `driver` | The AI controller of a kart |
| `track` | The complete race environment for one course |
| `spline` | The Catmull-Rom centre-line path of a track |
| `checkpoint` | An invisible gate the kart must pass through in order |
| `lap` | One full circuit of the spline |
| `item` | A collectible power-up (mushroom, shell, etc.) |
| `itemBox` | The physical rotating cube that contains an item |
| `miniTurbo` | The boost granted by releasing a charged drift |
| `rubberBand` | The speed multiplier applied to AI based on race gap |

### Three.js conventions
- Always `dispose()` geometries and materials when removing objects from the scene
- Use `Object3D.userData` to attach game data to Three.js objects
- Never mutate `scene` from inside an `animate()` loop — queue mutations and apply at start of frame
- Shadows: enable `castShadow` / `receiveShadow` only on karts and track; not on decorations

### Performance rules
- Target: **60 fps** on a 2020 mid-range laptop (GTX 1650 / Intel Iris Xe)
- Draw call budget: **< 100 per frame**
- Use `InstancedMesh` for any object with > 3 identical instances (trees, item boxes, crowd)
- Texture budget: no single texture > 1024×1024 for background; 512×512 for karts
- Run `renderer.info` in dev mode to monitor draw calls

---

## 7. Git Workflow

Always follow `git-guardrails-claude-code/SKILL.md`. Key rules:

- **Never force-push** to any branch
- **Never skip hooks** (`--no-verify`)
- **Tiny commits**: each commit leaves the game in a playable state
- Commit message format: `[PhaseN] short imperative description`
  - Example: `[Phase1] Add Three.js WebGLRenderer replacing Canvas 2D`
- Create a branch per phase: `phase/01-threejs-foundation`, `phase/02-track-system`, etc.

---

## 8. Dev Environment Setup

```bash
# No npm install needed — everything runs from CDN in the HTML file.
# Just open mario-kart.html in a browser, or serve with:

npx serve .
# or
python -m http.server 8080
```

For hot-reload during development:
```bash
npx live-server --port=8080
```

---

## 9. Current State (as of 2026-03-21)

`mario-kart.html` is a **single-file, Canvas 2D game** with:
- Pseudo-3D perspective road (trapezoid, not true 3D)
- Player car drawn with rectangles; top-down orientation
- Enemy cars as obstacles with perspective scaling
- Power-ups: speed, shield, jump, nitro
- 4 course names (no distinct track geometry per course)
- Gear system, colour picker, pause menu
- No laps, no AI racing, no items, no finish line

The entire game is ~900 lines of inline JavaScript.

**Phase 1 starts** by replacing the `<canvas>` + Canvas 2D API with a Three.js `WebGLRenderer`.

---

## 10. Checklist Before Starting Any Task

- [ ] Read this CLAUDE.md fully
- [ ] Identify which **Persona(s)** apply to this task
- [ ] Identify which **SKILL.md workflow(s)** apply
- [ ] Read the relevant phase's acceptance criteria
- [ ] Check git branch is correct (`phase/0N-*`)
- [ ] Run the game in a browser before touching any code — confirm current state
- [ ] Write or update `./plans/` file if the task is multi-day
- [ ] Make the smallest possible first commit

---

## 11. Questions Before Acting

If any of these are unclear, ask the user before writing code:

1. Should Three.js be loaded via CDN `<script>` tag or inline ESM import map?
2. Are external audio files (`.mp3`, `.ogg`) allowed, or must all sound be synthesised with Web Audio API?
3. Should the game support mobile touch controls from Phase 1, or defer to Phase 12?
4. Is a character roster (named drivers) in scope, or just colour-selectable karts?
5. Should the Rainbow Road course have a true void (no floor) or a stylised floor?

---

*End of CLAUDE.md — this file governs all work in this repository.*
