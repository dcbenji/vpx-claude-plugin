---
name: vpx-dev-conventions
description: "VPX VBScript pinball conventions naming layers timer performance optimization standalone compatibility"
version: "1.0.0"
last_updated: "2026-03-24"
---

# VPX Conventions & Best Practices

## When This Skill Applies

Load when: generating or reviewing VBScript for VPX tables, setting up timers, optimizing performance, preparing for standalone compatibility, organizing table layers, or validating a table for release.

**This skill's rules apply to ALL generated VBScript.** Other skills (physics, game-logic, lighting) generate code — this skill governs HOW that code is written.

## Quick Reference: Naming Conventions

| Category | Convention | Example |
|----------|-----------|---------|
| Bakemap primitives | `bm_` prefix | `bm_RampL` |
| Lightmap primitives | `lm_` prefix | `lm_GI_PF` |
| Playfield elements | `pf_` prefix | `pf_Rails` |
| Flashers | `f` + `1` + solenoid# | `f109` (sol 9), `f125` (sol 25) |
| Insert lit state | `_on` suffix | `p32on` |
| Insert unlit state | `_off` suffix | `p32off` |
| Insert bulb light | `_bulb` suffix | `p32bulb` |
| Table name | Max 31 characters | VPX silently truncates longer names |
| Component names | Max 6 characters | Prevents truncation in toolkit lightmap names |
| Object names | No leading digits | `9_Rails_BM` causes runtime errors |

**Source:** VPW naming standards (Police Force, Godzilla, Earthshaker, Iron Man)

## Layer Organization

Standard 11-layer structure:

| Layers | Contents |
|--------|----------|
| 1-3 | Collidable objects (ramps, flippers, walls, switches, targets, posts, pegs) |
| 4-6 | Primitives and inserts |
| 7-9 | Lights |
| 10-11 | VR objects and playfield-sized flashers |
| Timers | Dedicated layer (survives Blender re-bakes) |

Keep collidable objects isolated — hidden objects on forgotten layers create hard-to-debug physics issues. Use temporary layers ("quarantine", "unknown") during development, clean up before release.

**Source:** Batman DE layer structure

## Timer Architecture

### Three-Tier System

**GameTimer (interval = 10ms)** — Physics-critical. Must contain `Cor.Update` or `cor.BallVel` errors occur.

```vbs
Sub GameTimer_Timer()
    Dim BOT: BOT = GetBalls   ' GetBalls here, NOT gBOT (standalone segfault)
    Cor.Update BOT
    ' Physics-dependent game logic here
End Sub
```

**FrameTimer (interval = -1)** — Visual updates tied to rendering. Use for: DoDTAnim, DoSTAnim, ball shadows, rolling sounds, insert lighting.

```vbs
Sub FrameTimer_Timer()
    DoDTAnim
    DoSTAnim
    RollingUpdate
End Sub
```

**LampTimer (fixed interval, 6-16ms)** — Light fading and Lampz updates. Must be fixed interval (NOT -1). Required for Light State Controller to function.

```vbs
Sub LampTimer_Timer()
    Lampz.UpdateLamps
    LightController.Update
End Sub
```

### Critical Timer Rules

**Set interval BEFORE enabling:**
```vbs
' CORRECT:
MyTimer.Interval = 20
MyTimer.TimerEnabled = True

' WRONG — first tick fires at default interval:
MyTimer.TimerEnabled = True
MyTimer.Interval = 20
```

**Never use interval 1.** VPX processes all timers once per frame — a 1ms timer fires 10+ times per frame update. Changing from 1ms to -1 doubled FPS on Goldeneye (80 to 150+).

**Never use `vpmTimer.Add`.** It is broken and causes cryptic `VBSE_OLE_NO_PROP_OR_METHOD` errors with no line reference.

### Timer Gotchas

**Interval -1 extra execution bug:** Timers with interval -1 execute ONE additional time after `TimerEnabled = False`. The disable takes effect on the next screen refresh, not immediately. Worse in VR (larger time step per frame). Account for this in position/animation code.

**Frame-rate-dependent animations:** Interval -1 produces different speeds on different systems. Fix: multiply animation increments by a frame-rate scaling factor.

**Catch-up spiral:** Too many separate timers hurt performance. Consolidate into one frametimer and call all update functions from there.

**Source:** Goldeneye, Fish Tales, Space Station, Austin Powers

## Performance Checklist

| Issue | Check | Fix |
|-------|-------|-----|
| Interval-1 timers | Any timer with interval = 1? | Change to -1 (per-frame) |
| GetBalls calls | GetBalls in rolling/update subs? | Use `gBOT` array (but keep GetBalls in Cor.Update) |
| Render probes | Roughness > 0? | Set roughness = 0 on all probes (~40% FPS gain) |
| Collidable polys | High-poly primitives set collidable? | Reduce poly count or use simplified collision meshes |
| Transparency | Many non-opaque elements? | Limit transparent objects; each with refraction = separate render pass |
| debug.print | In hot code paths (timers, rolling)? | Remove — I/O overhead causes visible frame drops |

**Source:** Fish Tales, Police Force, Bad Cats, Guns N' Roses

## Standalone Compatibility

### gBOT vs GetBalls

Use `gBOT` (global ball array) for all gameplay code. **Exception:** `Cor.Update` must use `GetBalls` — using `gBOT` in Cor.Update causes segfaults on standalone (the #1 cause of "crashes on flipper hit" in standalone).

```vbs
' In Cor.Update — ALWAYS use GetBalls here:
Sub GameTimer_Timer()
    Dim BOT: BOT = GetBalls
    Cor.Update BOT
End Sub

' Everywhere else — use gBOT:
Sub RollingUpdate()
    Dim b : b = gBOT
    ' ... ball tracking logic
End Sub
```

### FlexDMD Auto-Detection

Detect FlexDMD via error handling, NOT filesystem checks. Filesystem paths fail on standalone (Android/Linux/Mac). FlexDMD is built into standalone VPX.

```vbs
On Error Resume Next
Set FlexDMD = CreateObject("FlexDMD.FlexDMD")
If Err Then UseFlexDMD = 0 Else UseFlexDMD = 1
On Error Goto 0
```

### Const Declaration Order

All `Const` declarations must appear BEFORE any code that references them. Standalone's VBScript engine is stricter than Windows VPX about declaration order — referencing a Const before its definition crashes.

### Preserve System Constants

Never remove these constants even if they appear unused — system VBS files require them:
```vbs
Const cSingleLFlip = 0
Const cSingleRFlip = 0
```

Removing them silently breaks flipper/solenoid management.

**Source:** Road Show, Rollercoaster Tycoon, Medieval Madness

## VBScript Gotchas

### Not 1 Returns -2

VBScript `Not` on an integer performs bitwise NOT, not logical NOT. `Not 1` = `-2` (truthy), not `0`.

```vbs
' WRONG — fails when cabinetmode = 1:
If Not cabinetmode Then ...

' CORRECT:
If cabinetmode = 0 Then ...
```

Always use `= 0` comparisons instead of `Not` for integer flags (service menu values, config flags).

### Duplicate Sub Names Silently Override

VBScript does not error on duplicate sub names. The last definition wins. This affects ANY sub — not just flipper Collide subs. Physics functions like CheckDampen and CheckLiveCatch vanish if a duplicate sub overrides them.

### debug.print in Hot Paths

`debug.print` writes to both console and log file. In timers or rolling subs, the I/O overhead causes visible frame drops. Remove all before release.

### Pure Black (#000000)

Never use pure black in materials or textures. It causes divide-by-zero in renderers and broken lighting. Use `#111111` (RGB 17,17,17) as the minimum dark value.

**Source:** Game of Thrones, Austin Powers, Johnny Mnemonic

## Physics Defaults

| Setting | Value | Notes |
|---------|-------|-------|
| Playfield friction | 0.15-0.25 | Default 0.02 is far too low |
| Minimum friction | 0.1 | Never use 0 on any surface |
| BallSize | 50 | Not legacy 25 (wrong physics) |
| Ball mass | 1 | Not 1.3 even for widebody (breaks flipper tuning) |
| Difficulty | 50-56 | NOT default 20 — all tuning at wrong difficulty is wrong |

**Source:** Iron Man, Hook, Judge Dredd

## Production Checklist

Before release, verify:

- [ ] All `debug.print` statements removed from hot paths
- [ ] BallSize = 50, ball mass = 1, difficulty 50-56
- [ ] Import .vpp physics materials (F11 dotted ball test to verify)
- [ ] No duplicate Sub/Function names (check flipper Collide subs especially)
- [ ] `Const` declarations before all references
- [ ] `cSingleLFlip=0` and `cSingleRFlip=0` present
- [ ] No pure black (#000000) in materials
- [ ] No zero-friction surfaces (minimum 0.1)
- [ ] Timer intervals appropriate (no 1ms timers for visual updates)
- [ ] All SolCallback subs have `If Enabled Then` guard
- [ ] gBOT used everywhere except Cor.Update (which uses GetBalls)
- [ ] VRRoom = 0 for desktop releases
- [ ] VSync override = -1 in table User properties
- [ ] Flipper trigger margins at 27 VP units
- [ ] Playfield physical sounds are mono
- [ ] Tilt functionality implemented

**Source:** VPW release checklist (across all builds)
