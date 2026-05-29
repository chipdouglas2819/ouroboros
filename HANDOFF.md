# OUROBOROS — Context Handoff

A snake / katamari / roguelike hybrid built as a single-file HTML game. This document captures the full state of the project: design philosophy, how systems work, what's been tried and rejected, and what comes next.

**File:** `index.html` (single file, no build step, no framework)
**Engine:** Vanilla JS, HTML5 Canvas, Web Audio synth

---

## What the game is

You are a glowing orb in a starfield void. You drift through an infinite toroidal arena hunting smaller orbs. Each one you devour becomes a segment in your trailing tail. Bigger orbs give more — more mass, more score, more tail — but they're risky. The largest orbs you can barely eat are the most rewarding catch. Periodically, after enough mass consumed, the game offers you three random mutations to pick from.

You die when reds bite off your tail and your head shrinks past its starting size. There's no win condition — you survive as long as you can. Score is the run's currency.

The fantasy: a tiny thing that grows into a serpent, navigating its own body as much as the world.

---

## Design pillars (the three ingredients)

These are the design lenses every decision routes through. Removing or weakening any one of them breaks the feel.

**Snake.** You ARE a chain. Your tail follows your head along your actual movement path. Hitting your own tail bites you — costing every segment past the bite point. Your body is both a trophy and a hazard. The longer you grow, the more navigation matters.

**Katamari.** Your head grows visibly with each orb. The camera zooms out as you grow, so the world shrinks around you. Things that filled your screen at the start become tiny dots later. The same red that was a predator becomes prey once you're big enough. The shift in scale IS the satisfaction.

**Roguelike slot machine.** Mutations come at intervals based on consumed mass. Three random options, pick one, gameplay never truly pauses for more than a few seconds (picker freezes the game, so reading happens at your own pace). 16 mutations, 2-4 levels each, mostly mechanic-changers rather than stat boosts. The variety IS the reason to replay.

---

## Core mechanics

### Eating and growth

Orb consumption uses an "mass" model: `mass = orb.r² × (gold ? 1.6 : 1)`. Every consumption metric is mass-based:

- **Score** = `mass × 0.4`
- **Head growth** = `√(player.r² + mass × 0.45)`, capped at 140
- **Tail segments added** scales with `ratio = orb.r / player.r`:
  - ratio < 0.3 → 1 segment
  - 0.3-0.5 → 2 segments
  - 0.5-0.75 → 3 segments
  - 0.75-0.9 → 4 segments
  - 0.9+ → 5 segments
  - +1 if gold, +1 if GORGE triggers

**Why this matters:** because mass is quadratic in orb radius, a 50% larger orb is 2.25× the reward. Hunting big-but-still-eatable orbs is dramatically more efficient than picking off scraps. The "just barely smaller than me" orb is the optimal target.

### Tail as health

The game has no HP bar. Your tail length IS your health.

When a red bites you, it takes a "depth" of segments from the end of your tail proportional to its size: `depth = clamp(2 + floor((orb.r − 14) / 8), 1, 30)`. If your tail is shorter than the depth, the overflow shrinks your head by `overflow × 4`. You die when your head shrinks below 12 with zero tail.

### Snake-style positional damage

When you bite your own tail, you lose **everything past the bite point**. Brushing the very tip costs 1-2 segments. Slicing into the middle of a long body is catastrophic. This makes spatial planning real: a wide loop with a long tail can be safer than a tight U-turn with a short tail because the tail-tip is far from the head.

### Dash

Mobile: ⟿ button. Desktop: SPACE. Dash speed scales with player size (always faster than running), gives 24-48 frames of i-frames depending on SURGE level, and leaves a streak. With GHOSTLINE, your tail phases through itself for ~1s after dashing.

Default cooldown is 1.5s. SURGE reduces it. Dash is the main escape tool — without it, large reds can corner you, especially after the tail has grown.

### Wrap-around (toroidal) world

The world is a torus: walking off any edge brings you in on the opposite side. There are no corners. This was the fix for an earlier problem where chasing fleeing prey would compress them against walls — and walls were exactly where you'd hit your own tail.

All distance and direction calculations use `wrapDelta(ax, ay, bx, by)`, which returns the shortest delta between two points accounting for wrap. Rendering tiles the world in a 3×3 grid so wrap edges are seamless.

### Camera

Smooth-follow camera that zooms out based on combined head + tail size: `targetZoom = clamp(14 / max(20, headSize + tailLength × 12 × 0.15), 0.14, 0.7)`. The player stays roughly the same size on screen, but you see more world around you as you grow. Camera also follows wraps using shortest-delta logic so it doesn't snap when you cross an edge.

---

## Threat variety (red orb kinds)

All three exist from the start, weighted by difficulty (run elapsed time / 6 minutes, capped at 1.5).

### Stalker (default red)
Drifts and chases at close range. The familiar threat. ~55% of reds early, ~35% late.

### Charger (orange, smaller)
State-machine AI: idle → windup (telegraphed yellow flare and forward arrow) → lunge in a locked direction → long recovery. The windup re-aims for the first half so you can't trivially sidestep, but locks in for the lunge so you CAN dodge if you commit. Unique among threats: dodging requires *reading the animation*, not just outpacing.

### Splitter (violet/magenta, larger, slower)
Fragile-egg appearance with crack lines and inner highlights. When eaten, splits into 2-3 medium stalker shards that scatter outward with brief invulnerability. The classic "is it safe to eat this?" decision — eating it generates immediate threat where there was once one big threat.

---

## Mutations (16 total)

All change behavior, not just stats. Three random options offered each pick.

### Tail-changing (6)
- **SERPENT** (×3): Tail follows tighter, more responsive turning.
- **GHOSTLINE** (×3): Tail phases through itself for 1.5s after dashing.
- **SHED** (×3): Dash sheds 4 tail segments, damages reds in radius.
- **SPIKE** (×3): Tip of tail (last 2-4 segments) damages reds it brushes.
- **SHRINK** (×2): Tail trims itself slowly over time. Trade mass for less self-hazard.
- **COMPRESS** (×3): Tail segments pack closer together — denser snake.
- **GORGE** (×3): Every Nth orb grants a bonus tail segment (L1: every 3rd, L3: every orb).

### Combat / dash (4)
- **SURGE** (×3): -25% dash cooldown, longer dash duration.
- **BOUNCE** (×2): Dashing into reds destroys small/medium ones, ricochets you off.
- **WRATH** (×2): After taking damage, your dash leaves a damaging trail for 4s.
- **VOID** (×2): Eating a gold orb releases an expanding shockwave that destroys nearby small reds.

### Hunting (5)
- **MAGNET** (×3): Cyan orbs drift toward you when nearby.
- **KEEN** (×3): Devour orbs nearer your own size (lowers eat threshold).
- **GILDED** (×3): Some orbs spawn gold, worth far more mass.
- **ECHO** (×3): Eating an orb has 25-45% chance to spawn another nearby.
- **HARVEST** (×2): Tail tip drops a small cyan orb every few seconds.

### Pacing
Mutation threshold = `400 + (totalLevels × 200)` base mass, plus a dynamic component `player.r² × 0.6` so growing raises the bar. Capped at 2000 base. There's also a hard floor of "at least 6 orbs eaten since last pick" to prevent one giant orb from popping the picker.

Picking a mutation grants 3 free tail segments as a bonus.

---

## Visual language

The visual identity is critical. Color coding is the primary information channel.

- **Cyan** = friendly / your aura / eatable orb halos / the player's body
- **Amber/gold** = orb body fill (cyan halo around amber body for eatable food)
- **Red** = hostile (stalker)
- **Orange** = hostile, lunging (charger). Yellow flare during windup.
- **Violet/magenta** = hostile, fragile (splitter). Cracked-egg pattern.
- **Pink** = "your own" (combo flashes, tail-bite damage popups)

### HUD layout
- Top-left: TAIL (turns red and pulses when below 4), SCORE
- Top-right: time (M:SS), NEXT MUTATION progress bar, active mutations row
- Bottom-right: dash button with cooldown radial fill
- Top-center: mute toggle
- World-space: damage popups (red), gain popups (cyan/gold), milestone popups

### Off-screen indicators
Arrows at the screen edge for orbs outside view. Limited to closest 3 gold + 4 cyan + 4 red. Distance fade within each set so the nearest one stands out. Without this, the wide camera made finding orbs frustrating.

---

## Code architecture

### File structure
Single `index.html` file: HTML + inline CSS + inline JS. ~2400 lines. No build step, runs anywhere.

### Game state
All state lives in a single `G` object (lowercase G, capital throughout). Key fields:
```
G = {
  state: 'title' | 'playing' | 'gameover',
  player: { x, y, vx, vy, r, tail: [], path: [], iframes, dashCd, dashFrames, ghostFrames, wrathFrames },
  orbs: [...],
  particles: [...],
  popups: [...],
  stars: [...],
  elapsed,                         // frame count since run start
  orbsEaten, mass, score,
  massSinceMut, massToNextMut,    // mutation pacing
  orbsAtLastMut,
  mutations: { [id]: level },
  pickerActive,
  shake, flash, flashColor,
  spawnTimer,
  deathReason,
}
```

Plus globals: `WORLD_W`, `WORLD_H`, `camera`, `keys`, `joystick`, `dashRequested`, `SFX`.

### Game loop
Single `requestAnimationFrame` loop calls `update()` and `draw()`. `update()` calls `gameTick()` only when `state === 'playing' && !pickerActive`. The picker freezing gameplay is built in via this gate.

### Key functions
- `gameTick()` — frame logic. Input → movement → tail follow → orb AI → collisions → spawning → particles.
- `consumeOrb(idx, o)` — handles eating: mass, score, growth, segments, ECHO, VOID, splitter children, mutation pacing.
- `damagePlayer(o)` — handles being bitten by a red. Computes positional bite depth.
- `tailHit(seg, segIndex)` — handles biting your own tail. Cuts everything past `segIndex`.
- `tryDash(p, ix, iy)` — handles dash with size-scaled speed, GHOSTLINE, SHED, WRATH activation.
- `drawWorld()` — renders one tile of the world. Called 9× per frame to handle wrap.
- `wrapDelta(ax, ay, bx, by)` — returns shortest delta between two points across the torus. Used everywhere distance matters.
- `wrapPos(o)` — keeps an orb's position within world bounds.

### Tail rendering
Tail uses a path-following model. Each frame, the head's current position is appended to `p.path[]`. The tail-follow loop walks backward through `p.path[]` from the head, accumulating distance, placing each segment at a fixed pixel offset (`p.r × 0.85`) from its predecessor. A safety snap kicks in if any segment drifts more than 4 baseGaps from its target — prevents disconnection during dashes or wraps.

The path array is trimmed periodically to bound memory: only kept long enough for the longest segment to use.

### Sound
Simple Web Audio synth in `SFX` object. No samples — all sound is `OscillatorNode + GainNode` envelopes plus filtered noise bursts. Triangle for melodic eats, square for impacts, sawtooth for dash, lowpass-filtered noise for damage.

---

## What was tried and rejected

These are documented to prevent rebuilding things that didn't work.

### Forced clearance per floor
Original design: each floor required eating every orb to descend, then picking a perk. Felt like busywork — the satisfying part of every floor (snowball mid-game) ended in a tedious cleanup phase chasing weak stragglers.
**Replaced with:** continuous spawning + mass-based mutation pacing.

### Floor structure with a final boss
Tried 8 floors → "VOID HEART" boss. Felt artificial because the runs aren't long enough to support a clear narrative arc, and the boss design felt arbitrary on top of a survival game. Imposing structure on what wanted to be endless.
**Replaced with:** endless run with elapsed-time difficulty curve.

### HP bar
Standard HP bar with healing items. Created complexity (max HP, regen, healing pickups, integrity) without clarity — players had to track an abstract number disconnected from what they could see.
**Replaced with:** tail length IS health. Damage shrinks the tail visibly. Healing happens by eating, which is already the core action.

### Combo system
Multiplicative chain bonus on consecutive eats. Existed in HUD, dimmed when broken, gave +4% score per stack. Players didn't notice it. It didn't drive any decision — combo timer was 4s, easily maintained accidentally, and score isn't tracked outside death screen.
**Replaced with:** removed entirely. One fewer thing to ignore.

### Pure stat-boost mutations (Velocity, Aegis, Genesis, Bloom)
Boring picks. When the slot machine offered three stat boosts, no choice felt meaningful.
**Replaced with:** mechanic-changing mutations (BOUNCE, ECHO, HARVEST, VOID, SHED, SPIKE etc.).

### Aggressive predator AI
Predators chased at 90% player speed and engaged from across the screen. Players couldn't escape simply by growing faster.
**Replaced with:** predators capped at 50-78% player speed, with quadratic chase ramp (lazy at distance, only aggressive up close). Player speed scales linearly with size, leaving threats behind.

### Walls
Original wall-bounded play area. Caused two problems: prey trapped in corners (too easy), and your own tail forced you into walls (death traps).
**Replaced with:** wrap-around world. No corners exist.

### Instant death from hitting your tail
Classic snake. Felt too punishing for a real-time chaotic game with knockbacks and crowded screens.
**Replaced with:** positional damage. Hitting your tail is costly but survivable — you lose everything past the bite point.

### Score-based mutation pacing
Mutations every X orbs. Got too frequent because big orbs gave many "orb count" credit. Players reported pickers firing constantly mid-game.
**Replaced with:** mass-based with dynamic threshold scaling on player size. Plus a "minimum 6 orbs eaten" floor so single giant orbs can't trigger it.

---

## Recently fixed (per latest audit)

These were the last batch of bugs identified and resolved:

1. `damagePlayer` knockback now uses `wrapDelta` for direction (no more wrap-edge launches).
2. `dashRequested` cleared when picker opens (no buffered dashes after picking).
3. Joystick state cleared when picker opens.
4. "−HEAD" popup replaced with sensible head-shrink text.
5. New tail segment placement uses last segment's actual position, not stale `pathIdx`.
6. ECHO now checks for player/tail collision before placing spawn.
7. GILDED rebalanced to `0.07 + 0.05 × level` (12-22% per cyan).
8. Single big orb can't pop picker (existing 6-orb floor + dynamic threshold).
9. WRATH active state now has visual feedback (player aura tint).
10. Charger lunge re-aim shortened — only first ~10 frames of windup.

---

## Known open issues / next priorities

In rough order of impact:

### Score has no purpose
Currently it's just a number that goes up. Death screen shows it. No high-score persistence, no comparison, no tier/grade. Should at minimum:
- localStorage-persisted personal best
- Show on title screen ("Best: 12,450")
- Maybe rank tiers (S/A/B/C) shown on death

### No surprise after first run
After 2-3 runs you've seen all 16 mutations and all 3 threat types. Replay value depends entirely on score chasing — which only works if score has weight. Tied to the issue above.

### Mutation synergies
Mutations are independent. There's no reward for theming a build. Possibilities:
- Build archetypes (offensive / defensive / scoring) with bonuses for picking 3 of one tag
- Specific named pairings ("Ghostline + Shed = shed segments deal extra damage")
- Currently each mutation works in isolation — picking 3 random ones is fine but never feels like a *build*

### Event-based pacing
Difficulty rises smoothly with elapsed time, but the run has no narrative shape — no exhale moments, no climactic hunts. Possibilities:
- Periodic "blooms" — orb-spawn flurries every 2 minutes
- "Hunt" events — a single faster predator spawns to chase you for 30 seconds
- Quiet phases between intensities

### Spatial features
The world is a flat featureless plane. All space plays the same. Possibilities:
- Drift currents (vector fields that push things)
- Bonus zones (rare 2× mass spots)
- Asteroids (large neutral obstacles to navigate around)

Pick at most 1-2 — the wrap-around playfield doesn't need much terrain.

### No persistence
Each run is identical to the first. No meta-progression. Even minimal additions help:
- localStorage high score
- Daily seed for shareable challenges
- Unlock starting mutations after milestones

### Charger and splitter visual distinction at small zoom
At low zoom (large player), the colors do most of the work, but the unique visual elements (charger arrow, splitter cracks) become hard to see. Worth tuning if user feedback mentions it.

---

## Open design questions

These shape the project's identity and don't have right answers yet.

**Should the game have an end?** Currently endless. Could add a "hidden depth" — eat your own tail tip as a triumphant ending. Or stay endless with leaderboards.

**Should reds get more types?** Three feels right but late game gets thin. A fourth type (e.g. an orbit-er that circles you) could add variety without overcomplication.

**Should mutations remove orbs from the spawn pool when picked?** Currently they can be re-rolled, just bumping their tier. If we filter, builds become more focused but luck-dependent.

**Is the tail-as-health design teaching itself well?** The relationship between tail length, damage, and death is implicit. Some players might never realize damage trims the tail until they die from it. Could use a tutorial, or first-damage popup.

---

## Tone & visual identity

The game's aesthetic is deliberate: deep navy void, glowing cyan player, warm amber food, cool blue particles, uppercase sans-serif (Syncopate) for HUD typography, monospace (DM Mono) for numbers, minimal UI chrome.

Keep this. It does work that 90% of bedroom-arcade games' visual styling fails to do — it makes a tiny canvas game feel intentional and considered. Don't add color noise, don't add decorative pixel art, don't add screenshake everywhere. The restraint is the look.

---

## How to develop further

The current state is "design is settled, polish remains." Don't redesign mechanics unless playtesting reveals deep issues. Do:

- Add persistence (highscore is the smallest, highest-impact change)
- Tune mutation balance based on which get picked / which dominate runs
- Test on actual mobile devices for touch responsiveness
- Test long runs (5+ minutes) for late-game pacing collapse
- Profile particle/render performance on slower phones

The game is small, finished-feeling, and self-contained. It doesn't need a framework, a publisher, or an art rework. It does need eyes on it to find what playtesting reveals about *real* friction — there are usually 2-3 issues no amount of internal review will surface.
