# PokeJam shot numbers: real vs guessed

Source: the kungFuDOOM/PokeBasketball repo (branch claude/poke-jam), commit `b801667` (read only, unchanged). Paths are relative to `src/`. Units are feet and seconds.

## The main finding

**The game has no release-angle input and no power input.** There is also no ball-vs-rim collision.

At release the game does this, in order (`game/game.ts:990-1070`):

1. Grade the timing.
2. Roll make or miss.
3. Pick a target for that result.
4. Set flight time from distance.
5. Solve the velocity that lands on the target.

So angle and speed are outputs, and the arc can't change the result. The simulator shows them as `SOLVED`. Moving those sliders switches to a hand-set mode labelled `SIM`, and make % stays the same in that mode because the real formula has no such term.

**The "power/timing meter" is only a timing meter.** It adds no power.

## Checked against the real engine

`node src/verify.mjs` bundles the source to a temp folder and drives the real `Game` class. Last run:

| Check | Cases | Largest difference |
|---|---|---|
| Make probability, jump shots and layups (threshold found by binary search on the real roll) | 3,922 random inputs | 4.7e-10 (search resolution) |
| Made-shot arc: `x0 y0 z0 tx ty tz vx vy vz T` | 3,922 | 0 |
| Jump height at release, from a real button press through `update()` | 3 meter values, plus hold to landing | under 1e-9 |
| Layup release height, through `update()` | 1 | under 1e-9 |

Not checked against the engine: dunk odds and path, and the miss and airball target rolls. These are ported by reading the code. Dunk odds use `contestOf`, which the jump-shot checks did exercise.

## Real numbers

### Court, hoop, ball

| Number | Value | Where |
|---|---|---|
| Court length, width | 94, 50 | `game/types.ts:8-9` |
| Hoop centre y | 25 | `game/types.ts:10` |
| Rim height | 10 | `game/types.ts:11` |
| Rim radius | 0.75 | `game/types.ts:12` |
| Ball radius | 0.45 | `game/types.ts:14` |
| Hoop x (attacking right, left) | 88.75, 5.25 | `game/types.ts:18` |
| Three-point radius | 23.75 | `game/types.ts:30`, `game/render.ts:386` |
| Corner-three rule | abs(dy) ≥ 22 and abs(dx) < 9 | `game/types.ts:29` |
| Paint | 19 long, y 17 to 33 | `game/render.ts:332-334` |
| Free-throw and centre circle radius | 6 | `game/render.ts:378, 384` |
| Restricted arc radius | 4 | `game/render.ts:391` |
| Corner line y | 3 and 47 | `game/render.ts:388-389` |
| Wood stripe | 1.25 band every 2.5 | `game/render.ts:324-326` |
| Backboard plane | 4 from baseline | `game/render.ts:624` |
| Backboard extent | y 22 to 28, z 9 to 13.5 | `game/render.ts:641` |
| Backboard inner square | y 24 to 26, z 10 to 11.5 | `game/render.ts:652` |
| Stanchion | 3 behind baseline, top 12.5 | `game/render.ts:627-629` |
| Net length, lower radius | 1.6, 0.55 × rim | `game/render.ts:664, 671` |

### Gravity and motion

| Number | Value | Where |
|---|---|---|
| Gravity | 32 ft/s² | `game/types.ts:13` |
| Simulation tick | 1/60 s | `game/sync.ts:48`, `screens/play.ts:18` |
| Player integration (`vz` first, then `z`) | semi-implicit Euler | `game/game.ts:1450-1451` |
| Ball floor bounce | vz × 0.6, vx and vy × 0.88, stops under 2.5 | `game/game.ts:1649-1654` |
| Net drop | 9 ft/s down to z 6.8, then vz -4, vx and vy ±1.5 | `game/game.ts:1500-1505` |

### Timing meter

| Number | Value | Where |
|---|---|---|
| Ideal release | 0.8 | `game/game.ts:11` |
| Meter fill time | 0.62 s | `game/game.ts:13` |
| Forced release | meter ≥ 1.25, or landing after 0.15 s | `game/game.ts:905-906` |
| Green window half-width | (0.022 + SHT/100 × 0.034) × difficulty × catch-up × (fire 1.7) | `game/game.ts:320-323` |
| Difficulty green multiplier: Rookie, Pro, All-Star, Legend | 1.9, 1.35, 1.1, 0.85 | `game/types.ts:46-49` |
| Grade bands: perfect, good, early/late, very | ≤ w, ≤ 2.3w, ≤ 4.5w, beyond | `game/game.ts:995-998` |
| Grade text | EXCELLENT, GOOD RELEASE, SLIGHTLY EARLY/LATE, VERY EARLY/LATE | `game/hud.ts:14-20` |
| CPU timing noise (listed, not modelled) | max(0.05, 0.09 + (100-SHT) × 0.0018 - ...) | `game/game.ts:959-960` |

**Derived, and confirmed in the engine:** a jump lasts 39 ticks (0.65 s), so the meter really stops at **1.048**. The 1.25 cap can't be reached by a shot started on the ground. One tick moves the meter 0.0269, so timing is coarser in the game than on the simulator's continuous slider. The slider's range is 0 to 1.048.

### Make %

| Number | Formula | Where |
|---|---|---|
| Base under 8 ft | 0.55 + INS × 0.004 | `game/game.ts:985` |
| Base, two | clamp(0.2 + SHT × 0.0042 - (d-8) × 0.007, 0.1, 0.75) | `game/game.ts:986` |
| Base, three | clamp(0.1 + SHT × 0.0037 - max(0, d-24) × 0.03, 0.02, 0.6) | `game/game.ts:987` |
| Perfect | (three 0.86, two 0.94) - (1-contest) × 0.55 | `game/game.ts:1006` |
| Good | base × contest + 0.06 | `game/game.ts:1007` |
| Early or late | base × contest × 0.8 | `game/game.ts:1008` |
| Very early or late | base × contest × 0.3 | `game/game.ts:1009` |
| Effort factor | × (0.7 + 0.3 × effort) | `game/game.ts:1011` |
| Effort | 1 at energy ≥ 0.55, else 0.82 + 0.18 × energy/0.55 | `game/game.ts:252-254` |
| On fire | min(0.97, p + 0.25) | `game/game.ts:1012` |
| Past 35 ft | max 0.25 if perfect, else 0.04 | `game/game.ts:1013` |
| Airball | 40% of misses on a very early/late release | `game/game.ts:1015` |
| Contest, one defender within 7 ft | 1 - (1-d/7)(0.25 + DEF × 0.0025 + size × 0.06)(jumping 1.4), clamp 0.35 to 1 | `game/game.ts:965-982` |
| Contest scale for the user: Rookie, Pro, All-Star, Legend | 0.5, 0.8, 0.9, 1 | `game/types.ts:46-49` |
| Layup | (0.55 + INS × 0.004)(0.45 + 0.55 × contest) × effort factor; perfect 0.97; fire +0.2, max 0.97 | `game/game.ts:1024-1026` |
| Dunk | clamp(0.5 + INS × 0.0035 - (1-contest) × 0.6, 0.3, 0.93) × effort factor; fire or perfect 0.99 | `game/game.ts:1077-1078` |
| Shot type range | dunk ≤ 4.2 (fire +4, turbo differs), layup ≤ 7.5 | `game/game.ts:261-265, 945` |
| Rating clamp | 20 to 99 | `game/game.ts:94` |
| Body size | about 0.8 to 1.3 | `data/pokemon.ts:31, 282` |

### Arc

| Number | Value | Where |
|---|---|---|
| Release x, y | shooter x + 0.3 toward the hoop, y | `game/game.ts:1034-1035` |
| Release z | body z + height + 0.4 | `game/game.ts:1036` |
| Body height | 2.8 + size × 1.7 | `game/game.ts:98` |
| Jump velocity: jump shot, layup | 10.5, 12 ft/s | `game/game.ts:954, 948` |
| Layup release | stateT ≥ 0.24, which lands on tick 15 (0.25 s) | `game/game.ts:911` |
| Make target | rim centre, z 10.1 | `game/game.ts:1037-1039` |
| Miss target | ring at rim radius × 1.25 = 0.9375, random angle | `game/game.ts:1049-1051` |
| Airball target | 3 to 6 short, ±2 sideways, z 6.5 | `game/game.ts:1044-1047` |
| **Flight time** | jump 0.75 + d × 0.017; layup 0.55 | `game/game.ts:1054` |
| **Velocity solve** | vx,vy = Δ/T; vz = (Δz + ½gT²)/T | `game/game.ts:1063` |
| Ball height in flight | z0 + vz t - ½gt² | `game/game.ts:1595` |
| Rim rebound | ±1.3 rad, speed 5 to 12, vz 7 to 13 | `game/game.ts:1628-1632` |
| Dunk path | T = clamp(0.42 + d × 0.04, 0.45, 1), peak 3 + d × 0.3, end z rim + 0.5 - height | `game/game.ts:1083-1084, 1100-1103` |
| Dunk rebound | ±1 rad, speed 6 to 11, vz 8 to 13 | `game/game.ts:1143-1147` |
| Random number generator | same `seededRng` | `game/types.ts:294-304` |
| Block chance (listed, not in make %) | clamp(0.15 + BLK × 0.004 - SHT × 0.002 - 0.05, 0.05, 0.6) | `game/game.ts:1603` |

Worked example at the default inputs: a 23.75 ft three, size 1, meter 0.80. Release z is 6.0 ft and T is about 1.15 s. The result is **47.0° at 29.9 ft/s, peak 13.5 ft, entry 36.0°**. The game never computes these four numbers directly; they follow from its velocity solve.

### Colours

| Use | Hex | Where |
|---|---|---|
| Background, panel, panel 2, line | `#0a0d1f #121736 #1a2050 #2a3170` | `style.css:3-6` |
| Text, muted | `#f2f0e6 #9aa0c8` | `style.css:7-8` |
| Accent, red, blue, green | `#ffcb05 #e3350d #3b6fd8 #21d07a` | `style.css:9-12` |
| Grade red | `#ff6b6b` | `style.css:735` |
| Ball, ball outline | `#e8762c #5a2a0a` | `game/render.ts:604, 608` |
| Rim, stanchion, arm | `#ff5a1f #2b2f45 #3a3f5c` | `game/render.ts:682, 630, 634` |
| Stand tiers, arena | `#1c2145 #20264f #0b0e22` | `game/render.ts:96, 215` |
| Floor: wood, paint, apron, lines | `#d99a5b #3b6fd8 #16224a #faf7ee` | `data/courts.ts:27` (default home court colours) |

Fonts are Barlow Condensed and Press Start 2P, inlined from `fonts/`.

## GUESSED or not in the code

### Sim-only behaviour

- **GUESSED: hand-set angle and power.** The game has neither. Slider ranges (5° to 85°, 5 to 70 ft/s) and dial scales are my choice.
- **GUESSED: the hand-set "through the rim" test.** Through means within rim radius minus ball radius at rim height; hitting the rim means within rim radius plus ball radius. The game has no such collision. Both tests use the real radii, but the rule is mine.
- **GUESSED: hand-set backboard bounce** (velocity × 0.5). The game has no shot-to-board contact.
- **GUESSED: "power" means release speed.** The game has no such concept; this is `hypot(vh, vz)` from its solved velocity.

### Simplified inputs

- **GUESSED: a standing shooter.** Real jump shots keep 25% of run speed, so the release point can drift.
- **GUESSED: layup position.** Real layups drift toward the hoop at up to 9 ft/s for 0.25 s (`game/game.ts:1412-1413`). The simulator releases from the slider's distance with no drift. Layup make % is unaffected unless the drift changes the shot type.
- **GUESSED: one defender, on the shooter-to-hoop line.** That skips the game's × 0.6 for defenders behind the shooter (`game/game.ts:976`). Real games take the worst of five.
- **Set to fixed values:** catch-up is 0 (`game/game.ts:279`) and turbo is off.
- **Left out of make %:** the separate block roll and CPU timing noise.

### Default values

- **GUESSED: starting values.** SHT 80, INS 70, DEF 75, size 1, energy 100%, Pro difficulty. All are inside the real ranges, but no real character is used.

### Cosmetic 3D

None of these affect any number on screen.

- **GUESSED:** pole radius 0.25, pole pad, arm thickness, board thickness 0.1, rim tube 0.05, rim neck, court line width 0.17.
- **GUESSED:** a full 12-strand net (the game draws 7 strands on a half rim), rim shake size, ball spin rate.
- **GUESSED:** fallback figure radius 0.42 + 0.22 × size, defender jump height 1 ft (the game only tests z > 0.8).
- **GUESSED: the gather.** The game jumps on the press, so the 0.18 s crouch before the shot clock starts is cosmetic. It delays the shot by 0.18 s of wall time and changes no model number.
- **GUESSED: squash amounts.** Gather 24%, release pop 10% with a 0.22 rad lean, landing 14%.
- **REAL: sprite size and motion.** Sprite height `heightFt - 0.5`, largest side scaled to it, feet at the cropped bottom row, team ring radius 1.15 × size, and 8% stretch while vz > 4 (`game/render.ts:25, 498, 526-537`). Feet height is the model's `bodyZ`.
- **GUESSED: back sprites.** The game always shows the front sprite while shooting (`game/game.ts:1459`). Here the back sprite shows when the camera is behind the way the Pokémon faces, because this camera can orbit.
- **GUESSED: sprite choice.** The picker changes the look only. Ratings and body size stay on their sliders, and the default defender is Gengar.
- **GUESSED:** stand geometry and apron size, camera angles, tour timing, the airball marker at 4.5 ft (midpoint of the real 3 to 6).

### Left out on purpose

The centre-court logo, floor text, and team and arena names are not used. Character ratings exist in the repo, but the picker's Pokémon don't bring theirs.

### Sprites and credit

Sprites load at runtime from PokeAPI's GitHub repo, so opening the page contacts `raw.githubusercontent.com`. None are embedded. If the fetch fails, the fallback figure stands in. The credit line is always on screen.
