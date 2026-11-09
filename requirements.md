# Tempest Clone — Requirements

## 0. Project Overview

A browser-based clone of Atari's 1981 arcade game **Tempest** (designed by Dave Theurer; first Atari colour-vector game). The player pilots a claw-shaped "Blaster" around the near rim of a wireframe tube (the "web") seen in perspective, shooting enemies that climb up the lanes from the far end. Clear every enemy to warp into the next tube.

Fidelity goal: capture the *feel* and rules of the original arcade game, not a pixel-exact ROM port. Items marked **[v1]** are required for the first playable version; **[later]** items are planned but deferred.

## 1. Implementation & Hosting

- Delivered as a single `index.html` (HTML5 + CSS + vanilla JS, no external dependencies).
- The "host" on Windows and Linux is a modern web browser (Chrome, Edge, Firefox). Same file, same code, opened directly via `file://` or served by any static server. No build step. **[v1]**
- **[later]** Optional desktop wrapper (e.g. Tauri/Electron) only if browser play proves insufficient (e.g. pointer-lock spinner feel).

## 2. Core Gameplay (as in the original)

- **The Web:** A wireframe tube drawn in perspective from a single vanishing point. The near end is the **rim** (where the player sits); the far end is the **centre / horizon** (where enemies appear). The surface is divided into **lanes** (segments).
- **Tube shapes:** The original has 16 distinct shapes, cycled in order. Shapes are either:
  - **Closed** (loop; the Blaster wraps around continuously) — e.g. circle, square, plus, bow-tie, cross, triangle, clover.
  - **Open** (distinct left/right end points; the Blaster stops at the ends) — e.g. V, steps, U, flat line, heart, star, W, fan, infinity.
  - Lane count is commonly cited as 16 for open shapes and 15 for closed shapes (secondary sources; not independently verified).
- **Player (Blaster):** Claw-shaped ship that sits on the rim and *snaps* from lane to lane. It cannot leave the rim, except during the end-of-level warp.
- **Shooting:** Rapid-fire shots travel down the Blaster's current lane. Holding fire gives continuous fire, with **at most 8 shots on screen**. Player shots also destroy enemy shots in the lane.
- **Enemies:** Spawn at the far end as swirling dots, one at a time early on, then faster and in greater numbers as levels progress. See §5.4.
- **Level clear:** The level ends when all of the wave's enemies are destroyed. The Blaster then **warps** down the tube (§5.6).
- **Death:** A life is lost on contact with an enemy, an enemy shot, an active Pulsar lane, or a spike during the warp. The game ends when all lives are lost.

## 3. Visual & Audio Style

- **Look:** Vector-arcade wireframe. Bright neon lines on a solid black background; glow via `shadowBlur`/layered strokes. Line colours change per tube/level-block (original: colour scheme changes every 16 levels, see §5.7).
- **Perspective:** Pseudo-3D projection from one central vanishing point. A depth value `d` (0 = far centre, 1 = rim) maps to screen position by linear interpolation between the inner and outer vertex rings, with an optional perspective-correct scaling for enemy size.
- **Colour convention (original-inspired):** Blaster yellow; Flippers red; Tankers purple; Spikers green; Fuseballs multi-colour flicker; Pulsars magenta/white; player shots yellow/white; enemy shots red/pink. Lane web in blue (level 1) and varies by level block.
- **Audio [v1, nice-to-have]:** Synthesised with the Web Audio API only (no audio files): fire, enemy destroyed, player death, Superzapper, warp, extra life. Must start only after the first user gesture (browser autoplay policy) and have a mute key (`M`).

## 4. Technical Architecture & Constraints

- **Language:** Single-file HTML5/CSS/vanilla JS.
- **Rendering:** `<canvas>` 2D context, everything drawn procedurally from paths. No images/audio assets.
- **Game loop:** `requestAnimationFrame` with a **fixed-timestep update** (60 Hz) and an accumulator, so behaviour is identical on 60/120/144 Hz displays. Render once per frame.
- **Display:** Canvas fills the window, keeps a fixed logical aspect ratio (letterboxed), and honours `devicePixelRatio` for crisp lines. Resizes on window change.
- **State machine:** `TITLE → PLAYING → (DYING → PLAYING) → WARP → PLAYING … → GAME_OVER → TITLE`, plus `PAUSED` (`P`/`Esc`).
- **Persistence:** High score stored in `localStorage` (wrapped in try/catch; game must work without it).
- **Determinism:** RNG isolated behind one function so a seed can be set for tests.

## 5. Detailed Component Specifications

### 5.1 The Web (Geometry)
- Each tube shape is a table of outer-rim vertices (and a derived inner ring at a fixed ratio, e.g. 0.1–0.15 of the outer size) plus a `closed` flag.
- Lanes are the quads between consecutive vertex pairs; **lane lines** radiate from the inner ring to the rim. Lane count follows §2.
- Lane indexing: lane 0 is the first segment at the bottom/left of the shape; increasing index = clockwise. Closed shapes wrap (`(i+n) % n`); open shapes clamp.
- Required for v1: **at least the circle (closed), square (closed), and a V or U (open)**, selected by level. The remaining shapes are **[later]** — the data-driven shape table makes them cheap to add.

### 5.2 The Player (The Blaster)
- **Visual:** Bright yellow claw ("crab-claw" / jagged U) straddling the current lane on the rim.
- **Movement:** Snaps lane to lane. Held key repeats at a fixed rate (≈ 1 lane per 4–5 ticks) so it feels like a spinner, rather than one step per key press.
- **Controls:**
  - Left / `A`: move counter-clockwise one lane. Right / `D`: move clockwise.
  - Space (hold): fire. `Z` or `Shift`: Superzapper.
  - **Mouse (v1, recommended):** horizontal mouse movement acts as the spinner (use Pointer Lock; accumulate `movementX` into lane steps). Left click = fire, right click = Superzapper. This is the closest match to the original dial.
  - **[later]** Gamepad (analog stick / D-pad).
- **Superzapper:** Two uses per level. First use destroys **all enemies currently on the playfield** (not enemy shots, spikes, or enemies not yet spawned); second use destroys **one random enemy**. Recharges at the start of each level. Fuseballs on the rim can only be killed this way.

### 5.3 Projectiles
- **Player shots:** Spawn at the rim in the Blaster's lane and travel toward the centre at a constant speed; removed at the centre. Max 8 live at once. Each shot hits at most one enemy (or an enemy shot) and is consumed.
- **Enemy shots:** Fired by Flippers, Tankers, and Spikers up their lane toward the rim. Destructible by player shots; kill the Blaster if they reach the rim in its lane.
- **Collision method:** Per-lane overlap test on the depth axis. To prevent tunnelling at speed, test the *swept* interval `[d_prev, d_now]` of the projectile against the enemy's depth interval each tick.

### 5.4 Enemies
Introduction levels are from the original; v1 may stage them in, but the spec target is:

| Enemy | Appears | Look | Behaviour | Points |
|---|---|---|---|---|
| **Flipper** | L1 (flips from L2) | Red chevron / bow-tie "X" | Climbs a lane; flips to adjacent lanes as it climbs. On reaching the rim it **chases the Blaster along the rim**; if it touches the Blaster it **captures/kills** it. May fire shots. | 150 |
| **Tanker** | L3 | Purple rhombus | Slowly climbs a lane; splits into **two Flippers** (one in each adjacent lane) when shot or when it reaches the rim. | 100 |
| **Spiker** | L4 | Green spiral | Climbs partway up a lane laying a **spike** trail, then reverses and goes back down. Shooting spikes shortens them. Spikes are a danger during the warp. | 50 (spike: 1–3) |
| **Fuseball** | L11 | Multi-colour spark ball | Moves erratically along lane **edges**, forward/back and sideways. Cannot be shot while crossing lanes in some phases; deadly on contact. No projectiles. | 250 / 500 / 750 (higher when closer to the rim) |
| **Pulsar** | L17 | Zig-zag line | Climbs and periodically **electrifies its lane**, killing the Blaster if it is in that lane. No projectiles. | 200 |
| Fuseball Tanker / Pulsar Tanker | later levels | Tanker variants | Split into two Fuseballs / Pulsars | 100 |

- Enemies appear one at a time early on; spawn interval shortens and enemy count per wave rises with level.
- **v1 scope:** Flipper, Tanker, Spiker/spikes, Fuseball, Pulsar are all in the original; recommend **v1 = Flipper + Tanker + Spiker**, with Fuseball and Pulsar **[later]**.

### 5.5 Systems / State Management **[v1]**
- **Collision detection:** per §5.3. Blaster hit tests: enemy on the rim in the same lane (or overlapping edge for Fuseballs), enemy shot reaching the rim in the Blaster's lane, active Pulsar lane.
- **Lives:** Start with 3. Hold at most 5. Bonus life awarded at score thresholds (original values not documented in the sources reviewed; **default: every 20,000 points** — adjustable constant).
- **Death sequence:** Blaster explodes, all enemies and shots clear, brief pause, Blaster respawns on the rim with a short invulnerability window (or enemies reset to the start of the wave — see Open Questions).
- **UI overlay** (monospaced font, top of canvas): score, high score, lives (as small claw icons), current level number. Centered messages: `PRESS START`, `LEVEL n`, `GAME OVER`.
- **Starting level:** Player may begin at levels 1, 3, 5, 7, or 9 (original "SkillStep"). **[later]**

### 5.6 Level Clear & Warp
- When all enemies of the wave are destroyed (and none remain on the web), the Blaster flies **down the tube** toward the centre while the web zooms past.
- Spikes left in lanes are obstacles: hit one during the warp and you lose a life, and the warp is retried. Shooting removes them.
- After the warp the next tube shape appears with a fresh wave and a fully recharged Superzapper.

### 5.7 Level Progression
- Tube shapes cycle through the 16-shape sequence; after 16 levels the sequence repeats with a **new colour scheme and higher difficulty**. (Original: scheme changes every 16 levels; levels 65–80 are black/"invisible"; the counter caps at 99, with random shapes afterward.)
- Difficulty ramps via: enemy count, spawn rate, climb speed, enemy fire rate, and the introduction of new enemy types.
- v1: at least 8 levels with increasing difficulty, repeating cycle thereafter.

## 6. Out of Scope (for now)
- Two-player alternating mode, attract-mode demo, SkillStep bonus scoring, secret/bonus levels from later ports, rim-capturing animation fidelity, cabinet-specific features (coin, rotary hardware).

## 7. Acceptance Criteria (v1)
1. `index.html` opens and plays in Chrome and Firefox on both Windows and Linux with no console errors.
2. Game runs at the same speed on 60 Hz and 144 Hz displays.
3. Circle, square, and one open shape (V or U) are playable; Blaster wraps on closed shapes and stops at the ends of open shapes.
4. Keyboard and mouse-spinner control both work; held fire is limited to 8 shots on screen.
5. Flippers, Tankers, and Spikers behave as in §5.4; a Flipper on the rim can kill the Blaster.
6. Superzapper works as specified (all, then one random).
7. Clearing a wave triggers the warp and advances the level and tube shape.
8. Lives, score, high score (persisted), and game over/restart all work.
9. No external files, requests, or libraries.

## 8. Open Questions (to discuss)
1. Respawn rule: invulnerability after death, versus the original's reset of the wave?
2. v1 enemy set: just Flipper/Tanker/Spiker, or all five?
3. How many tube shapes in v1?
4. Mouse spinner (pointer lock) as the primary control, or keyboard-first?
5. Bonus-life interval: the default is 20,000. Do you know the value you want?
6. Sound in v1, or silent first?
7. Do you want a desktop wrapper at all, or is "a browser" the host?

## 9. Known Gaps (details still to be specified)

These are behaviours the spec does not yet define. Each needs a concrete value or rule before implementation of the related feature.

### 9.1 Gameplay numbers
- **Wave composition:** No table of enemy counts per type per level, spawn intervals, or climb speeds. Needs at least a formula (e.g. `count = base + level * k`) so difficulty can be tuned and tested.
- **Enemy shots:** No fire rate, per-enemy cap, or global cap on simultaneous enemy shots.
- **Flipper behaviour:** Flip frequency, flip direction choice, and rim-chase speed are undefined.
- **Spikes:** Shots needed to destroy a spike, maximum spike length, and whether spikes persist across the rest of the wave and into the warp.
- **Hit-box sizes:** Depth extents for enemies and projectiles (needed by the swept collision test in §5.3), and a definition of "overlapping edge" for lane-edge enemies.

### 9.2 Warp (§5.6)
- Duration, speed, and what the player controls during the warp (lane changes?).
- What "retried" means after a spike kills the Blaster: restart the warp only, or the whole level?

### 9.3 Scoring
- Points (if any) for destroying enemy shots.
- Whether the two Flippers released by a Tanker score separately from the Tanker.
- Whether Superzapper kills award points per enemy.

### 9.4 Rendering
- Whether depth `d` is linear in screen space or world space (affects apparent approach speed).

### 9.5 Game flow & input
- Game-over flow: new high score message or initials entry; which key starts a new game.
- Key-repeat start delay and mouse sensitivity setting.
- Whether mouse and keyboard may be used at the same time.
- Prevent browser default actions (e.g. Space scrolling the page, right-click context menu).
- Behaviour when Pointer Lock is lost (Esc releases the lock): auto-pause.
- Auto-pause when the tab loses focus or becomes hidden.
- Cap the fixed-timestep accumulator so a long stall does not cause a burst of updates.

### 9.6 Testing
- Define which logic is pure and unit-testable without the DOM (lane math, wrapping/clamping, swept collision, score and bonus-life thresholds), using the seedable RNG from §4.

### 9.7 Accessibility
- Limit flashing/flicker (Superzapper, Fuseball, death) to avoid photosensitivity risk; consider a reduced-effects toggle.
- Make enemy types distinguishable by shape as well as colour.
