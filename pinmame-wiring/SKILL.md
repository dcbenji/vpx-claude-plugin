---
name: vpx-dev-pinmame-wiring
description: VPX VBScript pinball ROM PinMAME switch coil lamp solenoid SolCallback trough drain FastFlips wiring GI UseVPMModSol expansion daughter card HandleMech game definition
version: "1.0.0"
last_updated: "2026-03-24"
---

# PinMAME Wiring

## When This Skill Applies

Load when wiring a recreation (ROM-based) VPX table: mapping solenoids, switches, lamps, configuring trough, setting up FastFlips, GI dimming, or looking up game definitions. For original (non-ROM) tables, see the game-logic skill instead.

## Quick Reference

| Setting | Value | Notes |
|---------|-------|-------|
| UseVPMModSol | 0 = off, 1 = modulated, 2 = fast flips | See era recommendations below |
| UseLamps | 1 | Standard for ROM lamp mapping |
| HandleMech | 0 | Required for scripted stepper motors |
| FastFlips target | < 2ms | Second value in debug overlay |
| +100 offset | Flashers sharing lamp numbers | `SolCallback(17) = "SetLamp 117,"` |

## Patterns

### SolCallback Wiring

**When to use:** Every ROM-controlled solenoid (coils, kickers, diverters, flashers).

**Template:**
```vbs
' Map solenoid numbers to handler subs
SolCallback(1) = "SolFlipper_L"
SolCallback(2) = "SolFlipper_R"
SolCallback(15) = "SolScoop"

Sub SolScoop(enabled)
    If enabled Then
        PlaySoundAtVol "Scoop_Up", Scoop1, VolumeDial
        Scoop1.kick angle, speed
    End If
End Sub
```

**Gotchas:**
- **CRITICAL: `If Enabled Then` guard is mandatory.** SolCallback fires with BOTH True and False. Without the guard, sounds play twice and mechanisms double-fire. (Source: Medieval Madness)
- Flashers sharing lamp numbers use +100 offset: `SolCallback(17) = "SetLamp 117,"` with `Lampz.MassAssign(117)=F17` (Source: Judge Dredd)
- ROM tables using Lampz need a `SetLamp` sub (not in Example Table by default):
```vbs
Sub SetLamp(aNr, aOn)
    Lampz.state(aNr) = abs(aOn)
End Sub
```

### Switch Constants

**When to use:** Mapping VPX triggers/targets to ROM switch inputs.

Switch numbers use VPinMAME numbering (not physical machine matrix). Look up constants in the framework VBS files:
- **WPC:** `core.vbs` — standard switch numbering
- **S11/DE/GTS3:** `S11.VBS`, `DE.VBS`, `GTS3.VBS` — era-specific mappings

Switch constants are typically defined as `Const` at the top of the table script. Standalone requires all `Const` declarations BEFORE any code that references them.

### Trough Setup

**When to use:** Every ROM table needs a physical trough.

**Template (WPC-style):**
```vbs
' Physical trough with kicker objects — never destroy balls
Dim gBOT  ' Global ball-on-table array

Sub Table1_Init
    ' Create all balls at initialization
    Dim i
    For i = 1 To tnob
        Set gBOT(i) = Trough(i).CreateSizedBallWithMass(BallSize, BallMass)
    Next
End Sub

' Drain kicker fires ball into trough
Sub SolDrain(enabled)
    If enabled Then
        DrainKicker.kick angle, speed
    End If
End Sub
```

**Gotchas:**
- Use `CreateSizedBallWithMass` (not `CreateBall`) — ensures correct BallSize 50 and mass 1
- Gottlieb System 3 uses only 2 switches (outhole + ball release), NOT WPC-style per-ball position switches. Don't force WPC trough arrays on GTS3. (Source: Stargate)
- Pre-load trough balls at game start to eliminate ROM boot sequence delay. (Source: Star Trek TNG)
- `gBOT` is correct for gameplay code. In `Cor.Update` specifically, use `GetBalls` instead (gBOT causes standalone segfaults). (Source: Road Show)

### UseVPMModSol

**When to use:** Configuring solenoid modulation mode in table init.

| Mode | Behavior | Use When |
|------|----------|----------|
| 0 | Off — solenoids are binary on/off | Legacy tables, simple wiring |
| 1 | Modulated — ROM sends PWM values (0-255) | GI dimming, flasher intensity control |
| 2 | Fast flips — solenoid response bypasses ROM latency | Any table needing responsive flippers |

**Era recommendations:**
- **WPC (90s):** `UseVPMModSol = 2` + `UseSolenoids = 2` for fast flips
- **SAM (Stern 2006+):** `UseVPMModSol = 2` but also requires `InitVpmFFlipsSAM` (see FastFlips)
- **DE/Sega:** `UseVPMModSol = 2` with custom solenoid mapping
- **GTS3/S11:** `UseVPMModSol = 1` (fast flips less critical on older hardware)

### FastFlips

**When to use:** Reducing flipper response latency on ROM tables.

**SAM tables (Stern 2006+):**
```vbs
Sub Table1_Init
    InitVpmFFlipsSAM
End Sub
```
`UseSolenoids = 2` alone does NOT enable fast flips on SAM ROMs. You must call `InitVpmFFlipsSAM`. (Source: Tron, Iron Man)

**Data East solenoid mapping:**
```vbs
' DE uses different solenoid numbers than Williams
vpmFlips.FlipperSolNumber(2) = 47   ' DE Tommy uses sol 47
' Williams typically uses solenoid 22
```

**ROM bypass fallback:** When FastFlips doesn't work (some newer SAM ROM versions), bypass the ROM entirely — fire the flipper directly from the key press event while still sending the command to the ROM. Verify no other game actions are linked to flipper button presses first. (Source: Tron)

**Verification:** FastFlips timing should show under 2ms in the second value of the debug overlay.

### GI Dimming (SolModCallback)

**When to use:** ROM-controlled GI strings that dim rather than just on/off.

```vbs
' SolModCallback receives 0-255 intensity values (not just True/False)
SolModCallback(xx) = "SolGI_String1"

Sub SolGI_String1(level)
    ' level = 0 (off) to 255 (full brightness)
    Dim ratio: ratio = level / 255
    ' Apply to BlendDisableLighting on GI-affected primitives
    GI_Prim1.BlendDisableLighting = ratio * MaxGI
End Sub
```

Use `SolModCallback` (not `SolCallback`) for GI — it provides the PWM level for smooth dimming. `SolCallback` only gives binary on/off. See the lighting skill for full GI string management.

### Expansion Hardware

**When to use:** Tables with daughter cards, A/B relays, or extended I/O.

PinMAME appends expansion hardware I/O at the END of the standard numbering scheme. Daughter cards (auxiliary driver boards), A/B relay matrices, and extended lamp/solenoid boards add their outputs starting after the last standard solenoid/lamp number.

Check the game definition file for the specific table — expansion numbering varies by hardware generation and board configuration.

### HandleMech

**When to use:** Tables with scripted stepper motors (drawbridges, rotating elements, carousels).

```vbs
HandleMech = 0   ' Prevent PinMAME mech handling interference
```

Without this, PinMAME's internal mechanism handling conflicts with your scripted motor control. Always set `HandleMech = 0` when the table script manages the mechanism directly. (Source: Medieval Madness)

### Game Definition Lookup

**When to use:** Identifying switch/coil/lamp numbers for a specific ROM.

1. Check for local clone: `vpinball/pinmame-game-defs` in workspace or filesystem
2. If present: look up the game's `.json` file for complete switch, coil, and lamp mappings
3. If absent: guide user to clone it:
```bash
git clone https://github.com/vpinball/pinmame-game-defs.git
```
4. Fallback: use the debug solenoid output pattern to discover active solenoids at runtime:
```vbs
dim chgsol, xxx
chgsol = controller.changedsolenoids
If Not IsEmpty(chgsol) Then
    For xxx = 0 To UBound(chgsol)
        debug.print chgsol(xxx, 0) &" -> "& chgsol(xxx, 1)
    next
End If
```
Call this in a timer. Warning: may "eat" events from SolCallback — use only for discovery. (Source: VPW Resources)

## Anti-Patterns

### Missing If Enabled Guard
**What it looks like:** `Sub SolX(enabled)` with no conditional check
**Why it's wrong:** Solenoid sounds play twice, mechanisms double-fire
**Fix:** Always wrap action code in `If enabled Then`

### Using cvpmBallStack Instead of Physical Trough
**What it looks like:** `cvpmBallStack` destroying and recreating balls
**Why it's wrong:** Ball destruction/recreation sometimes fails; shadow tracking breaks (ball IDs change)
**Fix:** Use physical trough with kicker objects and `CreateSizedBallWithMass`. Never destroy balls.

### Wrong FastFlips for Era
**What it looks like:** `UseSolenoids = 2` on SAM table without `InitVpmFFlipsSAM`
**Why it's wrong:** Fast flips silently don't activate on SAM ROMs
**Fix:** Call `InitVpmFFlipsSAM` in `Table1_Init` for SAM tables; use correct solenoid mapping for DE
