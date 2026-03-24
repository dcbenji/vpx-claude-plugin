# Implementation Plan: VPX Dev Plugin

Content creation project — writing skill files by distilling VPW wiki guides into structured Claude Code skills. Organized by the 3-phase build order from the design doc.

**Process per skill file:**
1. Read the mapped wiki guide(s) from the design doc's Source Mapping table
2. Extract patterns, anti-patterns, constants, and code templates
3. Write the SKILL.md following the standard content structure (frontmatter, When This Applies, Quick Reference, Patterns, Anti-Patterns)
4. Verify: ask Claude a test question from the design doc's Testing Strategy table, confirm correct output
5. Check token count stays within budget

**File ownership:** Each task creates one file. No overlaps. Tasks within a phase can run in parallel.

---

## Phase 1: Foundation (no dependencies)

### Task 1: Master Skill

- [ ] 1.1 Create master skill file
  - Create `~/.claude/skills/vpx-dev/SKILL.md`
  - Frontmatter: name `vpx-dev`, version `1.0.0`, last_updated `2026-03-24`, description with ALL master keywords from design doc
  - Sections:
    - Build workflow (9 steps: scope → scan → model → script → physics → lighting → sound → test → release)
    - Skill index (one-line description of each of the 8 topic skills)
    - Critical globals table: difficulty 50-56, ball mass 1, BallSize 50, minimum friction 0.1, no pure black
    - VPX version feature matrix (10.8: incandescent fader; 10.8.1: native tilt, DisableStaticPreRendering counter)
    - Cross-reference directive: "When generating ANY VBScript, always apply conventions skill rules"
    - Recreation vs Original routing guidance
  - Token budget: ~3,500
  - _Requirements: 10.1-10.6, 1.1-1.5_

**Self-test:**
- [ ] [RUNTIME] Ask Claude "I'm starting a new VPX table" → presents build workflow
- [ ] [RUNTIME] Ask Claude "what VPX skills are available?" → lists all 8 topic skills
- [ ] [RUNTIME] Ask Claude "I'm building a recreation of Medieval Madness" → prioritizes PinMAME wiring + game logic
- [ ] [METRIC] File under ~3,500 tokens (`wc -w` × 1.3)

---

### Task 2: Conventions Skill

- [ ] 2.1 Create conventions skill file
  - Create `~/.claude/skills/vpx-dev/conventions/SKILL.md`
  - Read: `best-practices.md`, `vbscript-patterns.md` (timer section), `build-workflow.md`
  - Frontmatter: name `vpx-dev-conventions`, version `1.0.0`, last_updated `2026-03-24`, description with conventions keywords
  - Sections:
    - Naming conventions: `bm_`/`lm_`/`pf_` prefixes, flasher naming (`f1`+solenoid), insert suffixes (`_on`/`_off`/`_bulb`), 31-char table name limit
    - Layer organization: standard 11-layer structure
    - Timer architecture: three-tier (GameTimer 10ms, FrameTimer -1, LampTimer fixed). Set interval BEFORE enabling. Never use `vpmTimer.Add`. Timer -1 extra execution bug. Frame-rate-dependent animation scaling
    - Performance checklist: interval-1→-1, GetBalls→gBOT, render probe roughness, collidable primitive poly count, transparency, debug.print removal
    - Standalone compatibility: gBOT for gameplay (GetBalls in Cor.Update ONLY), FlexDMD auto-detection via error handling, Const declaration order, `cSingleLFlip=0`/`cSingleRFlip=0` preservation
    - VBScript gotchas: `Not 1` returns `-2`, duplicate Sub names silently override, `debug.print` in hot paths
    - Physics defaults: playfield friction 0.15-0.25, minimum 0.1 on anything, BallSize 50, ball mass 1, difficulty 50-56
    - Production checklist: remove debug.print, verify BallSize/mass/difficulty, import .vpp, check duplicate subs, verify Const order
  - Token budget: ~6,000
  - _Requirements: 6.1-6.19_

**Self-test:**
- [ ] [RUNTIME] Ask Claude "review this script for standalone compatibility" with a script using GetBalls in Cor.Update → warns about segfault, recommends keeping GetBalls there but using gBOT elsewhere
- [ ] [RUNTIME] Ask Claude to generate a timer → sets interval before enabling
- [ ] [RUNTIME] Generate any VBScript → follows naming conventions
- [ ] [METRIC] File under ~6,000 tokens (`wc -w` × 1.3)

---

### Task 3: Troubleshooting Skill

- [ ] 3.1 Create troubleshooting skill file
  - Create `~/.claude/skills/vpx-dev/troubleshooting/SKILL.md`
  - Read: `troubleshooting.md` (primary), all other guides for cross-cutting error patterns
  - Frontmatter: name `vpx-dev-troubleshooting`, version `1.0.0`, last_updated `2026-03-24`, description with troubleshooting keywords (including "frame rate fps performance standalone z-fighting NVRAM FlexDMD sounds")
  - Structure: symptom → diagnosis → fix (organized by what user reports)
  - Must cover ALL 28+ documented patterns. Minimum entries:
    1. Ball through walls (normals, collidable flag, duplicates, thickness)
    2. Ball sinks through playfield (blocker wall Z=-0.01, use VPX walls not primitives)
    3. Flippers wrong (duplicate Collide subs, trigger 27VP, era curve, mass≠1, GPS conflict)
    4. Physics materials zeroed (re-import .vpp, F11 dotted ball test)
    5. Frame rate drops (timer interval 1, GetBalls, render probes, collidable poly count, transparency)
    6. cor.BallVel error (missing GameTimer with Cor.Update)
    7. FlexDMD broken (error handling detection, scene array init, timer stays enabled)
    8. NVRAM corrupted (clear .nv file)
    9. Z-fighting (depth bias: top=negative, bottom=positive; 99.99% scale)
    10. Sounds play twice (missing If Enabled guard in SolCallback)
    11. Standalone crash on flipper (gBOT in Cor.Update → use GetBalls there only)
    12. Options menu breaks rendering (DisableStaticPreRendering counter: True once on entry, False once on exit)
    13. Flipper jam (sub-pixel alignment, 1-pixel shift)
    14. Ball hover near flipper (playfield mesh loop cuts)
    15. Ramp exit corkscrew (spin multiplier 70→50)
    16. Bumpers/slings permanently broken after tilt+crash (VPX saves state on exit)
    17. Duplicate collidable primitives (invisible physics doubles)
    18. Ramp friction too high (0.8 = velcro, unify at ~0.2)
    19. Pop bumper radius wrong (radius ≠ visual hat size)
    20. Timer catch-up spiral (consolidate into frametimer)
    21. vpmTimer.Add broken (never use)
    22. Fleep AudioPan crash on standalone (property check guard)
    23. Ball-in-Narnia (position check recovery)
    24. Lightmap pruning threshold (0.01)
    25. Render probe roughness VR artifacts
    26. Transparency performance bottleneck
    27. 32-bit VPX texture memory limit
    28. Ball collision sounds excessive in multiball (ID tracking + timer)
  - Each entry: symptom + check steps + fix + source citation
  - Token budget: ~5,500
  - _Requirements: 8.1-8.12_

**Self-test:**
- [ ] [RUNTIME] Ask "ball passes through ramp" → suggests normals, collidable flag, mesh thickness
- [ ] [RUNTIME] Ask "frame rate is dropping" → suggests timer interval, GetBalls→gBOT, render probes
- [ ] [RUNTIME] Ask "standalone crashes on flipper hit" → suggests gBOT→GetBalls in Cor.Update
- [ ] [METRIC] File under ~5,500 tokens (`wc -w` × 1.3)

---

## Phase 2: Core Value (needs Phase 1 for conventions cross-reference)

### Task 4: nFozzy Physics Skill

- [ ] 4.1 Create nFozzy physics skill file
  - Create `~/.claude/skills/vpx-dev/nfozzy-physics/SKILL.md`
  - Read: `physics-tuning.md` (primary), `example-table.md`, `best-practices.md`
  - Frontmatter: name `vpx-dev-nfozzy-physics`, version `1.0.0`, last_updated `2026-03-24`, description with physics keywords
  - Sections:
    - 9-step implementation checklist
    - Flipper tuning by era (table with strength, SOSRampup, EOSTorque, EOSReturn per era)
    - Flipper trigger sizing (27 VP, +3° end angle, hit height 150, PolarityCorrect in drain_hit)
    - Upper flipper simplification (no trajectory correction, keep tricks)
    - Rubber separation (dPosts/dSleeves/bands, materials with exact values: e, falloff, friction)
    - 5-object rubber band rigging template
    - TargetBouncer (factor 0.7, random 0.2-0.5, z<30, idx parameter required, exclusion zones)
    - Slingshot correction (4-7° in _Slingshot events, dsin/dcos, threshold matching)
    - FlipperCradleCollision (COR 0.4, all components, both balls)
    - Global settings (difficulty 50-56, ramp exit spin 50, mass 1, BallSize 50)
    - Anti-patterns: GPS, mass≠1, zeroed materials, friction=0
  - Include VBS code templates for: nFozzy chunk, flipper key subs, rubber separation, TargetBouncer
  - Token budget: ~6,000
  - _Requirements: 3.1-3.13_

**Self-test:**
- [ ] [RUNTIME] Ask "configure 90s WPC flipper" → strength 2800-3000, correct polarity curve, trigger 27 VP
- [ ] [RUNTIME] Ask "set up rubber separation for slingshots" → dPosts + bands, correct materials, 5 objects
- [ ] [RUNTIME] Ask "table uses Global Physics Sets" → warning generated
- [ ] [METRIC] File under ~6,000 tokens (`wc -w` × 1.3)

---

### Task 5: Game Logic Skill

- [ ] 5.1 Create game logic skill file
  - Create `~/.claude/skills/vpx-dev/game-logic/SKILL.md`
  - Read: `game-rules-mechanics.md` (primary), `vbscript-patterns.md`, `example-table.md`
  - Frontmatter: name `vpx-dev-game-logic`, version `1.0.0`, last_updated `2026-03-24`, description with game logic keywords (including "ball save tilt attract drain")
  - Sections:
    - Mode state machine (Start/Active/Complete template)
    - Multiball (gBOT pattern template — with NOTE: GetBalls in Cor.Update only)
    - Ball save (timer-based template, max return limits)
    - Ball lock variants (physical trough replacement, visual lock, captive ball scripting logic)
    - Scoring (per-player state save/restore template)
    - Tilt handling (force-reset modes, clear flipper flags, script + native 10.8.1)
    - Attract mode (FlexDMD scene arrays, StopSyncWithVpxLights)
    - Wizard mode (ball saver, multiball gating, reset, clean lights)
    - FlexMode state machine pattern
    - Ball-in-Narnia recovery (position check loop)
    - Quick reference table: pattern → key code → gotcha
  - Include VBS code templates for each pattern
  - Token budget: ~5,000
  - _Requirements: 2.1-2.10_

**Self-test:**
- [ ] [RUNTIME] Ask "implement 3-ball multiball with ball save" → gBOT, timer-based save, correct tracking
- [ ] [RUNTIME] Ask "implement tilt handling" → force-reset modes, clear flipper flags
- [ ] [RUNTIME] Generated SolCallback has `If Enabled Then` guard
- [ ] [METRIC] File under ~5,000 tokens (`wc -w` × 1.3)

---

## Phase 3: Specialized (can run in parallel, no cross-dependencies)

### Task 6: PinMAME Wiring Skill

- [ ] 6.1 Create PinMAME wiring skill file
  - Create `~/.claude/skills/vpx-dev/pinmame-wiring/SKILL.md`
  - Read: `vbscript-patterns.md` (ROM section), `example-table.md`
  - Frontmatter: name `vpx-dev-pinmame-wiring`, version `1.0.0`, last_updated `2026-03-24`, description with wiring keywords (including "trough drain GI")
  - Sections:
    - SolCallback pattern (signature, If Enabled guard, +100 offset)
    - Switch constants (VPinMAME numbering, framework constant lookup)
    - Trough setup template (CreateSizedBallWithMass, switch mapping, drain kicker)
    - UseVPMModSol (0/1/2 explained, era recommendations)
    - FastFlips (InitVpmFFlipsSAM, DE solenoid mapping, ROM bypass)
    - GI dimming (SolModCallback)
    - Expansion hardware (daughter cards, A/B relay, extended I/O)
    - HandleMech=0 for stepper motors
    - Game definition lookup (local pinmame-game-defs clone)
  - Token budget: ~4,000
  - _Requirements: 4.1-4.10_

**Self-test:**
- [ ] [RUNTIME] Ask "wire T2 solenoids" → correct SolCallback array, If Enabled guards
- [ ] [RUNTIME] Ask "set up trough for WPC table" → CreateSizedBallWithMass template
- [ ] [RUNTIME] Ask "how do I set up FastFlips for a SAM table?" → InitVpmFFlipsSAM
- [ ] [METRIC] File under ~4,000 tokens (`wc -w` × 1.3)

---

### Task 7: Lighting Skill

- [ ] 7.1 Create lighting skill file
  - Create `~/.claude/skills/vpx-dev/lighting/SKILL.md`
  - Read: `gi-and-flashers.md`, `insert-lighting.md`, `light-state-controller.md`
  - Frontmatter: name `vpx-dev-lighting`, version `1.0.0`, last_updated `2026-03-24`, description with lighting keywords (including "Light State Controller")
  - Sections:
    - GI string management (identification, per-string dimming, BlendDisableLighting, cross-dimming)
    - Flasher decay (exponential formula, per-element exponents, SolCallback vs SolModCallback)
    - Insert lighting (Toolkit lightmap standard, ON/OFF/BULB naming, mixed colors not pure RGB)
    - Light fading: native Incandescent fader (10.8+, primary) vs Lampz FadeSpeed (fallback)
    - Light State Controller (LampTimer requirement, multi-state cycling, blink groups, Panlandia)
    - Room brightness ("one method only")
    - PWM integration (10.8+ native, Niwak's PinMAME PWM)
    - Anti-patterns: pure black (#000000), pure RGB inserts, Lampz at frame rate
  - Token budget: ~4,000
  - _Requirements: 5.1-5.8_

**Self-test:**
- [ ] [RUNTIME] Ask "set up GI with 3 strings" → per-string BlendDisableLighting, correct fade
- [ ] [RUNTIME] Ask "how do I fade inserts?" → recommends native Incandescent fader for 10.8+
- [ ] [METRIC] File under ~4,000 tokens (`wc -w` × 1.3)

---

### Task 8: Sound Skill

- [ ] 8.1 Create sound skill file
  - Create `~/.claude/skills/vpx-dev/sound/SKILL.md`
  - Read: `fleep-sounds.md` (primary), `vbscript-patterns.md`
  - Frontmatter: name `vpx-dev-sound`, version `1.0.0`, last_updated `2026-03-24`, description with sound keywords (including "Fleep")
  - Sections:
    - Fleep setup (collections, "Fire events", collection mapping table)
    - Ball rolling (height threshold, ramp ZRRL sounds, height-based volume)
    - Ramp transitions (stop previous before starting new)
    - Direction detection (ActiveBall.vely)
    - Standalone guards (AudioPan property check)
    - SSF (stereo vs mono)
    - MP3 preloading (Table1_Init pattern)
  - Token budget: ~2,500
  - _Requirements: 7.1-7.7_

**Self-test:**
- [ ] [RUNTIME] Ask "set up Fleep ball rolling" → collections, height threshold, correct code
- [ ] [RUNTIME] Generated code includes AudioPan guard for standalone
- [ ] [METRIC] File under ~2,500 tokens (`wc -w` × 1.3)

---

### Task 9: Toys & Mechanisms Skill

- [ ] 9.1 Create toys & mechanisms skill file
  - Create `~/.claude/skills/vpx-dev/toys-mechs/SKILL.md`
  - Read: `vbscript-patterns.md` (mechanism section), `build-workflow.md`
  - Frontmatter: name `vpx-dev-toys-mechs`, version `1.0.0`, last_updated `2026-03-24`, description with mechanism keywords (including "captive ball")
  - Sections:
    - Diverter (SolCallback, wall/kicker activation, position tracking, sound)
    - Magnet (ball capture, timed release, force application)
    - VUK (KickZ parameters template)
    - Spinner (opacity cross-fade, 0°/180° meshes, FrameTimer)
    - Drop targets (Roth animation, DoDTAnim, RotZ=0 facing drain)
    - Motor mechanisms (cvpmMech, Sol1/Sol2, limit switches, HandleMech=0)
    - Ball philosophy ("real machines don't create balls" — trough replacement + gBOT)
    - Captive/Newton ball (physical mechanism: 57-wide ball, create/destroy on state). NOTE: game-logic owns lock/release scripting, this skill owns physical mechanism
  - Token budget: ~3,000
  - _Requirements: 9.1-9.8_

**Self-test:**
- [ ] [RUNTIME] Ask "implement a VUK" → KickZ params, timing, correct code
- [ ] [RUNTIME] Ask "how do I create balls for multiball?" → explains trough replacement, gBOT
- [ ] [METRIC] File under ~3,000 tokens (`wc -w` × 1.3)

---

## Phase 4: Integration & Cleanup

### Task 10: vpx-editor Skill Scope Fix

- [ ] 10.1 Update vpx-editor skill description
  - Edit `~/.claude/skills/vpx-editor/skill.md`
  - Change description from "Load when working on vpx-editor or VPX table development" to "Load when working on the vpx-editor codebase or Whitewood mode development"
  - Verify no content overlap with new VPX Dev Plugin skills
  - _Requirements: 1.4_

**Self-test:**
- [ ] [RUNTIME] Ask "how do I set up nFozzy flippers?" → vpx-dev-nfozzy-physics activates, NOT vpx-editor

---

### Task 11: Cross-Skill Verification

- [ ] 11.1 Run full test scenario suite
  - Execute ALL 15 test scenarios from design doc's Testing Strategy table
  - For each: ask Claude the question, verify the output matches pass criteria
  - Log results in verification doc
  - Fix any skill that produces wrong output
  - _Requirements: all_

- [ ] 11.2 Token budget verification
  - Count tokens in each skill file (use `wc -w` as rough proxy: words × 1.3 ≈ tokens)
  - Verify each file stays within its budget from the design doc
  - If over budget: trim examples, consolidate redundant patterns, move edge cases to anti-patterns
  - _Requirements: NFR_

- [ ] 11.3 Keyword activation verification
  - Test 20 natural-language queries across all topics
  - Verify correct skill(s) activate for each query
  - If a query misses: add keywords to the relevant skill's description
  - _Requirements: 1.1, 1.2, 1.3_

**Self-test:**
- [ ] [RUNTIME] All 15 design doc test scenarios pass
- [ ] [METRIC] No skill file exceeds its token budget
- [ ] [RUNTIME] 20/20 natural-language queries activate the correct skill(s)
