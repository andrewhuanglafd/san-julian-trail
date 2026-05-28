# THE SAN JULIAN ST TRAIL — Project Notes for Claude

A single-file, self-contained HTML game. Retro-CRT, Oregon-Trail-style EMS
training simulation aimed at EMT-level (BLS-scope) audiences. Players run
calls on a deficit budget while making it back from Lincoln Heights to
quarters in Skid Row.

© 2026 Fire Fueled Education LLC. See `COPYRIGHT` for license terms.

---

## Project Layout

```
index.html       The entire game — HTML, CSS, and ~1700 lines of JS in one file.
manifest.json    PWA manifest (installable).
sw.js            Service worker (offline + cache versioning).
icon.svg         Vector app icon (PWA, favicon).
icon-512.png     PWA icon, 512px.
firebase.json    Firebase Hosting headers.
COPYRIGHT        Ownership and licensing notice.
README.md        Player/deployer instructions.
CLAUDE.md        This file.
```

Everything lives in `index.html`. The other files exist only for PWA
behavior or deployment.

---

## Scope: BLS / EMT

This is a **BLS-scope** training tool. The game intentionally excludes
ALS-only skills (IV access, IV meds, advanced airways, cardiac drugs,
field I&D, etc.). Wrong-answer distractors that reference ALS skills are
tagged `(ALS — outside BLS scope)` so students learn the scope line.

Scope marker is visible on the boot screen (`> SCOPE OF PRACTICE
............... [BLS / EMT]`) and the title screen subtitle.

When editing PROC or CALLS, **keep additions BLS**. Check against the
LA County DHS skill sheets the user provided (NPA, OPA, naloxone, oral
glucose, epi auto-injector).

---

## Architecture

The JS lives in one `<script>` block at the bottom of `index.html`.
Top-down structure:

1. **Constants** — `CLASSES`, `STORE`, `COUNTS`, `BOOLS`, `ROUTES`,
   `EMPTY`, `PROC`, `CALLS`, `SKIDCALLS`, `EXPOSURES`, `RIPPLES`,
   `ROADS`, `SCAVS`, `CREWCHECKS`, `FILTHS`, `BARRIERS`, `CATAS`,
   `HEAT_CAP`, `MORALE_TIERS`.

2. **State** — global `S` (game state object), populated by `newState(cls)`.

3. **Utility/UI** — `render()`, `hud()`, `setScreen()`, `navBack()`,
   `navHome()`, etc.

4. **Screen functions** — `bootScreen()`, `titleScreen()`,
   `selectScreen()`, `storeScreen()`, `confirmStart()`,
   `chooseRouteScreen()`.

5. **Stop/event loop** — `nextStop()`, `advance()`, `gradeMeta()`,
   render functions for each event type (`renderCall`, `renderRoad`,
   `renderScav`, `renderEmpty`, `renderCrewCheck`, `renderFilth`,
   `renderBarrier`, `renderCata`, `renderExposure`).

6. **Call/decision flow** — `activeOptions()`, `captainDelegate()`,
   `renderCall()`, `pickCall()`, `classMod()`, `finalizeCall()`,
   `renderDet()`, `pickDet()`.

7. **Procedure runner** — `runProcedure()`, `renderProc()`, `pickProc()`,
   `renderProcDone()`.

8. **Result shell** — `showResult()` (the central hub; applies meta,
   morale, ripples, transport flow, etc.).

9. **End screens** — `winScreen()`, `failScreen()`, `sickFailScreen()`.

10. **Timers** — `startRushTimer()` / `rushTimeoutXxx()` (Tiller Killer's
    visible 15s timer) and `startQuestionTimer()` /
    `applyHiddenSpeedTimer()` (Ray's hidden 15s/25s thresholds).

`bootScreen()` is the entry point at the bottom of the script.

---

## Classes (player roles)

Five classes, each with `flags` that drive mechanics. Adding a new class
means adding flags AND wiring them into the right code path
(`finalizeCall`, `showResult`, etc.).

| Class | Multiplier | Key flags | Defining mechanic |
|---|---|---|---|
| Captain Clipboard | ×1.0 | `command`, `airwayRusty`, `morale` | Delegates to crew on most calls; rusty when forced solo. Morale starts at 2. |
| Pump Panel Pete | ×1.5 | `fixer`, `improv` | Auto-clears road hazards; reaches for fire gear on medical calls. |
| Tiller Killer | ×2.0 | `fixer`, `shortDist`, `rushed` | +3% bonus progress per stop; **visible 15s timer per question** — miss it, worst option is auto-picked. |
| Diesel Bolus Dave | ×2.5 | `fast`, `halfAss` | Time-critical bonus + no distance reset on transport. Half-ass assessments cost −3 pts/call but patient never dies. |
| Stay-and-Play Ray | ×3.0 | `byBook`, `sceneTime`, `morale` | +3 score per skill, +2 by-the-book proc steps, +1 holding per skill. Morale starts at 5; **hidden timer** — fast (<15s) +1 ★, slow (>25s) −1 ★. |

---

## Morale system

Universal stat (`S.morale`, 0–5). Currently used by Captain and Ray (both
have `flags.morale`). Different triggers, shared HUD display, shared
tier effects.

```
MORALE_TIERS:
  0 MUTINY     +1 holding per call, −2 pts/call (red, death-spiral)
  1 TIRED      −1 pt/call
  2 STEADY     neutral (Captain's starting tier)
  3 ENGAGED    +2% home/stop
  4 SHARP      +4% home/stop, +2 pts/call
  5 LOCKED IN  +6% home/stop, +4 pts/call (Ray's starting tier)
```

Tiers are applied uniformly in `showResult` after the gain/loss block.
Both classes can drop −2 ★ on bad calls ("easily lost").

---

## Routes

Three routes from Lincoln Heights → quarters. Each has `mods` that
modify event probabilities (`scav`, `road`, `crew`, `filth`, `barrier`,
`empty`, `transport`).

- **DOWN MAIN · FAST + LOADED** — high call density, most transports,
  heavy Skid Row near quarters
- **MISSION & ALAMEDA · BALANCED** — average everything
- **AROUND THE 5 · SLOW + SAFE** — fewest transports/barriers, lots of
  empty stops

Route choice fires:
- Once at start (after S&M)
- Once mid-route at 50% progress (auto-trigger)
- Once after every required transport for non-Dave classes
  (via `S._needsHospitalRoute` flag)

---

## The transport flow

When `r.transport && r.saved===true` fires:
- **Diesel Bolus Dave** (`flags.fast`): no penalty.
- **Everyone else**: progress −30%, sets `S._needsHospitalRoute = true`.
  Next `nextStop()` triggers `chooseRouteScreen('hospital')` before
  incrementing the stop counter.

---

## The ripple system

Every medical call has a chance of spawning a downstream consequence
(`RIPPLES.good` / `RIPPLES.bad`). Good outcomes → 55% chance of positive
ripple; bad outcomes → 55% chance of negative; mid outcomes → 30%
either way. Ripples modify progress, holding, score, supplies, and/or
trigger a scavenger find.

---

## The exposure system (Oregon-Trail-style "diseases")

`EXPOSURES` fires as its own event type during nextStop, scaling with
Skid Row proximity. Each exposure adds to `S.exposed` (cumulative,
displayed in HUD as `EXPOSED N`) and `S.expoLog` (titles listed on the
win screen under "WHAT YOU TOOK HOME").

The urgent BBP clock (`S.sick`, separate from `S.exposed`) is started
only by certain catastrophes (`CATAS` entries with `kind:'needle'`) and
ticks down each `advance()`; reaching 0 triggers `sickFailScreen()`.

---

## Scoring (Oregon-Trail-style breakdown)

`winScreen()` shows an itemized shift report. Score formula:

```
subtotal = decisions + patients + speed + composure + cash + supplies
           + scav − expoPenalty
final    = subtotal × class.mult
```

Where:
- decisions = `S.score` (running total from call/proc picks)
- patients = `S.saved*25 − S.lost*30`
- speed = `(16 − S.stopCount) * 4`
- composure = `(HEAT_CAP − S.peakHeat) * 3`
- cash = `S.budget` left over
- supplies = `4*units + 12*equip` remaining in `S.inv`
- scav = `S.scavenged * 5`
- expoPenalty = `S.exposed * 4`

---

## Common edits

**Adding a CALL:** copy an existing entry's shape. Required fields:
`id`, `crewCan`, `direct`, `title`, `who`, `scene` or `sceneVars`,
`q` or `qVars`, `options[]` (each option has `l`, `key`, and an outcome
shape: `ok:'good'/'mid'/'bad'`, `sc`, `saved`, `text`, `teach`, plus
optional `skill`, `needs`, `consume`, `transport`, `det`).

**Adding a SKIDCALL:** same as CALL but lives in `SKIDCALLS`. Calls
biased into the pool more heavily as Skid Row proximity rises.

**Adding a PROC step:** `PROC[skill].steps[].o[]` — each step has
`q`/`qV`, `teach`, and `o[]` options where `ok` is `0` / `0.5` / `1`
(or a function returning a number).

**Adding an EXPOSURE:** add to `EXPOSURES[]`. Required: `title`, `who`,
`scene`, `sickAdd`, `heatAdd`, `sc`, `teach`.

**Adding a RIPPLE:** add to `RIPPLES.good[]` or `RIPPLES.bad[]`. Each
ripple object has `title`, `effect`, plus any of `prog`, `heat`, `sc`,
`morale`, `scav`, `inv`/`invDelta`.

---

## Cache busting

After **any** edit that changes user-visible behavior, **bump the service
worker cache version** in `sw.js`:

```js
const CACHE = 'sjt-vN';   // increment N
```

Without bumping, deployed users will keep hitting the cached old
version because `sw.js` is cache-first.

Currently at `sjt-v15`.

---

## Testing locally

It's a static HTML file. Just `open index.html` in a browser. No build
step. No dependencies beyond Google Fonts (which the page loads from the
CDN; falls back gracefully offline).

After service worker has registered once, hard-refresh (⌘+Shift+R) to
bypass the cache when iterating.

---

## Coding conventions

- **No build step.** Plain JS + CSS + HTML in one file. Keep it that way.
- **Single-quote strings** in JS except when interpolating; **double-quote** when
  the string contains apostrophes (avoid escape hell).
- **Curly Unicode quotes** (`"`/`"`/`'`/`'`) inside strings to avoid
  conflicting with the delimiters.
- **Em-dashes** (`—`) in all UI copy — never `--` or ` - `.
- **`var(--body)`, `var(--pixel)`, `var(--crt)`** for fonts. Don't
  hardcode font-family.
- **CSS custom properties** for all colors (`--green`, `--amber`,
  `--red`, `--ink`, etc.). Don't hex-code colors inline.
- **VT323** for body and gameplay text (DOS / Turbo Pascal / Oregon
  Trail vibe). **Press Start 2P** only for the giant game title.
- **All sizes pixel-tuned for 24–60yo readability.** When adding new
  UI, default to 17px+ body, 13px+ for labels.
