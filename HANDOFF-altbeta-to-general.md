# OH DEER — alt beta → general beta

**Changelog of non-biome-specific changes, and a porting handoff.**

Prepared for Grok. Source of truth for every value below is the alt beta build
(`index.html` on branch `claude/oh-deer-beta12-polish-jqniry`, titled BETA13.1).

---

## 0. What these two files are

| | **Alt beta** (source) | **General beta** (target) |
|---|---|---|
| File | `index.html` / `ohdeerbeta11.html` lineage | `ohdeeraltbeta8libertybell.html` |
| Title tag | `OH DEER. — BETA13.1` | `OH DEER. — ALTBETA7` |
| Lines | ~8,900 | 7,571 |
| Biomes | **locked to forest** (`biomeReset` pins index 0, `biomeStep` is a no-op) | **live rotation** — random start, `BIOME_HOLD` timers, crossfades |
| Version markers present | ALTBETA3/4/5/7/8 + BETA10/11/12/13 + PEDAL TEST | ALTBETA3/4/5/7/8 only |

The alt beta is the experiment bench — biomes were pinned so the driving model
and the arsenal could be tuned without the road changing underneath. The general
beta is the real game. Everything below was worked out on the bench and now
needs to go into the game.

**Read section 1 before writing any code.** The general beta is missing an
entire subsystem that four of these changes are built on, and it still has a
live feature that one of them replaces. Porting in file order will not work.

---

## 1. Compatibility briefing — four things about the target

### 1.1 The general beta has no throttle system at all

This is the big one. The general beta computes road speed in a single line:

```js
const speed = Math.min(545, 235 + elapsed * 6.5) * (superOn ? 1.5 : 1);
```

There is no `throttle`, no `pedalBrake`, no `THROTTLE_*` constant, no brake
pedal, no speed gauge. Greps for `THROTTLE`, `pedalBrake`, `pedalGas`,
`baseSpeed` all return **zero hits**.

The alt beta's driving model came from BETA10/BETA11, which are not in the
target: the gas pedal was deleted, the van accelerates on its own toward a
natural top end, and the brake ramps linearly to a dead stop. Both throttle
changes in this changelog (§2.6, §2.7) are edits *to that model*. They cannot
be applied to the target as-is — there is nothing to edit.

**Decision required from Ray before §2.6/§2.7 are attempted.** Either:

- **(a)** port the whole BETA11 pedal model first — brake pedal, `throttle`,
  `THROTTLE_MAX/ACCEL`, `BRAKE_RATE`, the speed gauge, the `min(355, 178 +
  elapsed*3.2)` base curve — and *then* apply §2.6 and §2.7 on top; or
- **(b)** skip §2.6 and §2.7 in this pass and port the other nine changes,
  which are all independent of it.

Everything else in this document works against the target as it stands. Do not
attempt a partial throttle port — a settle value with no throttle to settle is
meaningless, and the `min(545, 235 + elapsed*6.5)` curve is roughly 1.5× the
alt beta's pace, so the tuning would not transfer even if it compiled.

### 1.2 The comet is still live in the general beta, and already Overdrive-gated

The alt beta retired Manifest Combustion in BETA12 and then re-armed the same
machinery as FREEDOM MODE in BETA13. The general beta never retired it:

```js
function startFlame(){
  if (mode !== 'play' || paused || !isSuper() || flameStock <= 0 || flame.phase) return;
  flameStock--;
  ...
```

So in the target, the 🔥 button already exists, already requires Overdrive, and
runs off a `flameStock` fuel counter fed by cans, gold deer and pickups.

This makes §2.10 (FREEDOM MODE) a **re-specification of a live feature, not a
new one**. What changes is the unlock condition, the blaze leg, and the
presentation — not the launch/blaze/return state machine, which is identical in
both files. Do not delete `flameStock`; decide with Ray whether fuel pickups
survive alongside the new arming rule or are removed.

### 1.3 The general beta predates the BETA10 entity-health layer

These do not exist in the target (`grep -c` = 0):

`echoStrike` · `hitMoose` · `hitGator` · `mortarEchoes` · `MORTAR_MAX` ·
`RIFLE_MAX` · `MORTAR_STREAK_STEP` · `carReturnFire`

Consequences:

- The moose is a **one-hit kill** in the target (`mooses.splice(i, 1)`), not a
  3-HP entity. The torch's moose clause (§2.9) must be rewritten as a plain
  removal, or dropped.
- The gator has no `hitGator` entry point — check how it takes damage in the
  target before wiring the torch or `blastRigs` to it.
- There are no live mortar echoes, so the `blastRigs` call inside `echoStrike`
  (§2.8) has no home. One less call site; the mortar's primary burst still
  needs it.
- Ammo caps are literals in the target (`mortarAmmo < 3`, `plowAmmo < 2`), not
  named constants. Any ported line referencing `MORTAR_MAX` or `RIFLE_MAX` must
  be rewritten against the literal or the constant introduced first.

### 1.4 Tuning constants differ — do not copy award thresholds across

| | Alt beta | General beta |
|---|---|---|
| Rifle re-arm | `nextRifleAt += 10`, **plus a score rail** | `nextRifleAt += 25`, streak only |
| Plow | `plowAmmo < 4`, `+= 14`, run starts with 4 | `plowAmmo < 2`, `+= 32`, run starts with 0 |
| Mortar | `mortarAmmo < MORTAR_MAX`, `MORTAR_STREAK_STEP` | `mortarAmmo < 3`, `+= 40`, score step 220 |
| Mortar button | right column, 84px, 58×58 | right column, **292px, 52×52** |

The gating changes in §2.8 wrap these conditions in an Overdrive test. **Wrap
the target's own numbers — do not import the alt beta's.** Whether the general
beta also wants the alt beta's looser rifle/plow economy is a separate call for
Ray, not part of this port.

---

## 2. Changelog — non-biome-specific changes

Ordered by dependency, not by when they were made. Each entry gives the intent,
the exact values, where it lives, and what to watch for in the target.

---

### 2.1 Traffic mix rebalance — `CAR_POOL`

**Problem.** Sedans were 14% of a pool where the working vehicles were 5–9%
each. Roughly every third car was the same grey sedan; pickups, box trucks and
towtrucks were rare enough not to register as part of the road.

**Change.** Sedan cut hard, working vehicles raised. Weights still sum to
exactly 1.0000 (verified programmatically).

| type | key | was | now |
|---|---|---|---|
| 0 | hatch | 0.12 | **0.10** |
| 1 | sedan | 0.14 | **0.08** |
| 2 | pickup | 0.11 | **0.14** |
| 3 | truck | 0.06 | **0.09** |
| 9 | suv | 0.14 | **0.12** |
| 10 | minivan | 0.12 | **0.10** |
| 11 | sport | 0.11 | **0.10** |
| 16 | muscle | 0.09 | **0.12** |
| 17 | towtruck | 0.06 | **0.08** |
| 18 | camper | 0.05 | **0.07** |

**Target status.** `CAR_POOL` in the general beta is **byte-identical** to the
alt beta's pre-change version. Clean drop-in replacement, no adaptation.

---

### 2.2 The clown car becomes a kei car

**Problem.** The clown car was a hot-pink blob with polka dots and a red nose.
At gameplay scale it did not read as a vehicle at all.

**Change.**

- `CAR_TYPES[4].key`: `'clown'` → `'kei'`. Hull dimensions unchanged
  (`hw:15, hh:24, wMin:70, wMax:120`).
- `HULLS.clown` → `HULLS.kei`, **numbers unchanged** — the short, tall, rounded
  hull was always right; only the paint was wrong.
- `paintCar()` branch rewritten: ordinary body colour from `CAR_COLORS` (the
  forced `'#ff3d6a'` is gone), tall upright cabin, near-vertical windshield,
  short rear glass, one thin door cut, small round headlamps stamped over the
  standard nose bar. No dots, no nose, no calliope.
- Removed the type-4 weaving/honking/calliope behaviour from the car update
  loop and the `sndCalliope()` call in the announce block. `sndCalliope()` itself
  is kept but is now uncalled.

**Target status.** Only 4 occurrences of `clown` in the general beta, same
shape as ours pre-change. Clean rename.

**Watch for:** the round-lamp overlay is centred with
`cx = xN - 1.6 - xN * 0.31` — this is derived from `carLights()`'s own rect
geometry. If the target's `carLights()` differs, recompute or the stock
rectangles peek out from behind the circles. (This was a real bug in the first
alt beta pass.)

---

### 2.3 Paint-code gate rewritten as a set

**Problem.** `carSprite()` forced `colIdx = 0` for every type ≥ 4 except an
explicit allow-list. With the kei at index 4, every kei would have baked silver.

**Change.** Replace:

```js
if (ti >= 4 && ti !== 9 && ti !== 10 && ti !== 11 && ti !== 16 && ti !== 17 && ti !== 18 && ti !== 14) colIdx = 0;
```

with:

```js
const PAINTED_TYPES = new Set([0, 1, 2, 3, 4, 9, 10, 11, 14, 16, 17, 18]);
...
if (!PAINTED_TYPES.has(ti)) colIdx = 0;
```

Identical behaviour for every existing type, plus 4.

**Target status.** The old chain is present verbatim at target line ~2197.
Direct swap. **This is a hard dependency of §2.2** — port them together or the
kei ships silver.

---

### 2.4 Kei removed from the late-game oddball pool

**Change.** In `pickCarType()`, the `score > SPECIAL_AT` branch: pool goes from
`[4, 5, 6]` to `[5, 6]`. The `push(7)` at 160 and `push(8)` at 220 are unchanged.

**Consequence worth stating plainly:** with the kei out of the special pool and
never added to `CAR_POOL`, **type 4 no longer spawns in normal play at all** —
it is reachable only via `__OD.car(4)`. That was deliberate on the bench ("for
now"), so the sprite could be fixed before deciding its slot. If the general
beta should actually put keis on the road, it needs a `CAR_POOL` weight, which
means re-balancing §2.1 to keep the sum at 1.00.

**Target status.** `pickCarType()` is byte-identical to ours pre-change.

---

### 2.5 The plow blade gets comically wide

**Problem.** The blade was ±30px on a 22px-half-width van — barely wider than
the bumper it was bolted to, and it did not read as municipal equipment.

**Change.**

```js
const PLOW_HW = 78;                       // most of a lane
const plowSpan = () => (plowT > 0 ? PLOW_HW : 0);
function vanSweep(x, halfW){
  return Math.abs(x - van.x) < Math.max(van.hw, plowSpan()) + halfW;
}
```

Every collision test that should respect the blade now calls `vanSweep()`
instead of comparing against `van.hw`. Sites converted in the alt beta —
**all six anchors confirmed present in the general beta**:

| what | target anchor |
|---|---|
| traffic | `Math.abs(c.x - van.x) < van.hw + c.hw - 4` |
| deer | `Math.abs(d.x - van.x) < van.hw + 15` |
| moose | `Math.abs(m.x - van.x) < van.hw + 26` |
| barrels | `van.hw + rw` |
| rigs | `const hitX = Math.abs(rg.x - van.x)` (see §2.8) |
| draw | `if (plowT > 0 && mode === 'play')` |

**`vanSweep()` is a no-op when the blade is up** — `plowSpan()` returns 0, so
the expression collapses to exactly the original test. That is what makes this
safe to apply at every site at once.

**One extra clause, needed.** A car can now be caught well outside the bumper
line, where no van collision is geometrically possible. Inside the traffic
branch, before the normal handling:

```js
if (plowT > 0 && Math.abs(c.x - van.x) >= van.hw + c.hw - 4){
  // outboard of the bumper — purely the blade's business, launch and continue
}
```

Without it, a car clipped by the blade tip falls through to the van-collision
path and ends the run.

**Art.** The mouldboard is redrawn at ±78 with a lengthwise gradient, mounting
arms back to the bumper, a cutting edge, ribs spaced off the width, and hazard
chevrons on the outboard thirds. Full replacement block, ~30 lines.

---

### 2.6 Throttle floor after a full stop — ⚠️ **needs §1.1 resolved**

**Problem.** Braking to a dead stop cost nothing: the frame you released, the
throttle snapped straight back to `THROTTLE_MIN = 0.55`.

**Change.** `THROTTLE_MIN = 0.55` → `THROTTLE_CRAWL = 0.15`. Off the brake from
a standstill the van creeps and builds from there.

**Measured on the bench:** 8 MPH off the line, climbing to cruise over ~7s.
Previously it resumed instantly at ~29 MPH.

**Target status.** Blocked — no throttle system. See §1.1.

---

### 2.7 Overdrive hands the wheel back at a cruise — ⚠️ **needs §1.1 resolved**

**Problem.** Overdrive spends nine seconds pushing the throttle to the ceiling
and then handed it back still pinned there. The ordinary road *after* your first
Overdrive therefore ran permanently faster than the ordinary road before it —
for the rest of the run. This was Ray's "the car moves too fast of its own
accord after PO" complaint, and this is the actual mechanism.

**Change.** In `endSuper()`:

```js
const THROTTLE_SETTLE = 0.60;
...
if (!pedalBrake && throttle > THROTTLE_SETTLE){
  throttle = THROTTLE_SETTLE;
  popup(van.x, van.y - 58, 'EASING OFF.', 14, false);
}
```

**Measured:** 1.22 during Overdrive → 0.60 at the drop → back to cruise over
~4s.

**Target status.** Blocked — see §1.1. **But flag the underlying complaint to
Ray regardless:** the general beta's `min(545, 235 + elapsed*6.5)` curve is
faster than the alt beta's `min(355, 178 + elapsed*3.2)` at every point in the
run, and its Overdrive multiplier is the same 1.5×. If the pace felt wrong on
the bench, it is worse in the target, and no amount of throttle work fixes a
base curve that reaches 545.

---

### 2.8 The kit / arsenal split

This is the largest behavioural change and the one most likely to need
discussion before it lands. It went through two passes on the bench; only the
final state is described here.

**The rule.** The fleet is split in two and the halves never share the screen.

| | **Standing kit** | **Arsenal** |
|---|---|---|
| Members | rifle, plow | mortar, bottle (spray), air strike, horn, bell |
| Earned | ordinary road only | under Overdrive only |
| Fired | ordinary road only | under Overdrive only |
| Visible | ordinary road only | under Overdrive only |

The **torch** (§2.9) belongs to neither column — 500 lifetime deer buys it
outright and it burns whenever. FREEDOM MODE (§2.10) is Overdrive-only by
definition.

**Implementation, four parts:**

1. **Two guards**, mirror images:
   ```js
   function arsenalLive(what){        // arsenal needs the colors up
     if (isSuper()) return true;
     popup(van.x, van.y - 62, 'NOT UNTIL THE COLORS FLY', 14, false);
     return false;
   }
   function kitLive(what){            // kit needs the colors down
     if (!isSuper()) return true;
     popup(van.x, van.y - 62, 'STOWED UNDER THE COLORS', 14, false);
     return false;
   }
   ```
   Called at the top of `fireMortar`, `fireSpray`, `fireBombRun`, `soundHorn`,
   `fireBell` (arsenal) and `fireRifle`, `firePlow` (kit).

2. **Award gating.** The mortar/spray/bomb award conditions get an `isSuper() &&`
   prefix. ⚠️ **Wrap the target's own thresholds — see §1.4.** Rifle and plow
   awards are left alone.

3. **Visibility.** Each `update*Btn()` show-condition gains `&& isSuper()`
   (arsenal) or `&& !isSuper()` (kit). Horn and bell had no hidden state at all
   and need `#hornBtn.hidden,#bellBtn.hidden{display:none}` added to the CSS.

4. **One per-frame authority.** This is the part that is easy to miss and was a
   real bug on the bench:

   > The `update*Btn()` functions were only ever called when a weapon was
   > **awarded or fired**. Nothing re-read them when an Overdrive *started or
   > ended*, so the buttons showed the wrong set until you next earned
   > something.

   Fix: one `updateArsenalBtns()` that calls all of them plus the horn/bell
   toggles, invoked every frame from `update()`.

**Ammo survives the transition on both sides** — a shell earned in the last
second of Overdrive is not wasted, it waits for the next one. Only earning and
firing are gated, never the holding.

**Button slots — the two sets share them.** Because kit and arsenal are mutually
exclusive, they occupy the same right-column positions, so nothing floats out of
thumb reach when half the fleet is stowed:

| slot | ordinary road | under the colors |
|---|---|---|
| right 14px | rifle | horn |
| right 84px | plow | mortar |
| right 154px | **torch** | **torch** |
| left 84/154/224 | — | bell / bottle / strike |
| centre, 92px | — | FREEDOM banner |

Verified in both states: every visible button has a unique bounding box, no
overlaps, on a 420×760 viewport.

⚠️ **Target adaptation.** The general beta's slots differ — its 🔥 comet button
holds right/84 and its mortar sits at right/292 at 52×52 rather than 58×58. The
shared-slot table above has to be re-derived against the target's own layout
and against whatever §2.10 decides about the 🔥 button.

---

### 2.9 The oncoming rig stops being invulnerable

**Problem.** The wrong-way rig could only ever be dodged. Every weapon in the
game passed straight through it, which made it the one thing on the road with no
answer.

**Change.** One shared exit so no route can leave a half-dead rig:

```js
function killRig(i, line){ /* splice, boom, fireRing ×2, debris, sndSlam,
                              sndRigHorn, shake, buzz, triggerFreedom(2), popup */ }
function blastRigs(x, y, R){ /* radius test → killRig */ }
const RIG_KILL_LINES = ['JACKKNIFED.', 'WRONG WAY, WRONG DAY.', 'REROUTED.', 'THAT ONE YIELDED.'];
```

Wired into: **rifle rounds** (point test, added at the top of the bullet loop so
a round spends on the rig), **mortar burst** (`blastRigs(x, y, MORTAR_R)`),
**air strike** (`blastRigs(bm.x, bm.y, R)` at the bomb detonation, `R = 84`),
**plow blade** (new branch in the rig loop, using `vanSweep`), **torch** (cone
test), **Overdrive shoulder** (existing branch rerouted through `killRig`), plus
the fireball and the pre-existing avalanche.

**Verified on the bench:** a rig spawned in the van's lane, six weapons, six
kills — rifle, mortar, air strike, blade, torch, Overdrive.

**Target status.** `killRig`'s dependencies (`debris`, `fireRing`, `boom`,
`triggerFreedom`, `popup`) all exist. Call sites: the rig loop, `mortarBurst`
and the bomb detonation (`const R = 84;`) are all present. **`echoStrike` does
not exist in the target** — that call site is simply absent, one fewer edit.

One extra site in the target worth folding in: its avalanche already clears
rigs, but with its own inline removal —

```js
if (inBand(rigs[i].y)){ boom(rigs[i].x, rigs[i].y, 60); rigs.splice(i, 1); }
```

Route that through `killRig(i)` as well. The whole point of the shared exit is
that a rig cannot leave the board by a path that skips the effects, the sound
and the `triggerFreedom` credit — an avalanche that quietly deletes one is
exactly the inconsistency this is meant to remove.

---

### 2.10 FREEDOM MODE

**On the bench** this was a revival: BETA12 had retired the comet but
deliberately left `flame.phase` and its ~50 guards standing, so re-arming it was
a re-specification rather than a rebuild.

**In the target it is a live feature already** (§1.2). What follows describes
the end state; the diff against the general beta is smaller than the diff
against the alt beta was.

**Unlock.** `ledger.lifetime >= 1500`. Career equipment, not a run reward — 1500
in a single drive is not happening.

**Arming.** Ten *consecutive* kills inside a single Overdrive. "Consecutive" is
defined by the existing combo clock: `poKills` increments in `killDeer()` on the
`isSuper()` branch and is zeroed wherever `combo` is zeroed, so letting the road
go quiet for ~2s costs the run-up. Also zeroed by `activateSuper()` and
`endSuper()` — armed and unspent, it dies with the flag.

**Presentation.** A wide red/white/blue banner across the centre of the screen
at bottom+92px, strobing on a 3-step `steps(3,end)` animation. Not a round
button in a column — it appears perhaps once in a drive and should not have to
be hunted for.

**What spending it does:**

| leg | duration | behaviour |
|---|---|---|
| `launch` | until wall clears top | existing fire wall, unchanged |
| `blaze` | `FREEDOM_BLAZE = 2.6s` | **rewritten** — see below |
| `return` | until van catches up | existing, unchanged |

The blaze leg was 0.55s of screen-shake with the wall parked offscreen. It is
now the point of the mode:

- road speed × `FREEDOM_SPEED = 3.0` (blaze leg only)
- everything entering the road dies continuously — deer, traffic, rigs, wolves
- fire patches laid the whole way, not just where the wall passed
- the van is **drawn** as the fireball (`drawFreedomVan()`): three offset
  red/white/blue tail lobes streaming ~470px back, an additive comet head, and
  flame licks off the leading edge, with the van still legible inside it
- full-screen RWB wash on a 3-step strobe at 0.17 alpha — deliberately held to
  a third of full so the road stays drivable and it does not become a
  photosensitivity problem

**Constants:** `FREEDOM_LIFETIME = 1500`, `FREEDOM_KILLS = 10`,
`FREEDOM_BLAZE = 2.6`, `FREEDOM_SPEED = 3.0`.

**Verified:** `launch → blaze → return → null`, ~3.5s end to end, no errors.

**Open question for Ray, and the reason this entry needs a conversation before
it lands:** the target's 🔥 button currently runs off `flameStock` fuel from
cans, gold deer and pickups. Does the new arming rule *replace* that fuel
economy, or sit alongside it? The bench build has no fuel path at all.

---

### 2.11 The flamethrower ("the torch")

**Unlock.** `ledger.lifetime >= 500`. Permanent once earned.

**Deliberately not Overdrive-gated** — it is the answer to the ordinary road,
standing alongside the rifle and the blade.

**Held, not fired.** Fuel drains while the trigger is down and refills while it
is up, so the ceiling is nerve rather than ammunition:

```js
const TORCH_TANK = 3.2, TORCH_REFILL = 0.42, TORCH_REACH = 250, TORCH_SPREAD = 0.40;
```

**Cone geometry.** Measured from the van's nose (`van.y - van.hh`), half-width
at distance `dy` is `16 + dy * 0.40`, out to 250px. The draw and the collision
test read the same numbers — **if you adjust one, adjust both or the fire will
lie about its reach.**

**What it kills:** deer (instant), traffic (one car per 0.1s, so a long hold
walks up a queue instead of vaporising the screen), rigs, crates, wolves, moose,
sasquatch, gator. Lays fire patches behind the cone.

**Controls.** Round button, right column 154px, with a fuel bar. `pointerdown` /
`pointerup` / `pointercancel` / `pointerleave`, plus `F` held on desktop.

**Art.** Three stacked additive wedges (orange → amber → near-white core) with a
ragged breathing tip and a muzzle glow.

**Target adaptation.** ⚠️ The moose and gator clauses call `hitMoose` /
`hitGator`, which **do not exist in the general beta** (§1.3). Rewrite the moose
clause as a plain removal and check how the target damages the gator, or drop
both clauses for this pass.

---

## 3. Explicitly excluded — biome-specific work

Per Ray's scope, these are **not** in this handoff. They exist on the bench and
can be ported later once the general beta's rotation is accounted for:

- **`logger` (type 19) and `ranger` (type 20)** — two new vehicles appended to
  `CAR_TYPES` so no existing index shifts, with matching `HULLS` entries and
  full paint branches. Spawned by a single roll in `pickCarType()` **while the
  biome is forest** — ranger 1-in-~22 ordinary spawns, logger 1-in-~13 — the
  same arrangement farmland has with the tractor and combine. Neither is in
  `CAR_POOL`.
- **The `FOREST_TYPES` constant** and the first-sighting toasts for both.
- **The forest biome lock itself** (`biomeReset` pinning index 0, `biomeStep`
  short-circuited) — bench-only scaffolding, must never reach the general beta.

Note for whoever ports these later: they are *structurally* clean — appended
indices, one spawn hook, no shared state — but the logger was specced to run
over deer on contact as an ally, and that behaviour was never built. It is still
outstanding.

---

## 4. Suggested port order

Dependency-ordered. Each group is independently shippable and testable.

**Group A — traffic (no dependencies, lowest risk)**
1. §2.3 paint-code set — *must precede §2.2*
2. §2.2 clown → kei
3. §2.1 `CAR_POOL` rebalance
4. §2.4 kei out of the special pool

**Group B — the blade (self-contained)**

5. §2.5 `PLOW_HW` / `vanSweep` + the outboard-launch clause + the new art

**Group C — the rig (depends on B for the blade branch only)**

6. §2.9 `killRig` / `blastRigs` and all call sites

**Group D — needs a decision first**

7. §2.10 FREEDOM MODE — resolve §1.2 (fuel economy) with Ray first
8. §2.11 the torch — resolve §1.3 (moose/gator) first
9. §2.8 kit/arsenal split — **do this last**; it touches the visibility of
   everything above, and the shared-slot layout depends on what §2.10 does with
   the 🔥 button

**Group E — blocked**

10. §2.6 and §2.7 throttle — blocked on §1.1. Needs Ray's call on porting the
    BETA11 pedal model.

---

## 5. Verifying the port

The bench build exposes everything through `window.__OD` (the whole script is
inside an IIFE — top-level names are **not** reachable from the console or from
an automation harness, only `__OD` is). Hooks worth adding to the general beta
while porting:

| hook | purpose |
|---|---|
| `__OD.grant(n)` | set `ledger.lifetime` outright — the 500/1500 unlocks are otherwise a very slow thing to test honestly |
| `__OD.torch(bool)` | hold / release the torch |
| `__OD.freedom()` | run the real arming path, so the 1500 gate stays testable |
| `__OD.blaze()` | arm and immediately spend |
| `__OD.rigHere()` | spawn a rig **in the van's lane** — essential, since a randomly-placed rig will simply drive past a rifle test and produce a false negative |
| `__OD.sprite(t, c)` | return a baked body sprite for looking at paint up close |
| `__OD.state()` | extended with `lifetime, superOn, poKills, freedomArmed, flame, torchFuel, torchOn, plowT, throttle, ammo{}` |

Checks that actually caught bugs on the bench, worth repeating:

1. **Pool weights sum to exactly 1.0000** — parse `CAR_POOL` and add them up.
2. **Kei paints in more than one colour** — bake `carSprite(4, c)` for several
   `c` and confirm they differ. Catches a missed §2.3.
3. **Rig vs. every weapon** — `rigHere()`, then each weapon in turn, asserting
   `rigs` decrements. Six for six.
4. **Button visibility in both states** — read `classList` and
   `getBoundingClientRect()` for every button on the ordinary road and again
   under Overdrive. Assert the correct set is visible **and that no two visible
   buttons overlap**.
5. **Fire the arsenal outside Overdrive and the kit inside it** — assert ammo
   counts are *unchanged*. A refusal that silently consumes ammo is the obvious
   failure mode here.
6. **Overdrive transition** — trigger Overdrive and confirm the button sets swap
   *without* awarding or firing anything. This is the §2.8 part-4 bug.
7. **Flame phase progression** — `launch → blaze → return → null`. Sample every
   500ms; sampling once mid-blaze produces a false "stuck" reading.

A headless Chromium harness driving `__OD` and screenshotting the canvas was
enough for all of the above. The canvas screenshot does **not** capture the HUD
buttons — those are DOM overlays, so assert on `classList` rather than pixels.

---

## 6. Summary for Ray

Eleven non-biome changes. Nine port cleanly against the general beta as it
stands. Two are blocked on a decision, and two more need a conversation first:

- **Blocked:** §2.6 and §2.7 (throttle) — the general beta has no throttle
  system to edit. Either port the BETA11 pedal model first or defer both.
- **Needs a decision:** §2.10 (does the new FREEDOM arming replace the existing
  `flameStock` fuel economy?) and §2.11 (the moose is one-hit in the general
  beta, so the torch's moose clause needs rewriting or dropping).
- **Also worth raising:** the general beta's base speed curve tops out at 545
  against the bench's 355. If the pace felt wrong on the bench, it is worse
  here, and that is a base-curve problem rather than a throttle problem.

Biome-specific work — the logger, the ranger, and the forest lock — is excluded
by scope and listed in §3 for a later pass.
