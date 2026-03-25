# Design Document: VPX Dev Plugin

## Overview

A Claude Code plugin consisting of skill files that encode VPX table development knowledge. Unlike a software application, this plugin has no code to execute — it's a structured knowledge base that Claude loads contextually to generate correct VBScript, configure physics, and troubleshoot issues.

The plugin is a single directory (`~/.claude/skills/vpx-dev/`) containing topic-specific SKILL.md files. Claude Code's description-matching system activates relevant skills when the user's query contains matching keywords.

_Requirements: 1.1-1.5_

## Architecture

### Decision: Single Directory vs Subdirectories

**Options:**
1. **Single SKILL.md** — one massive file (~30k tokens) covering everything
2. **Subdirectories** — `vpx-dev/game-logic/SKILL.md`, `vpx-dev/physics/SKILL.md`, etc.
3. **Flat files** — `vpx-dev/SKILL.md` (master) + `vpx-dev/game-logic.md`, `vpx-dev/physics.md` (referenced)

**Decision:** Option 2 — Subdirectories with independent SKILL.md files

**Rationale:** Each skill activates independently via its own `description` field. A single file wastes context when only physics knowledge is needed. Subdirectories match the Kiro plugin pattern (proven structure). Each file stays under ~5k tokens for efficient context use.

**Implication:** The master skill (`vpx-dev/SKILL.md`) must have the broadest keyword coverage to activate as the entry point. Topic skills activate alongside it when their keywords match.

### Plugin Structure

```
~/.claude/skills/vpx-dev/
├── SKILL.md                    — Master: workflow overview, skill index, routing
├── object-reference/SKILL.md   — VPX object types: events, properties, methods
├── glf/SKILL.md                — Game Logic Framework for original tables
├── core-vbs/SKILL.md           — core.vbs API: cvpmTrough, cvpmSaucer, etc.
├── flexdmd/SKILL.md            — FlexDMD display system: scenes, labels, animations
├── game-logic/SKILL.md         — Modes, multiball, scoring, ball locks, tilt, attract
├── nfozzy-physics/SKILL.md     — Rubber separation, flipper tuning, targets, materials
│   └── nfozzy-reference.md     — Complete nFozzy physics data (curves, constants)
├── pinmame-wiring/SKILL.md     — ROM switches, coils, lamps, SolCallback, FastFlips
├── lighting/SKILL.md           — GI, flashers, inserts, Light State Controller
├── conventions/SKILL.md        — Naming, layers, timers, performance, standalone compat
├── sound/SKILL.md              — Fleep, ball rolling, ramp transitions, SSF
├── troubleshooting/SKILL.md    — 28+ error patterns with diagnosis + fix
└── toys-mechs/SKILL.md         — Diverters, magnets, VUKs, spinners, motors
```

### Keyword Strategy

Each skill's `description` field must contain keywords broad enough to match natural-language queries. The pattern: `"VPX VBScript pinball [topic] [synonyms] [common questions]"`.

| Skill | Description Keywords |
|-------|---------------------|
| **Master** | `VPX VBScript pinball table development scripting game rules physics lighting sound ROM PinMAME` |
| **object-reference** | `VPX object event property method Hit Unhit Spin Timer Slingshot Collide Dropped Raised trigger kicker gate spinner flipper bumper` |
| **glf** | `VPX GLF Game Logic Framework original table mode shot multiball scoring event trough ball device counter timer` |
| **core-vbs** | `VPX core.vbs cvpmTrough cvpmSaucer cvpmDropTarget cvpmMagnet cvpmMech SolCallback framework class` |
| **flexdmd** | `VPX FlexDMD DMD LCD display scene label font image video animation action score scoreboard segment UltraDMD` |
| **game-logic** | `VPX VBScript pinball game rules modes multiball scoring ball lock ball save tilt attract wizard mode state machine FlexDMD drain` |
| **nfozzy-physics** | `VPX VBScript pinball physics nFozzy flipper rubber bumper slingshot target elasticity friction material tuning cradle polarity velocity curve trigger separation` |
| **pinmame-wiring** | `VPX VBScript pinball ROM PinMAME switch coil lamp solenoid SolCallback trough drain FastFlips wiring GI` |
| **lighting** | `VPX VBScript pinball light GI flasher insert lamp fade lighting BlendDisableLighting Lampz Light State Controller` |
| **conventions** | `VPX VBScript pinball conventions naming layers timer performance optimization standalone compatibility` |
| **sound** | `VPX VBScript pinball sound Fleep rolling SSF audio music MP3 ball rolling ramp` |
| **troubleshooting** | `VPX VBScript pinball debug fix error crash ball through flipper wrong physics broken frame rate fps performance standalone z-fighting NVRAM FlexDMD sounds` |
| **toys-mechs** | `VPX VBScript pinball toy mechanism diverter magnet VUK kicker spinner motor drop target captive ball` |

_Requirements: 1.1, 1.2, 1.3_

### Scope Boundary with vpx-editor Skill

| Topic | VPX Dev Plugin | vpx-editor Skill |
|-------|---------------|-----------------|
| VBScript patterns | YES | NO |
| Physics tuning | YES | NO |
| ROM wiring | YES | NO |
| Whitewood mode development | NO | YES |
| Three.js scene code | NO | YES |
| vpx-editor codebase | NO | YES |
| Coordinate systems | YES (scripting context) | YES (rendering context) |
| VPX element defaults | YES (physics values) | YES (data model) |

**Action required:** Update `vpx-editor` skill description from "Load when working on vpx-editor or VPX table development" to "Load when working on the vpx-editor codebase or Whitewood mode development."

_Requirements: 1.4_

---

## Skill Content Structure

Each SKILL.md follows a standard internal structure:

```markdown
---
name: vpx-dev-[topic]
description: [keyword-rich description]
version: "1.0.0"
last_updated: "2026-03-24"
---

# [Topic Title]

## When This Skill Applies
[Trigger conditions — what user queries activate this]

## Quick Reference
[Tables/checklists for common lookups — values, constants, defaults]

## Patterns
### [Pattern Name]
**When to use:** [scenario]
**Template:**
```vbs
[VBScript code template with placeholders]
```
**Gotchas:**
- [Common mistake 1]
- [Common mistake 2]
**Source:** [VPW wiki guide + table name]

## Anti-Patterns
### [Anti-Pattern Name]
**What it looks like:** [code or behavior]
**Why it's wrong:** [explanation]
**Fix:** [correct approach]
```

---

## Component Designs

### Component 1: Master Skill (`SKILL.md`)

**Purpose:** Entry point for all VPX table development queries. Provides workflow overview and routes to topic skills.

**Content sections:**
1. **Build workflow** — 9-step process: scope → scan → model → script → physics → lighting → sound → test → release
2. **Skill index** — one-line description of each topic skill
3. **Recreation vs Original** — different skill priorities for ROM-based vs non-ROM tables
4. **Critical globals** — values that must be set before ANY work: difficulty 50-56, ball mass 1, BallSize 50
5. **VPX version notes** — features by version (10.8 incandescent fader, 10.8.1 native tilt, DisableStaticPreRendering counter)
6. **Cross-reference directive** — explicit instruction: "When generating ANY VBScript, always apply the rules from the conventions skill (naming, timers, standalone compatibility, anti-patterns). This is not optional."
7. **Recreation vs Original routing** — when user specifies recreation (ROM-based): prioritize pinmame-wiring + game-logic. When original (non-ROM): prioritize game-logic + lighting (Light State Controller) + FlexDMD

**Token budget:** ~3.5k tokens

_Requirements: 10.1-10.6_

---

### Component 2: Game Logic Skill (`game-logic/SKILL.md`)

**Purpose:** Generate VBScript for game rules, modes, multiball, scoring, tilt, attract mode.

**Content sections:**

1. **Mode State Machine** — Start/Active/Complete pattern with template
2. **Multiball** — gBOT pattern (global ball array, never destroy balls), ball save, ball lock variants (physical trough replacement, visual lock, captive ball)
3. **Scoring** — per-player state save/restore, multi-player support template
4. **Tilt** — force-reset all modes, clear flipper flags, script-based + native 10.8.1 tilt
5. **Attract Mode** — FlexDMD scene arrays, StopSyncWithVpxLights
6. **Wizard Mode** — ball saver, multiball gating, reset on fail/complete, clean lights
7. **FlexMode** — state machine pattern (1=gameplay, 2=intro, etc.)
8. **Ball-in-Narnia Recovery** — detect/recover lost balls via position checks

**Quick reference table:**

| Pattern | Key Code | Gotcha |
|---------|----------|--------|
| SolCallback | `Sub SolX(enabled)` | Must include `If Enabled Then` guard |
| gBOT | `Dim gBOT()` | Use GetBalls ONLY in Cor.Update (standalone segfault) |
| Ball save | Timer-based, configurable duration | Max 10 returns for high ball counts |
| Mode reset | Force-reset in tilt handler | Clear flipper-held flags too |

**Token budget:** ~5k tokens

_Requirements: 2.1-2.10_

---

### Component 3: nFozzy Physics Skill (`nfozzy-physics/SKILL.md`)

**Purpose:** Configure physics correctly for any era. Rubber separation, flipper tuning, targets, materials.

**Content sections:**

1. **Implementation Checklist** — 9 steps (flipper meshes → rdampen → key subs → fire code → nFozzy chunk → check duplicates → rubbers → import .vpp → test)
2. **Flipper Tuning by Era** — polarity/velocity curves, strength values, EOS constants

   | Era | Strength | SOSRampup | EOSTorque | ReturnStrength |
   |-----|----------|-----------|-----------|----------------|
   | EM | 500-1000 | 2.5 | 0.3 | 0.048 |
   | Late 70s - Early 80s | 1400-1600 | 2.5 | 0.3 | 0.045 |
   | Mid 80s (System 11) | 2000-2600 | 2.5 | 0.275 | 0.045 |
   | Late 80s - Early 90s | 2000-2600 | 2.5 | 0.275 | 0.035 |
   | Mid 90s+ (WPC/Stern) | 3200-3300 | 2.5 | 0.3 | 0.018 |

3. **Flipper Trigger Sizing** — 27 VP margin (2024 standard), +3° end angle for sizing, hit height 150, PolarityCorrect in drain_hit
4. **Upper Flipper Simplification** — disable trajectory correction, keep flipper tricks
5. **Rubber Separation** — dPosts (bare), dSleeves (sleeved), bands. Materials: `z_col_rubberposts` (e=0.9), `z_col_rubberpostsleeves` (e=0.765), `z_col_rubberbands` (e=0.85). Rubber band rigging: 5-object structure (2 posts + 2 side walls + 1 middle wall)
6. **TargetBouncer** — factor 0.7, random 0.2-0.5, z<30. `TargetBounce_Hit(idx)` — idx parameter REQUIRED (silent fail without it). Exclusion zones: don't apply near bumpers or scoops
7. **Slingshot Correction** — 4-7° in `_Slingshot` event subs (NOT SolCallback), dsin/dcos helpers, threshold 1.5-2.0, sling threshold must match hit threshold
8. **FlipperCradleCollision** — COR dampening 0.4, all velocity components (x,y,z), both balls, when either on held flipper
9. **Global Settings** — difficulty 50-56 (NOT default 20), ramp exit spin multiplier 50 (NOT default 70), ball mass always 1, BallSize always 50

**Anti-patterns:**
- Global Physics Sets (silently override nFozzy)
- Ball mass ≠ 1 (breaks all flipper physics)
- Zeroed physics materials (re-import .vpp, detect via F11 dotted ball)
- Friction = 0 on any surface (use 0.1 minimum)

**Token budget:** ~6k tokens

_Requirements: 3.1-3.13_

---

### Component 4: PinMAME Wiring Skill (`pinmame-wiring/SKILL.md`)

**Purpose:** Wire ROM switches, coils, and lamps correctly for recreation tables.

**Content sections:**

1. **SolCallback Pattern** — callback signature, `If Enabled Then` guard, +100 offset for shared flasher/lamp numbers
2. **Switch Constants** — VPinMAME numbering, framework constant lookup (core.vbs, S11.VBS)
3. **Trough Setup** — physical trough with `CreateSizedBallWithMass`, switch mapping, drain kicker positioning
4. **UseVPMModSol** — 0/1/2 modes explained, era-appropriate recommendations
5. **FastFlips** — `InitVpmFFlipsSAM` for SAM, DE solenoid mapping (e.g., solenoid 47 for DE Tommy), ROM bypass fallback
6. **GI Dimming** — SolModCallback for ROM-controlled GI
7. **Expansion Hardware** — daughter cards, A/B relay, extended I/O at end of numbering
8. **HandleMech** — set `HandleMech=0` for scripted stepper motors (drawbridges, rotating elements)
9. **Game Definition Lookup** — check local clone of `vpinball/pinmame-game-defs`, guide user to clone if absent

**Token budget:** ~4k tokens

_Requirements: 4.1-4.10_

---

### Component 5: Lighting Skill (`lighting/SKILL.md`)

**Purpose:** GI fading, flasher effects, insert lighting, Light State Controller.

**Content sections:**

1. **GI String Management** — identification (wire tracing, frame-stepping, ROM test), per-string dimming, BlendDisableLighting, cross-dimming boost
2. **Flasher Decay** — exponential decay formula, per-element fade curve exponents, SolCallback vs SolModCallback
3. **Insert Lighting** — Toolkit lightmap inserts (current standard), ON/OFF/BULB naming convention, mixed colors (no pure RGB for realistic bulbs)
4. **Light Fading** — VPX 10.8+ native `Incandescent` fader (40ms up, 120ms down) as primary. Lampz FadeSpeed as fallback: LED (1/2 up, 1/8 down), incandescent (1/40 up, 1/120 down)
5. **Light State Controller** — LampTimer must be enabled, multi-state cycling, blink groups, Panlandia coordinate-based sequences
6. **Room Brightness** — "one method only" rule for darkening
7. **PWM Integration** — VPX 10.8+ native incandescent fading, Niwak's PinMAME PWM system

**Anti-patterns:**
- Pure black (#000000) in materials (divide-by-zero)
- Pure RGB insert colors (unrealistic)
- Lampz fading at frame rate (-1 interval — causes speed variation)

**Token budget:** ~4k tokens

_Requirements: 5.1-5.8_

---

### Component 6: Conventions Skill (`conventions/SKILL.md`)

**Purpose:** Naming, layers, timers, performance, standalone compatibility.

**Content sections:**

1. **Naming Conventions** — `bm_`, `lm_`, `pf_` prefixes, flasher naming (`f1` + solenoid), insert suffixes (`_on`/`_off`/`_bulb`), 31-character table name limit
2. **Layer Organization** — standard 11-layer structure
3. **Timer Architecture** — three-tier: GameTimer (10ms), FrameTimer (-1), LampTimer (fixed). Set interval BEFORE enabling. Never use `vpmTimer.Add`
4. **Timer Gotchas** — interval -1 extra execution after disable, frame-rate-dependent animation scaling, catch-up spiral prevention
5. **Performance Checklist** — interval-1 timers → -1, GetBalls → gBOT (except Cor.Update), render probe roughness, collidable primitive poly count, transparency optimization
6. **Standalone Compatibility** — gBOT for gameplay (GetBalls in Cor.Update only), FlexDMD auto-detection via error handling (not filesystem), Const declaration order (define before reference), `cSingleLFlip=0`/`cSingleRFlip=0` preservation
7. **VBScript Gotchas** — `Not 1` returns `-2` (use `= 0` instead of `Not`), duplicate Sub names silently override, `debug.print` in hot paths kills FPS
8. **Physics Defaults** — playfield friction 0.15-0.25, minimum friction 0.1 on anything, BallSize 50, ball mass 1, difficulty 50-56
9. **Production Checklist** — remove debug.print, verify BallSize/mass/difficulty, import .vpp materials, check for duplicate subs

**Token budget:** ~6k tokens (19 acceptance criteria, many with code examples)

_Requirements: 6.1-6.19_

---

### Component 7: Sound Skill (`sound/SKILL.md`)

**Purpose:** Fleep sounds, ball rolling, ramp transitions, SSF.

**Content sections:**

1. **Fleep Setup** — collections with "Fire events" enabled, collection mapping (dPost, dSleeve, GatesWire, Apron, Metals, Rubbers, Rollovers, Walls)
2. **Ball Rolling** — height threshold adjustment, ramp-specific ZRRL sounds, height-based volume
3. **Ramp Transitions** — explicitly stop previous rolling sound before starting new (plastic↔wire)
4. **Direction Detection** — `ActiveBall.vely` for direction-aware sounds
5. **Standalone Guards** — AudioPan property check before panning (prevents Fleep errors)
6. **SSF** — stereo vs mono conversion
7. **MP3 Preloading** — preload in `Table1_Init` to prevent first-play frame drops

**Token budget:** ~2.5k tokens

_Requirements: 7.1-7.7_

---

### Component 8: Troubleshooting Skill (`troubleshooting/SKILL.md`)

**Purpose:** Symptom → diagnosis → fix for 28+ documented issues.

**Content structure:** Organized by symptom (what the user reports), not by system (what's broken). Each entry:

```markdown
### Ball passes through walls
**Check:** Normals direction, collidable flag, duplicate collidable primitives, missing thickness
**Fix:** [specific steps]
**Source:** [wiki reference]
```

**Symptom catalog:**

| # | Symptom | Root Cause | Fix |
|---|---------|-----------|-----|
| 1 | Ball through walls | Wrong normals / not collidable | Flip normals, check collidable flag |
| 2 | Ball sinks through playfield | Collidable primitive as rest surface | Use VPX wall + under-playfield blocker Z=-0.01 |
| 3 | Flippers feel wrong | Duplicate Collide subs / wrong era curve / mass≠1 | Check dups, match era, mass=1 |
| 4 | Physics materials zeroed | VPX operation stripped .vpp | Re-import .vpp (F11 dotted ball test) |
| 5 | Frame rate drops | Timer interval 1 / GetBalls / render probes | Interval -1, gBOT, reduce probes |
| 6 | cor.BallVel error | Missing GameTimer / Cor.Update | Add GameTimer with Cor.Update |
| 7 | FlexDMD broken | Filesystem detection / scene recreation | Error handling detection, init once |
| 8 | NVRAM corrupted | Bad state saved | Clear .nv file |
| 9 | Z-fighting | Wrong depth bias | Top=negative, bottom=positive, or 99.99% scale |
| 10 | Sounds play twice | Missing If Enabled guard | Add `If Enabled Then` to SolCallback |
| 11 | Standalone crash on flipper | gBOT in Cor.Update | Replace with GetBalls in Cor.Update only |
| 12 | Options menu breaks rendering | DisableStaticPreRendering spammed | Set True once on entry, False once on exit |
| 13 | Flipper jam | Sub-pixel alignment | 1-pixel position shift |
| 14 | Ball hover near flipper | Playfield mesh loop cuts | Remove loop cuts in flipper area |
| 15 | Ramp exit corkscrew | Spin multiplier too high | Reduce from 70 to 50 |

**Token budget:** ~5.5k tokens (28+ entries at ~150 tokens each + structure/headers)

_Requirements: 8.1-8.12_

---

### Component 9: Toys & Mechanisms Skill (`toys-mechs/SKILL.md`)

**Purpose:** Script patterns for mechanical table features.

**Content sections:**

1. **Diverter** — SolCallback, wall/kicker activation, position tracking, sound
2. **Magnet** — ball capture, timed release, force application
3. **VUK** — KickZ parameters (angle, speed, inclination, heightz), timing
4. **Spinner** — opacity cross-fade between 0°/180° meshes in FrameTimer
5. **Drop Targets** — Roth animation system, DoDTAnim in FrameTimer, RotZ=0 facing drain
6. **Motor Mechanisms** — cvpmMech, Sol1/Sol2 wiring, limit switch handling, HandleMech=0
7. **Ball Philosophy** — real machines don't create/destroy balls. Use physical trough + gBOT
8. **Captive/Newton Ball** — oversized (57 wide) ball at post position, create/destroy on mechanism state. NOTE: game-logic skill covers captive ball as a ball LOCK pattern (scripting logic); this skill covers the PHYSICAL mechanism (mesh, sizing, position). No overlap — game-logic owns the "when to lock/release" logic, toys-mechs owns the "how the physical captive ball works"

**Token budget:** ~3k tokens

_Requirements: 9.1-9.8_

---

## Content Extraction Pipeline

### Source Material → Skill Content

For each skill file, the extraction process:

1. **Read relevant wiki guide(s)** — primary source
2. **Extract patterns** — each distinct technique becomes a Pattern section
3. **Extract anti-patterns** — each gotcha/warning becomes an Anti-Pattern section
4. **Extract constants/values** — populate Quick Reference tables
5. **Write code templates** — generalize table-specific VBS into reusable templates with placeholders
6. **Add source citations** — every pattern cites the wiki guide + original table where learned
7. **Verify against VPW Example Table** — cross-check code patterns against the reference implementation

### Source Mapping

| Skill | Primary Wiki Source(s) | Secondary Sources |
|-------|----------------------|-------------------|
| game-logic | `game-rules-mechanics.md`, `vbscript-patterns.md` | `example-table.md` |
| nfozzy-physics | `physics-tuning.md` | `example-table.md`, `best-practices.md` |
| pinmame-wiring | `vbscript-patterns.md` (ROM section) | `example-table.md` |
| lighting | `gi-and-flashers.md`, `insert-lighting.md`, `light-state-controller.md` | — |
| conventions | `best-practices.md`, `vbscript-patterns.md` (timer section) | `build-workflow.md` |
| sound | `fleep-sounds.md` | `vbscript-patterns.md` |
| troubleshooting | `troubleshooting.md` | All guides (cross-cutting) |
| toys-mechs | `vbscript-patterns.md` (mechanism section), `build-workflow.md` | Per-table extractions |

_Requirements: NFR Accuracy_

---

## Error Handling

Since this is a knowledge plugin (not software), "errors" are incorrect VBScript output.

| Error Type | Prevention | Detection |
|-----------|-----------|-----------|
| Wrong physics constants | Quick reference tables with exact values | User tests with F11 dotted ball |
| Wrong SolCallback pattern | Code templates with `If Enabled Then` baked in | Sounds playing twice = missing guard |
| Standalone incompatibility | Conventions skill standalone section | Crash on flipper hit = gBOT in Cor.Update |
| Wrong era flipper tuning | Era selection table with per-era values | Shot angles feel wrong = try different era curve |
| Missing timer architecture | Three-tier timer template | cor.BallVel error = missing GameTimer |

_Requirements: NFR Quality_

---

## Testing Strategy

### Verification Method

Each skill is verified by generating VBScript for a test scenario and checking correctness:

| Skill | Test Scenario | Pass Criteria |
|-------|-------------|--------------|
| game-logic | "3-ball multiball with ball save" | gBOT pattern, timer-based save, correct ball tracking |
| game-logic | "Implement tilt handling" | Force-reset modes, clear flipper flags, both script + native |
| nfozzy-physics | "Configure 90s WPC flipper" | Strength 2800-3000, polarity curve correct, trigger 27 VP |
| nfozzy-physics | "Set up rubber separation for 2 slingshots" | dPosts + dSleeves + bands, correct materials, 5-object assembly |
| nfozzy-physics | "Table uses Global Physics Sets" | Warning generated, recommends removing GPS |
| pinmame-wiring | "Wire T2 solenoids" | SolCallback array correct, If Enabled guards, trough setup |
| pinmame-wiring | "Set up trough for WPC table" | CreateSizedBallWithMass, switch mapping, drain kicker |
| lighting | "GI with 3 strings" | Per-string BlendDisableLighting, correct fade values |
| conventions | "Review script for standalone" | GetBalls in Cor.Update, Const order, cSingleLFlip preserved |
| conventions | "Is this recreation or original?" | Different skill prioritization (master routing test) |
| sound | "Fleep ball rolling + ramp" | Collections correct, height threshold, ramp transition stop |
| troubleshooting | "Ball passes through ramp" | Suggests normals check, collidable flag, mesh thickness |
| troubleshooting | "Frame rate is dropping" | Suggests timer interval check, GetBalls → gBOT, render probes |
| troubleshooting | "Standalone crashes on flipper hit" | Suggests gBOT → GetBalls in Cor.Update |
| toys-mechs | "Implement VUK" | KickZ params present, timing correct |

### Continuous Validation

When VPW releases new techniques or VPX adds features:
1. Check if existing skill content is affected
2. Update relevant skill file
3. Bump `version` and `last_updated` in frontmatter
4. Verify no contradictions introduced

_Requirements: NFR Maintenance_

---

## Token Budget Summary

| Skill | Estimated Tokens | Sections |
|-------|-----------------|----------|
| Master (SKILL.md) | ~3,500 | Workflow, index, globals, routing |
| object-reference | ~4,000 | All VPX object types, events, properties |
| glf | ~4,500 | GLF framework: devices, events, modes, shots |
| core-vbs | ~3,500 | core.vbs API classes and patterns |
| flexdmd | ~5,000 | FlexDMD display, scenes, animations |
| game-logic | ~5,000 | 8 patterns + quick ref |
| nfozzy-physics | ~6,000 | 9 sections + era tables + reference data |
| pinmame-wiring | ~4,000 | 10 sections |
| lighting | ~4,000 | 7 sections + anti-patterns |
| conventions | ~6,000 | 9 sections + 19 ACs + checklists |
| sound | ~2,500 | 7 sections |
| troubleshooting | ~5,500 | 28+ symptom entries |
| toys-mechs | ~3,000 | 8 mechanism patterns |
| **Total** | **~56,500** | |

At ~56.5k tokens total, if all skills loaded simultaneously they'd use ~5.6% of a 1M context window. In practice, only 2-3 skills load per query (~8-15k tokens, <1.5%).

---

## Implementation Order

### Decision: Build Order

**Decision:** Foundation-first — master skill + conventions, then high-impact topic skills, then specialized skills.

**Phase 1:** Master + Conventions + Troubleshooting
- Master provides entry point and routing
- Conventions provides rules every other skill depends on
- Troubleshooting is the most immediately useful (users hit bugs first)

**Phase 2:** nFozzy Physics + Game Logic
- Highest complexity, highest value
- Most code template content

**Phase 3:** PinMAME Wiring + Lighting + Sound + Toys
- Lower complexity
- Can be built in parallel (no cross-dependencies)

**Rationale:** Phase 1 skills are useful standalone. Phase 2 skills are the core value proposition. Phase 3 fills out the complete picture.
