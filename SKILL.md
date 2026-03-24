---
name: vpx-dev
description: VPX VBScript pinball table development scripting game rules physics lighting sound ROM PinMAME
version: "1.1.0"
last_updated: "2026-03-24"
---

# VPX Table Development — Master Skill

## When This Skill Applies

Load when working on VPX table development: writing VBScript, configuring physics, wiring ROMs, setting up lighting/sound, or troubleshooting table issues. This is NOT for vpx-editor codebase development (that's the vpx-editor skill).

## Build Workflow

Follow this 9-step process for any new table:

1. **Scope** — Recreation (ROM-based) or original? Era (70s/80s/90s/modern)? Target VPX version?
2. **Scan** — Gather reference materials: rule cards, manual, playfield photos. For recreations: identify ROM, switch/coil/lamp mappings
3. **Model** — Import/build 3D playfield, primitives, plastics. Organize into standard 11 layers
4. **Script** — Game logic: modes, multiball, scoring, tilt, attract. Wire ROM if recreation
5. **Physics** — nFozzy setup: flipper tuning for era, rubber separation, target bouncers, materials. Set difficulty 50-56 FIRST
6. **Lighting** — GI strings, flasher decay, insert lighting, Light State Controller
7. **Sound** — Fleep collections, ball rolling, ramp transitions, MP3 preloading
8. **Test** — F11 dotted ball test, standalone compatibility check, production checklist
9. **Release** — Remove debug.print, verify BallSize/mass/difficulty, import .vpp, check duplicate subs

## Available Skills

| Skill | Purpose |
|-------|---------|
| **object-reference** | Every VPX object type: events, properties, methods, usage examples |
| **glf** | Game Logic Framework for original tables: modes, shots, multiball, scoring, events |
| **conventions** | Naming, layers, timers, performance, standalone compatibility |
| **nfozzy-physics** | Flipper tuning by era, rubber separation, targets, materials |
| **game-logic** | Modes, multiball, scoring, ball locks, tilt, attract, wizard mode |
| **pinmame-wiring** | ROM switches, coils, lamps, SolCallback, FastFlips, trough setup |
| **lighting** | GI strings, flasher decay, inserts, Light State Controller |
| **sound** | Fleep setup, ball rolling, ramp transitions, SSF |
| **toys-mechs** | Diverters, magnets, VUKs, spinners, drop targets, motors |
| **troubleshooting** | 28+ symptom-to-fix patterns for common VPX issues |

## Critical Globals

Set these BEFORE any other work. Wrong values invalidate all subsequent tuning.

| Setting | Required Value | Why |
|---------|---------------|-----|
| Difficulty | 50-56 | VPX default 20 is wrong. All physics tuning assumes 50-56 |
| Ball mass | 1 | Any other value breaks nFozzy flipper physics |
| BallSize | 50 | Legacy value 25 produces wrong physics |
| Min friction | 0.1 | Zero friction on any surface causes unpredictable behavior |
| No pure black | #111111 minimum | #000000 causes divide-by-zero in renderers |

## VPX Version Features

| Version | Feature | Notes |
|---------|---------|-------|
| 10.8 | Native Incandescent fader | 40ms up, 120ms down. Preferred over Lampz FadeSpeed |
| 10.8.1 | Native tilt support | Script-based tilt still works as fallback |
| 10.8.1 | DisableStaticPreRendering counter | Set True ONCE on menu entry, False ONCE on exit. Do NOT spam per-option-change |

## Recreation vs Original Routing

**Recreation (ROM-based):** Prioritize in this order:
1. **pinmame-wiring** — Get ROM switches, coils, lamps mapped correctly first
2. **game-logic** — Wire ROM events to game modes, scoring, multiball
3. **nfozzy-physics** — Tune flipper/rubber physics to match the real machine's era
4. **lighting** — GI dimming via ROM, flasher decay, insert maps

**Original (non-ROM):** Prioritize in this order:
1. **glf** — Game Logic Framework for modes, multiball, scoring, shots (recommended for new originals)
2. **game-logic** — Alternative: hand-coded state machines if GLF is not used
3. **nfozzy-physics** — Pick an era feel, tune flippers and rubbers
4. **lighting** — Light State Controller for insert management, FlexDMD for display
5. **sound** — Fleep setup, custom sound design

## Quick Start — New Original Table from Scratch

When a novice asks to start a new table, generate this foundation. Everything gets nFozzy physics by default.

### Step 1: Table Properties (set in VPX Table > Properties)
```
BallSize = 50
BallMass = 1
Difficulty = 56
Playfield Friction = 0.2
Playfield Elasticity = 0.25
Playfield Scatter = 2
```

### Step 2: Required VPX Objects on the Table
Before any scripting, the table must have these VPX objects created in the editor:
- **Flippers:** `LeftFlipper`, `RightFlipper` — with nFozzy trigger objects `LFTrigger`, `RFTrigger` (27 VP units from flipper, hit height 150)
- **Drain kicker:** `BallRelease` — at bottom center, aimed up (-90° or equivalent)
- **Trough kickers:** `Trough1`, `Trough2`, `Trough3` (minimum 3 for standard games)
- **GameTimer:** Timer object, interval = 10, enabled = True
- **FrameTimer:** Timer object, interval = -1, enabled = True
- **LampTimer:** Timer object, interval = 10, enabled = True (needed for Light Controller)

### Step 3: Minimal Table Script (nFozzy + Fleep + Scoring)

Paste the VPW nFozzy code chunk (from VPW Example Table or the `nfozzy_*.vbs` files). Then add this game structure:

```vbs
' ============ CRITICAL CONSTANTS — DO NOT REMOVE ============
Const cSingleLFlip = 0   ' Required by system VBS
Const cSingleRFlip = 0   ' Required by system VBS

' ============ GAME VARIABLES ============
Dim Score : Score = 0
Dim BallsRemaining : BallsRemaining = 3
Dim gBOT : gBOT = Array()    ' Global ball tracking (never destroy balls)

' ============ TABLE INIT ============
Sub Table1_Init
    ' Set difficulty FIRST
    Table1.Difficulty = 56

    ' Create balls in trough (physical trough — never destroy)
    Dim i
    For i = 0 To 2
        Set gBOT(i) = Trough(i).CreateSizedBallWithMass(BallSize, 1)
    Next

    ' Preload MP3 sounds to prevent first-play stutter
    PlaySound "music_main", 0, 0.001

    ' Initialize FlexDMD (auto-detect, no filesystem check)
    On Error Resume Next
    Set FlexDMD = CreateObject("FlexDMD.FlexDMD")
    On Error Goto 0
End Sub

' ============ TIMERS ============
Sub GameTimer_Timer()
    Dim BOT: BOT = GetBalls   ' GetBalls HERE — NOT gBOT (standalone crash)
    Cor.Update BOT
End Sub

Sub FrameTimer_Timer()
    DoDTAnim        ' Drop target animation
    DoSTAnim        ' Standup target animation
    RollingUpdate   ' Ball rolling sounds (Fleep)
End Sub

' ============ FLIPPER INPUT ============
Sub Table1_KeyDown(ByVal keycode)
    If keycode = LeftFlipperKey Then
        LeftFlipper.RotateToEnd
        LF.Fire    ' nFozzy flipper fire
    End If
    If keycode = RightFlipperKey Then
        RightFlipper.RotateToEnd
        RF.Fire    ' nFozzy flipper fire
    End If
    If keycode = PlungerKey Then PlungerAction
End Sub

Sub Table1_KeyUp(ByVal keycode)
    If keycode = LeftFlipperKey Then
        LeftFlipper.RotateToStart
        LF.UnFire
    End If
    If keycode = RightFlipperKey Then
        RightFlipper.RotateToStart
        RF.UnFire
    End If
End Sub

' ============ SCORING — ADD TARGETS HERE ============
Sub Target1_Hit()
    Score = Score + 1000
    ' Update display: DMDScore.Text = FormatNumber(Score, 0)
End Sub

Sub Bumper1_Hit()
    Score = Score + 100
End Sub

' ============ DRAIN / BALL MANAGEMENT ============
Sub Drain_Hit()
    RF.PolarityCorrect Activeball   ' nFozzy drain fix
    LF.PolarityCorrect Activeball
    BallsRemaining = BallsRemaining - 1
    If BallsRemaining > 0 Then
        BallRelease.Kick 90, 5   ' Launch next ball
    Else
        ' Game over
    End If
End Sub
```

### Step 4: Adding More Scoring
For each new target/bumper/ramp the user draws, add a `_Hit` sub following the pattern above. Claude should generate these automatically when the user describes what they want: "I added a drop target bank of 3 — when all are down, score 5000 bonus."

### Important: What NOT to Do
- Do NOT start from the VPW Example Table (it's a reference, not a template — too complex for beginners)
- Do NOT skip nFozzy — even the simplest table should have proper flipper physics
- Do NOT use `GetBalls` anywhere except `Cor.Update` in GameTimer
- Do NOT use `vpmTimer.Add` — it's broken

## Cross-Reference Directive

When generating ANY VBScript, always apply the rules from the **conventions** skill:
- Naming prefixes (`bm_`, `lm_`, `pf_`, flasher naming, insert suffixes)
- Timer architecture (three-tier, set interval BEFORE enabling, never use `vpmTimer.Add`)
- Standalone compatibility (`gBOT` for gameplay, `GetBalls` in `Cor.Update` ONLY, Const order, `cSingleLFlip`/`cSingleRFlip` preservation)
- Performance (no `debug.print` in hot paths, no `GetBalls` in rolling subs, no interval-1 timers)
- VBScript gotchas (`Not 1` = `-2`, duplicate Sub names silently override)

This is not optional. Every VBScript output must comply.
