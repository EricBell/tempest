# Tempest Clone — Requirements

## 0. Project Overview

A browser-based clone of Atari's 1981 arcade game **Tempest** (designed by Dave Theurer; first Atari colour-vector game). The player pilots a claw-shaped "Blaster" around the near rim of a wireframe tube (the "web") seen in perspective, shooting enemies that climb up the lanes from the far end. Clear every enemy to warp into the next tube.

Fidelity goal: capture the *feel* and rules of the original arcade game, not a pixel-exact ROM port. Items marked **[v1]** are required for the first playable version; **[later]** items are planned but deferred.

## 1. Implementation & Hosting

- Delivered as a single `index.html` (HTML5 + CSS + vanilla JS, no external dependencies).
- The "host" on Windows and Linux is a modern web browser (Chrome, Edge, Firefox). Same file, same code, opened directly via `file://` or served by any static server. No build step. **[v1]**
- **Decided:** a browser is the host. No desktop wrapper is planned; revisit only if browser play proves insufficient.

## 2. Core Gameplay (as in the original)

- **The Web:** A wireframe tube drawn in perspective from a single vanishing point. The near end is the **rim** (where the player sits); the far end is the **centre / horizon** (where enemies appear). The surface is divided into **lanes** (segments).
- **Tube shapes:** The original has 16 distinct shapes, cycled in order. Shapes are either:
  - **Closed** (loop; the Blaster wraps around continuously) — e.g. circle, square, plus, bow-tie, cross, triangle, clover.
  - **Open** (distinct left/right end points; the Blaster stops at the ends) — e.g. V, steps, U, flat line, heart, star, W, fan, infinity.
  - Every shape is defined by **16 points**. A **closed** shape joins the last point back to the first, giving **16 lanes**. An **open** shape does not, giving **15 lanes**. (Source: the original ROM level data as documented by the TempEd/Tempest Code Project; this corrects an earlier note that had them reversed.)
- **Player (Blaster):** Claw-shaped ship that sits on the rim and *snaps* from lane to lane. It cannot leave the rim, except during the end-of-level warp.
- **Shooting:** Rapid-fire shots travel down the Blaster's current lane. Holding fire gives continuous fire, with **at most 8 shots on screen**. Player shots also destroy enemy shots in the lane.
- **Enemies:** Spawn at the far end as swirling dots, one at a time early on, then faster and in greater numbers as levels progress. See §5.4.
- **Level clear:** The level ends when all of the wave's enemies are destroyed. The Blaster then **warps** down the tube (§5.6).
- **Death:** A life is lost on contact with an enemy, an enemy shot, an active Pulsar lane, or a spike during the warp. The game ends when all lives are lost.

## 3. Visual & Audio Style

- **Look:** Vector-arcade wireframe. Bright neon lines on a solid black background; glow via `shadowBlur`/layered strokes. Line colours change per tube/level-block (original: colour scheme changes every 16 levels, see §5.7).
- **Perspective:** Pseudo-3D projection from one central vanishing point. A depth value `d` (0 = far centre, 1 = rim) maps to screen position by linear interpolation between the inner and outer vertex rings, with an optional perspective-correct scaling for enemy size.
- **Colour convention (original-inspired):** Blaster yellow; Flippers red; Tankers purple; Spikers green; Fuseballs multi-colour flicker; Pulsars magenta/white; player shots yellow/white; enemy shots red/pink. Lane web in blue (level 1) and varies by level block.
- **Audio (issue #3):** Synthesised with the Web Audio API only (no audio files). Effects: fire, lane change, spike hit, enemy shot, Tanker split, enemy destroyed (a distinct sound per enemy type), Pulsar pulse, player death, Superzapper (different for the first and second use), warp, extra life, level start, game over. No background music. The `AudioContext` is created only after the first user gesture. Controls: `M` mutes, `-` / `+` (also numpad, and `=`) change the volume in 10% steps, and a slider on the title and pause screens sets it. Volume defaults to 50% and, with the mute state, persists in `localStorage` (§4). The title and pause screens show the controls and the current volume or MUTED; a brief readout appears when it changes during play.

## 4. Technical Architecture & Constraints

- **Language:** Single-file HTML5/CSS/vanilla JS.
- **Rendering:** `<canvas>` 2D context, everything drawn procedurally from paths. No images/audio assets.
- **Game loop:** `requestAnimationFrame` with a **fixed-timestep update** (60 Hz) and an accumulator, so behaviour is identical on 60/120/144 Hz displays. Render once per frame.
- **Display:** Canvas fills the window, keeps a fixed logical aspect ratio (letterboxed), and honours `devicePixelRatio` for crisp lines. Resizes on window change.
- **State machine:** `TITLE → PLAYING → (DYING → PLAYING) → WARP → PLAYING … → GAME_OVER → TITLE`, plus `PAUSED` (`P`/`Esc`).
- **Persistence:** Scoring is **off by default**. Only when the page is loaded with `?keep-score=1` does the game keep **high scores** in `localStorage` (wrapped in try/catch; game must work without it). Without the flag no *score* data is read or written. The **audio settings** (volume and mute) are separate: they persist under their own key (`tempest_audio`) regardless of the flag.
- **Determinism:** RNG isolated behind one function so a seed can be set for tests.

## 5. Detailed Component Specifications

### 5.1 The Web (Geometry)
- Each tube shape is a table of outer-rim vertices (and a derived inner ring at a fixed ratio, e.g. 0.1–0.15 of the outer size) plus a `closed` flag.
- Lanes are the quads between consecutive vertex pairs; **lane lines** radiate from the inner ring to the rim. Lane count follows §2 (16 closed / 15 open).
- Lane indexing: lane 0 is the first segment at the bottom/left of the shape; increasing index moves **right at the bottom of the screen** (so Right = +1 lane; on the circle this is counter-clockwise on screen, which keeps Left/Right matching the screen at the Blaster's starting position). Closed shapes wrap (`(i+n) % n`); open shapes clamp.
- Required for v1 (decided): **circle (closed), square (closed), and V (open)**, cycling by level. The remaining shapes are **[later]** — the data-driven shape table makes them cheap to add.

### 5.2 The Player (The Blaster)
- **Visual:** Bright yellow claw ("crab-claw" / jagged U) straddling the current lane on the rim.
- **Movement:** Snaps lane to lane. Held key repeats at a fixed rate (≈ 1 lane per 4–5 ticks) so it feels like a spinner, rather than one step per key press.
- **Controls:**
  - Left / `A`: move counter-clockwise one lane. Right / `D`: move clockwise.
  - Space (hold): fire. `Z` or `Shift`: Superzapper.
  - **Mouse (v1, opt-in):** keyboard works by default. Clicking the canvas enables the mouse spinner, where horizontal mouse movement moves the Blaster (use Pointer Lock; accumulate `movementX` into lane steps). Left click = fire, right click = Superzapper. This is the closest match to the original dial.
  - **[later]** Gamepad (analog stick / D-pad).
- **Superzapper:** Two uses per level. First use destroys **all enemies currently on the playfield** (not enemy shots, spikes, or enemies not yet spawned), killing them one after another over a short burst (~19 ticks, from the farthest-up enemy down); second use destroys **one enemy** (~5 ticks). No new enemies spawn while a zap is running. Recharges at the start of each level. Fuseballs on the rim can only be killed this way.

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
| **Spiker** | L4 | Green spiral | Climbs a lane **almost to the rim** laying a **spike** trail, turns around just short of the rim, then retreats. If no enemies are left to spawn, it turns into a Flipper-carrying Tanker. Shooting a spike lowers it to the shot position. Spikes are a danger during the warp. | 50 (spike: 1–3) |
| **Fuseball** | L11 | Multi-colour spark ball | Moves erratically along lane **edges**, forward/back and sideways. Cannot be shot while crossing lanes in some phases; deadly on contact. No projectiles. | 250 / 500 / 750 (higher when closer to the rim) |
| **Pulsar** | L17 | Zig-zag line | Advances while far from the Blaster and retreats when close. It **pulses** on a beat; while pulsing, the Blaster dies if it is in the Pulsar's lane within a set distance. No projectiles. | 200 |
| Fuseball Tanker / Pulsar Tanker | later levels | Tanker variants | Split into two Fuseballs / Pulsars | 100 |

- Enemies appear one at a time early on; spawn interval shortens and enemy count per wave rises with level.
- **v1 scope:** Flipper, Tanker, Spiker/spikes, Fuseball, Pulsar are all in the original; **Decided: v1 = Flipper + Tanker + Spiker.** Fuseball and Pulsar (and their Tanker variants) were added afterwards (issue #2); see §11.

### 5.5 Systems / State Management **[v1]**
- **Collision detection:** per §5.3. Blaster hit tests: enemy on the rim in the same lane (or overlapping edge for Fuseballs), enemy shot reaching the rim in the Blaster's lane, active Pulsar lane.
- **Lives:** Start with 3. Hold at most 6 (the ROM caps extra lives at 6). Bonus life awarded at score thresholds (original is a DIP-switch option: 10k / **20k (default)** / 30k / 40k / 50k / 60k / 70k / none; use **every 20,000 points** as an adjustable constant).
- **Death sequence (decided: original behaviour):** the Blaster is blown apart, all enemies and shots clear, pending-enemy timers reset and the wave starts over. No invulnerability window.
- **UI overlay** (monospaced font, top of canvas): score, high score, lives (as small claw icons), current level number. High scores (only with `?keep-score=1`): top three with 3-letter initials, stored in `localStorage`; the `HI` line, the title-screen table and the initials screen are hidden otherwise. Centered messages: `PRESS START`, `LEVEL n`, `GAME OVER`.
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
3. Circle, square, and V are playable; Blaster wraps on closed shapes (16 lanes) and stops at the ends of open shapes (15 lanes).
4. Keyboard and mouse-spinner control both work; held fire is limited to 8 shots on screen.
5. Flippers, Tankers, and Spikers behave as in §5.4; a Flipper on the rim can kill the Blaster.
6. Superzapper works as specified (all, then one random).
7. Clearing a wave triggers the warp and advances the level and tube shape.
8. Lives, score, and game over/restart work. With `?keep-score=1`, top-3 high scores with initials persist; without it, no high-score UI appears and no score data touches `localStorage` (only the separate `tempest_audio` key is used).
9. No external files, requests, or libraries.

## 8. Decisions

Settled in discussion; these override anything earlier in the document that conflicts.

| Topic | Decision |
|---|---|
| Enemies | v1: Flipper + Tanker + Spiker. Added later (issue #2): Fuseball (L11+), Pulsar (L17+), Fuseball/Pulsar Tankers (L20+) |
| Death | Original behaviour: clear enemies/shots, reset pending timers, restart the wave; no invulnerability |
| Controls | Keyboard by default; mouse spinner (Pointer Lock) opt-in by clicking the canvas |
| Sound | Synthesised Web Audio (issue #3): effects only, no music. `M` mutes, `-`/`+` and a slider set volume (default 50%), settings persist in `localStorage`, on-screen hints |
| Tube shapes | Circle, square, V in v1 |
| Host | Browser only; no wrapper |
| Bonus life | Every 20,000 points |
| High scores | Opt-in via `?keep-score=1`: top three with 3-letter initials, in `localStorage`. Off by default: no table, no initials screen, no HUD `HI` line, no storage access |
| §10 proposals | All accepted as defaults; speeds and sizes are tuned by feel during play |

## 9. Reference Data from the Original

Taken from the reverse-engineered 6502 source (`tempest.a65`, final ROM revision) and the Tempest Code Project / TempEd notes. Treat as the target; v1 may simplify. Level numbers are 1-based. The ROM stores speeds as opaque tick counts, so speeds are given as trends and need tuning by feel.

### 9.1 Hard limits
| Limit | Value |
|---|---|
| Player shots on screen | 8 |
| Enemies on screen at once | 7 (Tanker loads count double toward the limit) |
| Enemy shots in flight | hard cap 4; per-level limit starts at **2** (L1–3) and rises to **3–4** later |
| Pending (not yet spawned) enemies | up to 64 queued |
| Lives held | **max 6** |
| Levels | 99 (counter caps; L99 picks a random tube) |
| Lanes | 16 closed / 15 open |

### 9.2 Enemies per wave (total spawned in the wave)
Levels 1–16: **10, 12, 15, 17, 20, 22, 20, 24, 27, 29, 27, 24, 26, 28, 30, 27**. Higher levels continue upward (roughly 27 → 43 by L81+).

### 9.3 Max of each type alive at once
| Enemy | Levels → max |
|---|---|
| Flipper | 1–4: 4 · 5–16: 5 · 17–19: 3 · 20–25: 4 · 26+: 5 |
| Tanker | 1–2: 0 · 3: 1 · 4: 0 · 5: 1 · 6–16: 2 · 17–32: 1 · 33–44: 2 · 45+: 3 |
| Spiker | 1–3: 0 · 4: 2 · 5: 3 · 6–10: 4 · 11–16: 3 · 20–25: 2 · later mostly 1–2 |
| Fuseball | 0 until L11 · 11–16: 1 · 33–39: 4 · 40+: 3 |
| Pulsar | 0 until L17 · 17: 5 · 18: 3 · 19+: ~2 · 33+: 3 |

Minimum guaranteed per wave: Flipper ≥1 on L1–4; Tanker ≥1 from L3 (not L4); Spiker ≥1 on L4, ≥2 on L5–16; Fuseball ≥1 from L11; Pulsar ≥3 on L17–32.

Spawn selection: new enemies come from a pending queue with countdown timers. A Spiker is chosen preferentially when the shortest spike in the tube is short. Spawning is suspended while the Superzapper runs.

### 9.4 Enemy behaviour (from the ROM routines)
- **Flipper:** climbs, then flips lane-to-lane using one of several per-level *movement programs* (e.g. "flip twice, then three times"). A flip runs until its angle completes. On open tubes it reverses direction at the end lane. On reaching the rim it heads toward the Blaster's lane. A non-retreating enemy that shares the Blaster's lane at the rim kills it.
- **Tanker:** carries Flippers, Fuseballs or Pulsars. Splits into two when shot or when it reaches the point just short of the rim; the second piece is placed in the adjacent lane.
- **Spiker:** grows its lane's spike as it climbs; turns around just short of the rim; hops lanes at the far end; becomes a Flipper-carrying Tanker if nothing else is pending.
- **Fuseball:** moves at its own speed with randomised direction changes along lane edges; once nothing else is pending it rushes to the rim. Kills the Blaster on contact (same lane and depth). Hit tolerance is a fixed small value.
- **Pulsar:** advances while far from the Blaster, retreats when close, pulses on a beat; the Blaster dies if it is in the Pulsar's lane within range while it pulses.
- **Enemy shots:** an enemy only fires once it has climbed a minimum distance from the far end. The chance of firing depends on how many enemy shots are already in flight, via a table `$00,$E0,$F0,$FA,$FF` compared against a random byte; how the index maps is not fully clear from the source, so tune by feel. After firing, a per-level hold-off timer must elapse.
- **Hit tolerance:** every enemy type has its own hit tolerance (how close a shot must be); Fuseball is fixed and small, others scale with level.

### 9.5 Enemy shots (timing)
- Hold-off between shots: ~80 ticks on L1, decreasing by ~3 per level to ~23 at L20, then ~20 (L21–64), then ~10 (L65+).
- Shot speed is nearly constant across levels (slightly faster at L81+).

### 9.6 Spikes
- No spikes on L1–3. From L4 the starting spike height in each lane grows with level (short at L4, much longer by L13+).
- A Spiker extends its lane's spike. A player shot that reaches a spike **lowers the spike to the shot's position** and scores **1 point**; a shot is removed after hitting a spike twice.
- During the warp, if the spike in the Blaster's lane is taller than the Blaster's current depth, the Blaster dies.

### 9.7 Speeds and the warp
- Flipper speed rises steadily with level until ~L64, then plateaus. Pulsars and Fuseballs have their own tables. Spikers move at a near-constant speed, a little slower after L15.
- **Warp:** the Blaster moves down the tube from the rim to the far end. Its speed accelerates every tick; the base acceleration is `max(12, level) × 4` plus a constant, capped for low levels. The warp ends when the Blaster reaches the far end.

### 9.8 Scoring, lives, high scores
- Points: Spike 1 per shot · Spiker 50 · Tanker 100 · Flipper 150 · Pulsar 200 · Fuseball 250/500/750 (closer to the rim = more). Score is 6 BCD digits.
- Lives: 3 default (DIP options 2/3/4/5), max 6. Bonus life every 20,000 (DIP options 10k–70k or none).
- Superzapper: 2 uses per level (first ~19 ticks, second ~5 ticks).
- High scores: top three, each with a **3-letter initials** entry. "GAME OVER" is shown for ~40 ticks before the high-score check.
- DIP difficulty (Easy/Medium/Hard): Easy = one fewer enemy shot and slower Flippers; Hard = more enemy shots (up to 3), faster Flippers, more enemies per wave, more Pulsar fire. **[later]**

### 9.9 Player
- Position is continuous: 16 sub-steps per lane (`position >> 4` = lane). Spinner input per frame is clamped to ±31 sub-steps. Closed tubes wrap; open tubes clamp at the ends.
- No explicit fire-rate timer; the rate is limited by the 8 shot slots and a per-frame button debounce.
- The player-shot colour shows how many are in flight: **yellow** under 6, **blue** 6–7, **red** at 8.
- Death: the Blaster is blown apart (shot hit) or "spiked/pulsared apart" (pieces fly out). After death all shots and enemies are removed and pending timers reset.

### 9.10 Tube order and colour
- Tube order (index into the 16 stored shapes): `0,1,2,3,4,5,6,7,13,9,8,12,14,15,10,11`. Six of the shapes are open. 
- 6 colour "cycles" of 8 entries (shots, claw, tankers, flippers, pulsars, spikers, level/web, info). Cycle 1 is the blue levels, cycle 2 the red levels, and so on.

## 10. Gap Defaults (accepted)

Items the source did not settle. Each default below is accepted for v1 and may be tuned by feel during play.

### 10.1 Gameplay
- **Exact speeds** in lanes/sec or depth/tick. **Default:** tune by feel, keeping the trends in §9.7.
- **Flipper movement programs:** the byte patterns aren't decoded. **Default:** flip toward the Blaster after N ticks of climbing, N shrinking with level.
- **Per-type hit tolerances** beyond the Fuseball. **Default:** enemy depth extent ≈ 4% of the tube, shots a point.
- **Pulsar beat/fire intervals** in real time.
- **Fuseball lane-edge movement probabilities.**
- **Whether wave size resets after death** (the ROM resets the pending timers; the wave count likely restarts).

### 10.2 Warp (§5.6)
- Duration and lane control during the warp. **Default:** ~2–3 seconds, the Blaster can still change lanes, and a spike hit costs a life and restarts the warp only.

### 10.3 Scoring
- Points for destroying enemy shots (not found), and for each Tanker-spawned Flipper (assume normal values).
- Start-level bonus table is **[later]**.

### 10.4 Rendering
- Depth mapping. **Default:** a perspective projection with a fixed camera; depth is linear in world space.

### 10.5 Game flow and input
- Key-repeat delay and mouse sensitivity.
- Mouse and keyboard together: allowed.
- Prevent browser defaults (Space scroll, right-click menu).
- Pointer Lock loss (Esc): auto-pause. Auto-pause on tab hidden; cap the fixed-timestep accumulator.
- High-score entry: the original uses top-3 with 3-letter initials. **Default:** implement it, stored in `localStorage`.

### 10.6 Testing
- Define which logic is pure and unit-testable without the DOM (lane math, wrapping/clamping, swept collision, score and bonus-life thresholds), using the seedable RNG from §4.

### 10.7 Accessibility
- Limit flashing/flicker (Superzapper, Fuseball, death); offer a reduced-effects toggle.
- Make enemy types distinguishable by shape as well as colour.

## 11. Implementation Notes (index.html, v1)

Where the first build deviates from, or fills in, this spec:

- **Lane direction:** see §5.1 — Right key = +1 lane = rightward at the bottom of the screen.
- **Death:** enemies on the web are cleared; wave enemies that were still alive go back to the front of the spawn queue (kills stay kills), then spawning restarts after ~1.5 s. After a death *in the warp* only the warp is retried, after a 2.5 s pause.
- **Warp:** the Blaster can change lanes and fire during the warp. It starts automatically 1 s after the web is clear. Initial spikes (L4+) are short, so shooting one clears a lane.
- **Spikes:** each hit on a spike shortens it by 10% of the tube depth and scores 1 point; the shot is consumed.
- **Superzapper:** first use kills enemies one by one over ~19 ticks, second use kills one. Tankers killed by the zapper do not split. Spawning pauses while zapping. Charges show as two cyan rings under the score.
- **Pause:** `P` toggles; `Esc`, losing Pointer Lock, tab blur and tab hide all pause (never unpause).
- **Fixed-timestep loop:** 60 Hz, accumulator capped at 250 ms. Seed the RNG with `?seed=123`. `window.TempestCore` and `window.__tempest` expose the logic for testing.
- **Fuseball (issue #2):** sits on a lane *edge* and hops sideways across lanes (18 ticks per hop). It can only be shot **while crossing a lane** (the middle 60% of a hop) and never at the rim, where only the Superzapper kills it. Mid-tube it hops, reverses and climbs erratically; once nothing is left to spawn it rushes to the rim, then hunts the Blaster along the edges. It kills the Blaster when it is at the rim on an edge next to the Blaster's lane or crossing it. Score 250 / 500 / 750 by depth (far / middle / near). Colour cycles at ~5 Hz and the spark jitter at 15 Hz (§10.7).
- **Pulsar (issue #2):** climbs to 92% depth, retreats to 45%, and repeats. It flips lanes toward the Blaster (70%) or randomly. It pulses for 28 of every 90 ticks; while pulsing the whole lane from the Pulsar down to the rim is electrified and kills the Blaster if it is in that lane. Score 200. Never shoots.
- **Tanker variants (issue #2):** from L20 some Tankers carry Fuseballs (a pair, on the Tanker's two lane edges) or Pulsars (in the neighbouring lanes). The payload shows as the colour of the inner cross: red = Flippers, orange = Fuseballs, magenta = Pulsars. Payload enemies don't count against the per-type caps. A Tanker killed by the Superzapper does not split.
- **Caps and minimums:** Fuseball and Pulsar maxima and per-wave minimums follow §9.3. Tanker/Spiker/Fuseball/Pulsar shares of the remaining wave slots are tuned by feel.
- **Audio (issue #3):** `Game` only emits events (`level`, `fire`, `lane`, `kill {type}`, `eshot`, `split`, `spike`, `pulse`, `death`, `zap {n}`, `warp`, `life`, `gameover`) through an `onEvent` callback; it contains no Web Audio code. `AudioEngine` (same file, separate class, takes a context factory so it can be tested with a fake) turns events into synthesised sounds behind one master gain node (`gain = volume²`, 0 when muted). Fire and lane sounds are rate-limited and low-priority sounds are dropped above 10 live voices (hard cap 24). The context is suspended while paused or the tab is hidden. The slider is an HTML range input shown only on the title and pause screens (the mouse is free then) and blurs itself after use so it never takes Space or arrow keys. Audio randomness (noise) uses `Math.random`, never the seeded game RNG.
- **Not built (per §8):** the other 13 tube shapes, gamepad, start-level selection.
