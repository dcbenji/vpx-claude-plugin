# Requirements: VPX Dev Plugin

A Claude Code plugin (collection of skills) that enables AI-assisted VPX table development. Helps novice designers create game rules, wire PinMAME ROMs, configure physics, manage lighting, and script table logic — without needing to know VBScript.

## Target Users

1. **Novice table designer** — wants to build an original or recreation table, knows the physical game but not VBScript
2. **Intermediate author** — knows some VBScript, needs help with complex patterns (multiball, mode stacking, nFozzy)
3. **Experienced author** — uses Claude for productivity (bulk operations, debugging, optimization)

## Constraints

- Generated VBScript must work in VPX 10.8+ standalone (Mac/Linux/Windows)
- Each skill's `description` field must contain keywords matching common VPX queries so skills activate via Claude Code's description-matching system (NOT directory scanning — Claude Code skills match on description keywords, not working directory contents)
- Source of truth: VPW wiki guides (distilled from 88 Discord channels, 166+ patterns)
- Plugin must not conflict with the existing `vpx-editor` skill (which covers Whitewood/editor development only — its description must be scoped to "vpx-editor codebase development", NOT "VPX table development")

---

## Requirement 1: Plugin Architecture

**User Story:** As a Claude Code user working on a VPX table, I want relevant VPX knowledge to automatically load when I ask about table development topics, so I don't have to manually invoke skills.

**Acceptance Criteria:**
1. WHEN a user asks about VBScript patterns, game rules, physics, lighting, ROM wiring, sounds, or VPX table development THEN the relevant skill(s) SHALL activate via Claude Code's description-keyword matching
2. Each skill's `description` field SHALL contain broad keyword coverage: "VPX VBScript pinball table scripting [topic-specific keywords]" — so any VPX-related query triggers activation
3. WHEN multiple sub-skills are relevant to a query THEN Claude Code's skill system SHALL load all matching skills (this is native behavior — no custom routing needed)
4. IF the `vpx-editor` skill is also installed THEN the plugin SHALL not conflict. **Dependency:** The `vpx-editor` skill description must be narrowed to scope it to "vpx-editor codebase and Whitewood mode development" only (remove any "VPX table development" language)
5. The plugin SHALL be structured as a single `~/.claude/skills/vpx-dev/` directory containing either one comprehensive SKILL.md or multiple topic-specific SKILL.md files in subdirectories

**Priority:** High | **Complexity:** Medium | **Dependencies:** None

---

## Requirement 2: Game Logic Skill

**User Story:** As a novice designer, I want to describe my table's rules in plain English and have Claude generate correct VBScript, so I can implement game modes without learning scripting.

**Acceptance Criteria:**
1. WHEN user describes a mode (e.g., "when all 3 drop targets are down, start multiball") THEN the system SHALL generate correct VBScript using the Start/Active/Complete pattern
2. WHEN generating multiball logic THEN the system SHALL use the gBOT pattern (global ball array, never destroy balls). NOTE: `gBOT` is correct for gameplay code, but `Cor.Update` specifically must use `GetBalls` (gBOT causes segfaults in standalone's Cor.Update — see Req 8.11)
3. WHEN generating ball save logic THEN the system SHALL implement correct timer-based ball save with configurable duration
4. WHEN generating ball lock mechanics THEN the system SHALL support physical trough replacement, visual lock, and captive ball patterns
5. WHEN generating scoring logic THEN the system SHALL implement per-player state save/restore for multi-player support
6. WHEN generating tilt handling THEN the system SHALL force-reset all active modes, clear flipper flags, and handle both script-based and native 10.8.1 tilt
7. WHEN generating attract mode THEN the system SHALL use FlexDMD scene arrays (initialized once, not recreated) with `StopSyncWithVpxLights` on game start
8. WHEN generating wizard mode THEN the system SHALL include ball saver, proper multiball gating, reset on fail/complete, and clean light management (no "Christmas tree")
9. WHEN a user asks about mode state tracking THEN the system SHALL recommend the FlexMode state machine pattern
10. WHEN generating VBScript THEN the system SHALL include the `If Enabled Then` guard in all SolCallback subs

**Priority:** High | **Complexity:** High | **Dependencies:** Req 6 (conventions)

---

## Requirement 3: nFozzy Physics Skill

**User Story:** As a table designer, I want Claude to configure nFozzy physics correctly for my table's era, so flippers, rubbers, and targets feel realistic without manual tuning.

**Acceptance Criteria:**
1. WHEN user specifies a table era (70s, 80s, 90s, modern) THEN the system SHALL recommend correct polarity/velocity curves, flipper strength, and EOS values
2. WHEN configuring rubber separation THEN the system SHALL generate correct dPosts, dSleeves, and band configurations with proper materials (`z_col_rubberposts`, `z_col_rubberpostsleeves`, `z_col_rubberbands`)
3. WHEN configuring flippers THEN the system SHALL set trigger margin to 27 VP units (2024 standard), add 3° to end angle for trigger sizing, and include PolarityCorrect in drain_hit. Trigger hit height must be set to 150
4. WHEN the table has upper flippers THEN the system SHALL disable trajectory correction but keep flipper tricks
5. WHEN configuring drop targets THEN the system SHALL apply TargetBouncer (factor 0.7, random 0.2-0.5, z < 30) and include DoDTAnim in FrameTimer. SHALL warn that `TargetBounce_Hit(idx)` sub signature MUST include the `(idx)` parameter (omitting it causes silent failure with no error message)
6. WHEN configuring slingshots THEN the system SHALL place correction in `_Slingshot` event subs (NOT SolCallback — SolCallback is suppressed by built-in cooldown), include `dsin`/`dcos` helpers, apply 4-7° correction, set threshold to 1.5-2.0, and ensure sling threshold matches hit threshold
7. WHEN user asks about physics THEN the system SHALL warn against Global Physics Sets, ball mass ≠ 1, and interval-1 timers
8. WHEN generating the nFozzy implementation THEN the system SHALL follow the 9-step checklist (flipper meshes → rdampen → key subs → fire code → nFozzy chunk → check duplicates → rubbers → import .vpp → test)
9. WHEN configuring flipper cradling THEN the system SHALL include FlipperCradleCollision with COR dampening factor 0.4 in OnBallBallCollision, dampening all velocity components (x, y, z) of both balls when either ball is on a flipper held at end angle
10. WHEN the user's physics materials show zeroed-out values THEN the system SHALL suggest re-importing the .vpp file (detection: F11 dotted ball, abnormal inlane spin)
11. WHEN setting up a new table for physics tuning THEN the system SHALL recommend setting difficulty to 50-56 range (NOT the VPX default of 20) before any physics tuning — all tuning done at wrong difficulty will be wrong
12. WHEN the user reports unrealistic ball behavior exiting ramps THEN the system SHALL check the ramp exit spin multiplier (default 70, recommend 50)
13. WHEN applying TargetBouncer THEN the system SHALL warn against placing it on targets or posts in close proximity to pop bumpers or scoops (causes ball-through-geometry issues in exclusion zones)

**Priority:** High | **Complexity:** High | **Dependencies:** None

---

## Requirement 4: PinMAME Wiring Skill

**User Story:** As a recreation table builder, I want Claude to help me wire ROM switches, coils, and lamps correctly, so the game logic matches the real machine.

**Acceptance Criteria:**
1. WHEN user identifies the ROM (e.g., "T2 WPC") THEN the system SHALL look up the game definition from a local clone of `vpinball/pinmame-game-defs` if present in the workspace or user's filesystem, otherwise SHALL guide the user to clone it or provide common mappings from embedded knowledge
2. WHEN wiring SolCallbacks THEN the system SHALL use the correct callback pattern with `enabled` parameter and `If Enabled Then` guard
3. WHEN flashers share numbers with lamp assignments THEN the system SHALL use +100 offset convention
4. WHEN configuring `UseVPMModSol` THEN the system SHALL explain the 0/1/2 modes and recommend the correct one for the table's hardware era
5. WHEN the table uses expansion hardware (daughter cards, A/B relay) THEN the system SHALL explain that PinMAME appends extended I/O at end of numbering
6. WHEN user needs switch mapping THEN the system SHALL generate switch constants matching VPinMAME numbering (not physical machine numbering, unless they match)
7. WHEN configuring the trough THEN the system SHALL implement proper physical trough with `CreateSizedBallWithMass` at kicker positions, correct switch mapping, and drain kicker at physical bottom
8. WHEN ROM-controlled GI is needed THEN the system SHALL configure SolModCallback for dimming with correct PWM integration
9. WHEN configuring flipper response for ROM tables THEN the system SHALL recommend the appropriate FastFlips implementation: `InitVpmFFlipsSAM` for SAM tables, correct solenoid number mapping for Data East (e.g., solenoid 47 for DE Tommy), and ROM bypass as a fallback when standard FastFlips fails
10. WHEN the table uses scripted stepper motor mechanisms (drawbridges, rotating elements) THEN the system SHALL set `HandleMech=0` in the table script to prevent PinMAME mech handling interference

**Priority:** High | **Complexity:** Medium | **Dependencies:** Req 6 (conventions)

---

## Requirement 5: Lighting Skill

**User Story:** As a table designer, I want Claude to help me set up GI fading, flasher effects, and insert lighting, so the table has realistic lamp behavior.

**Acceptance Criteria:**
1. WHEN configuring GI strings THEN the system SHALL help identify string groupings (wire tracing, frame-stepping, ROM test) and set up per-string dimming with correct BlendDisableLighting values
2. WHEN configuring flasher decay THEN the system SHALL use the exponential decay formula with per-element fade curve exponents
3. WHEN setting up insert lighting THEN the system SHALL recommend the current standard (Toolkit lightmap inserts) and explain the ON/OFF/BULB naming convention
4. WHEN configuring light fading for VPX 10.8+ tables THEN the system SHALL recommend the native `Incandescent` fader type on VPX light objects (40ms up, 120ms down) as the preferred approach. For pre-10.8 compatibility or advanced control, Lampz FadeSpeed values are the fallback: LED (1/2 up, 1/8 down), incandescent (1/40 up, 1/120 down)
5. WHEN the table uses the Light State Controller THEN the system SHALL ensure LampTimer is enabled and explain multi-state cycling, blink groups, and Panlandia sequences
6. WHEN GI affects rubber/ramp brightness THEN the system SHALL configure cross-dimming boost calculations
7. WHEN the user asks about room brightness THEN the system SHALL explain the "one method only" rule for darkening
8. WHEN insert colors look wrong THEN the system SHALL warn against pure RGB values and recommend mixed colors for realistic bulb appearance

**Priority:** Medium | **Complexity:** Medium | **Dependencies:** Req 4 (PinMAME wiring for ROM-controlled lights)

---

## Requirement 6: Conventions Skill

**User Story:** As a table author, I want Claude to follow VPW naming conventions, layer organization, and performance best practices, so my table is compatible with the community ecosystem.

**Acceptance Criteria:**
1. WHEN generating VBScript THEN the system SHALL follow VPW naming conventions: `bm_` (bakemap), `lm_` (lightmap), `pf_` (playfield) prefixes for primitives, `f` + "1" + solenoid number for flashers, `_on`/`_off`/`_bulb` suffixes for inserts
2. WHEN organizing layers THEN the system SHALL recommend the standard 11-layer structure
3. WHEN creating timers THEN the system SHALL use the three-tier architecture: GameTimer (10ms, physics), FrameTimer (-1, visual), LampTimer (fixed interval, lights)
4. WHEN user has performance issues THEN the system SHALL check for: interval-1 timers, GetBalls calls, excessive render probes, high-poly collidable primitives, unoptimized transparency
5. WHEN the table name exceeds 31 characters THEN the system SHALL warn about VPX truncation
6. WHEN configuring playfield friction THEN the system SHALL recommend 0.15-0.25 range
7. WHEN generating timer code THEN the system SHALL set interval BEFORE enabling (avoids default-interval first tick bug)
8. WHEN the user is using `-1` timer intervals for animations THEN the system SHALL recommend frame-rate-dependent scaling factors to prevent speed variation across systems
9. WHEN generating code that will run on standalone THEN the system SHALL ensure gBOT compatibility for gameplay code (not Cor.Update — see Req 8.11) and FlexDMD auto-detection via error handling (not filesystem checks)
10. WHEN generating or cleaning up table scripts THEN the system SHALL preserve `cSingleLFlip=0` and `cSingleRFlip=0` constants even if they appear unused (required by system VBS files — removing them silently breaks flipper/solenoid management)
11. WHEN the user creates materials or textures THEN the system SHALL warn against pure black (#000000) which causes divide-by-zero in renderers and broken lighting — recommend #111111 as the minimum dark value
12. WHEN generating boolean checks on variables that may be integer 0/1 (service menu values, config flags) THEN the system SHALL use `If variable = 0 Then` instead of `If Not variable Then` (VBScript boolean gotcha: `Not 1` returns `-2`, not `0`)
13. WHEN setting friction on any VPX object THEN the system SHALL never use 0 — use 0.1 as the minimum. Zero friction causes unpredictable ball behavior
14. WHEN reviewing code for production release THEN the system SHALL flag `debug.print` statements in hot code paths (ball rolling subs, timers) — I/O overhead causes visible frame drops
15. WHEN generating or reviewing VBScript THEN the system SHALL warn about duplicate Sub/Function names — VBScript silently overrides duplicates with the last definition (not just flipper Collide subs — ANY duplicate sub name)
16. WHEN generating timer code THEN the system SHALL never use `vpmTimer.Add` — it is broken and causes cryptic errors
17. WHEN disabling timers with interval -1 THEN the system SHALL account for the extra-execution bug: VPX timers with interval -1 execute one additional time AFTER being disabled (disable takes effect on next screen refresh, not immediately)
18. WHEN generating standalone-compatible code THEN the system SHALL ensure all `Const` declarations appear BEFORE any code that references them (standalone crashes if a Const is referenced before its definition — unlike Windows VPX which is more forgiving)
19. WHEN validating table physics settings THEN the system SHALL verify `BallSize = 50` (not legacy value of 25, which produces wrong physics)

**Priority:** High | **Complexity:** Low | **Dependencies:** None

---

## Requirement 7: Sound Skill

**User Story:** As a table designer, I want Claude to help me set up Fleep sounds, ball rolling, and mechanical sound effects correctly.

**Acceptance Criteria:**
1. WHEN configuring Fleep sounds THEN the system SHALL set up collections with "Fire events" enabled and correct collection mapping (dPost, dSleeve, GatesWire, Apron, Metals, Rubbers, Rollovers, Walls)
2. WHEN implementing ball rolling sounds THEN the system SHALL use height threshold adjustment and ramp-specific ZRRL sounds with height-based volume
3. WHEN implementing ramp transition sounds THEN the system SHALL explicitly stop the previous rolling sound before starting the new one (plastic→wire, wire→plastic)
4. WHEN implementing direction-aware sounds THEN the system SHALL use `ActiveBall.vely` for direction detection
5. WHEN sound panning is needed THEN the system SHALL guard with AudioPan object property checks (prevents Fleep errors on standalone)
6. WHEN SSF (surround sound feedback) is used THEN the system SHALL handle stereo vs mono conversion
7. WHEN the table uses MP3 sound files THEN the system SHALL generate preloading code in `Table1_Init` to prevent first-play frame drops

**Priority:** Medium | **Complexity:** Low | **Dependencies:** None

---

## Requirement 8: Troubleshooting Skill

**User Story:** As a table author encountering bugs, I want Claude to diagnose common VPX issues and suggest proven fixes.

**Acceptance Criteria:**
1. WHEN the ball passes through walls or colliders THEN the system SHALL check: normals direction, collidable flag, duplicate collidable primitives, missing thickness for two-sided collision
2. WHEN the ball sinks through the playfield THEN the system SHALL suggest: under-playfield blocker wall (Z=-0.01), check for collidable primitives being used as resting surfaces (use VPX walls instead)
3. WHEN flippers feel wrong THEN the system SHALL check: duplicate Collide subs (silently overridden), flipper trigger sizing (27 VP standard), polarity curve era mismatch, ball mass ≠ 1, Global Physics Sets conflict
4. WHEN physics materials show zeroed values THEN the system SHALL suggest re-importing .vpp file
5. WHEN frame rate drops THEN the system SHALL check: timer interval 1 (change to -1), GetBalls in rolling sub (use gBOT), render probe count and roughness, high-poly collidable primitives, transparency overdraw
6. WHEN `cor.BallVel` errors occur THEN the system SHALL verify GameTimer exists and contains `Cor.Update`
7. WHEN FlexDMD fails THEN the system SHALL check: auto-detection via error handling (not filesystem), scene array initialization (not per-show creation), update timer stays enabled between games
8. WHEN NVRAM is corrupted THEN the system SHALL guide the user to clear the `.nv` file
9. WHEN z-fighting occurs THEN the system SHALL explain depth bias (counter-intuitive: top = negative, bottom = positive) and the 99.99% scale workaround
10. WHEN sounds play twice THEN the system SHALL check for missing `If Enabled Then` guard in SolCallback
11. WHEN standalone crashes on flipper hit THEN the system SHALL check for `gBOT` usage in `Cor.Update` and recommend replacing with `GetBalls` in that specific context (gBOT causes segfaults in standalone's Cor.Update — this is the cause of 95% of "ball hits flipper crashes" in standalone). NOTE: gBOT is correct everywhere ELSE in the script — only Cor.Update needs GetBalls
12. WHEN generating options menu code for VPX 10.8.1+ THEN the system SHALL set `DisableStaticPreRendering = True` once on menu entry and `False` once on exit (not per-option-change, due to 10.8.1 counter semantics — spamming True breaks it)

**Priority:** High | **Complexity:** Medium | **Dependencies:** All other skills (cross-cutting)

---

## Requirement 9: Toys & Mechanisms Skill

**User Story:** As a table designer implementing mechanical toys (diverters, magnets, motors, VUKs), I want Claude to generate correct scripting patterns for complex mechanisms.

**Acceptance Criteria:**
1. WHEN implementing a diverter THEN the system SHALL generate SolCallback with correct wall/kicker activation, position tracking, and sound
2. WHEN implementing a magnet THEN the system SHALL generate ball capture with timed release and force application
3. WHEN implementing a VUK (Vertical Up Kicker) THEN the system SHALL generate correct KickZ parameters (angle, speed, inclination, heightz) with timing
4. WHEN implementing a spinner THEN the system SHALL use opacity cross-fade between 0° and 180° primitive meshes in FrameTimer
5. WHEN implementing drop targets THEN the system SHALL generate Roth drop target animation system with DoDTAnim in FrameTimer, correct RotZ=0 facing drain
6. WHEN implementing a motor mechanism (carousel, goalie, etc.) THEN the system SHALL use cvpmMech with correct Sol1/Sol2 wiring and limit switch handling
7. WHEN a mechanism requires "creating" balls THEN the system SHALL explain that real machines don't create balls — use physical trough replacement and gBOT tracking
8. WHEN the table uses scripted stepper motor mechanisms THEN the system SHALL set `HandleMech=0` to prevent PinMAME interference (see also Req 4.10)

**Priority:** Medium | **Complexity:** High | **Dependencies:** Req 2 (game logic), Req 4 (PinMAME wiring)

---

## Requirement 10: Reference / Master Index Skill

**User Story:** As a user starting a new table project, I want a single entry point that helps me understand what skills are available and recommends a workflow.

**Acceptance Criteria:**
1. WHEN user says "I'm starting a new VPX table" THEN the system SHALL present the recommended build workflow: scope → scan → model → script → physics → lighting → sound → test → release
2. WHEN user asks "what can you help with?" THEN the system SHALL list all available skills with one-line descriptions
3. WHEN user describes a problem THEN the system SHALL route to the appropriate skill (troubleshooting for bugs, physics for flipper feel, lighting for GI issues, etc.)
4. WHEN generating any VBScript THEN the system SHALL always cross-reference conventions skill for naming, timers, and performance patterns
5. IF the user is working on a recreation (ROM-based) table THEN the system SHALL prioritize PinMAME wiring and game logic skills
6. IF the user is working on an original (non-ROM) table THEN the system SHALL prioritize game logic, Light State Controller, and FlexDMD patterns

**Priority:** High | **Complexity:** Low | **Dependencies:** All other skills

---

## Non-Functional Requirements

### Quality
- WHILE generating VBScript THEN the system SHALL produce code that follows VPW conventions (Req 6) without manual correction
- WHEN generated code references physics constants THEN values SHALL match the nFozzy reference exactly (not approximate)

### Accuracy
- WHEN the plugin references a VPX feature THEN it SHALL specify which version introduced it (e.g., "VPX 10.8.1 native tilt" vs "script-based tilt for 10.7", "VPX 10.8.1 DisableStaticPreRendering counter change")
- WHEN the plugin recommends a pattern THEN it SHALL cite the VPW wiki source (guide name + table name where learned)

### Scope
- The plugin SHALL NOT cover 3D art pipeline (Blender modeling, texture baking, lightmap generation) — that's visual work, not scripting
- The plugin SHALL NOT cover editor/Whitewood development — that's the `vpx-editor` skill's scope (description must be scoped accordingly)
- The plugin SHALL focus on VBScript patterns that control gameplay, physics, lighting, sound, and ROM integration

### Maintenance
- Each skill file SHALL include a `version` field and `last_updated` date
- WHEN VPW releases new techniques or VPX adds new features THEN the relevant skill SHALL be updated

---

## Success Criteria

- [ ] A novice designer can describe table rules in plain English and receive working VBScript
- [ ] Generated nFozzy physics configuration passes the "dotted ball" test (F11, no abnormal spin)
- [ ] Generated ROM wiring matches PinMAME switch/coil/lamp numbering for target game
- [ ] Generated lighting code produces correct GI fading, flasher decay, and insert behavior
- [ ] The plugin catches at least 80% of the 28+ documented troubleshooting patterns when a user describes symptoms
- [ ] All generated code works on VPX 10.8+ standalone without modification
- [ ] gBOT is used for gameplay ball tracking, GetBalls is used in Cor.Update (standalone-safe)
