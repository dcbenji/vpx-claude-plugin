---
name: vpx-dev-sound
description: "VPX VBScript pinball sound Fleep rolling SSF audio music MP3 ball rolling ramp mechanical sound effects panning stereo mono preload"
version: "1.0.0"
last_updated: "2026-03-24"
---

# VPX Sound Implementation

## When This Skill Applies

Use when configuring Fleep sound packages, ball rolling sounds, ramp transition audio, direction-aware sound effects, SSF setup, or MP3 preloading in a VPX table.

## Quick Reference

| Collection | Purpose | Notes |
|-----------|---------|-------|
| `dPosts` | Rubber-wrapped post sounds | nFozzy separated posts |
| `dSleeves` | Rubber sleeve sounds | nFozzy separated sleeves |
| `GatesWire` | Wire gate sounds | Must be "GatesWire" not "Gates" |
| `Apron` | Apron collision sounds | Hit events must be enabled |
| `Metals` | Metal contact sounds | Hit threshold = 2 for guide rails |
| `Woods` | Wood contact sounds | |
| `Rubbers` | Rubber bounce sounds | Separate from dPost/dSleeve |
| `Rollovers` | Rollover switch sounds | |
| `Walls` | Wall collision sounds | |

## Patterns

### Fleep Collection Setup

**When to use:** Every table using Fleep sound packages.

Every collection above must have **"Fire events for this collection"** enabled in VPX. Without this, collection hit events never fire and sounds are silent.

**Commonly missing items:**
- `Rubber_4` sound (duplicate `Rubber_3` and rename)
- Plunger pull sound effects
- `Metals` and `Rollovers` collections
- `Apron` collection with hit events
- `min()` function (defined under the ZMAT section of Fleep example table)

**Cartridge selection:** Match by era and system, not just manufacturer. Compare by ear -- even matching assembly part numbers doesn't guarantee matching sounds (cabinet resonance, replacement coils).

**Flipper near-ball triggers:** Newer Fleep packs (TNA, Whirlwind+) require `TriggerBallNearLF` and `TriggerBallNearRF` trigger objects near flippers to distinguish dampened vs full-stroke flipper sounds.

**Source:** Fleep Sounds Guide, VPW Example Table

### Ball Rolling Sounds

**When to use:** Playfield and ramp rolling audio.

**Playfield rolling** uses a height threshold to separate playfield from ramp:
```vbs
If BallVel(BOT(b)) > 1 AND BOT(b).z < 30 Then
    ' playfield rolling sound
End If
```
Adjust `30` to raise/lower the height cutoff.

**Ramp rolling (ZRRL)** requires separate ZRRL sound effects plus:
- A timer for the rolling sound loop
- Triggers on ramps for hit events
- Height-based volume: half volume with +50000 pitch when `BOT(b).z > 30`

**Gotchas:**
- Ball rolling sound persists after multiball ball destruction -- add cleanup code to stop sounds when balls are removed
- Disable collision sounds below Z=0 (trough/destroyed balls cause phantom sounds)
- `tnob` (total number of balls) must match ball shadow count or shadow update breaks

**Source:** Fleep Sounds Guide, Austin Powers

### Ramp Sound Transitions

**When to use:** Tables with both plastic and wire ramps.

**Always stop the previous rolling sound before starting the new one.** Failing to do this causes overlapping sounds (e.g., plastic rolling continues while wire rolling starts).

Pattern: plastic ramp entry stops wire sound, wire ramp entry stops plastic sound. Use triggers at ramp entry/exit points.

**Source:** Fleep Sounds Guide

### Direction Detection

**When to use:** Entry/exit sounds that depend on ball travel direction.

```vbs
Sub Ramp1_Enter_Hit
    If activeball.vely < 0 Then WireRampOn False
End Sub
Sub Ramp1_Enter_UnHit
    If activeball.vely > 0 Then WireRampOff
End Sub
```

- **Negative `vely`** = ball moving up (toward backbox)
- **Positive `vely`** = ball moving down (toward drain)

**Source:** Fleep Sounds Guide

### Standalone AudioPan Guard

**When to use:** Always, in any table using Fleep's AudioPan function.

Fleep's `AudioPan` expects an object with an `x` property. Walls and some objects lack positional properties, causing runtime crashes on standalone. Always pass a physical VPX object (primitive, light, or positioned wall) with positional properties.

Before calling AudioPan, verify the object has the expected property. If integrating into a standalone-targeted table, test all sound-emitting objects for `x` property availability.

**Gotchas:**
- Sound panning pan law: Left=100%, Middle=200%, Right=100% (VPX engine limitation, not scriptable)
- Passing a timer or collection to AudioPan crashes silently on some platforms

**Source:** Fish Tales, Fleep Sounds Guide

### SSF Stereo vs Mono

**When to use:** Tables with surround sound feedback or positional audio.

VPX's positional audio system expects **mono WAV** inputs. Stereo files bypass pan calculations and produce incorrect spatial audio. VPX 10.8.1 reports validation warnings for stereo WAV files.

**Rule:** Convert all WAV sound samples to mono before importing.

**Source:** Ghostbusters, Stargate

### MP3 Preloading

**When to use:** Any table using MP3 sound files (music, callouts).

First-time MP3 playback causes frame drops as the audio subsystem decodes. Pre-cache in `Table1_Init`:

```vbs
Sub Table1_Init
    ' ... other init code ...
    PlaySound "song1", 0, 0.001
    PlaySound "song2", 0, 0.001
    PlaySound "callout1", 0, 0.001
End Sub
```

Volume `0.001` is inaudible but forces the decode. Do this for every MP3 used during gameplay.

**Source:** Goonies

## Anti-Patterns

### Missing `If Enabled Then` in Sound SolCallbacks
**What it looks like:** Solenoid sound plays on both activation AND deactivation.
**Why it's wrong:** SolCallback fires with both `True` and `False`. Without the guard, every solenoid sound plays twice.
**Fix:** Always wrap with `If Enabled Then`.

### VolumeDial Has No Effect
**What it looks like:** Adjusting VolumeDial doesn't change mechanical sound volume.
**Why it's wrong:** Fleep code bug -- some sounds exceed 1.0 before VolumeDial scaling, saturating at PlaySound's 1.0 cap.
**Fix:** Add `min()` function (from ZMAT section of Fleep example table) to pre-clamp volume to 1.0 before VolumeDial scaling.

### Multiple Sound File Amplitudes
**What it looks like:** Sound files with `_amp9`, `_amp6` suffix variants.
**Why it's wrong:** Wastes import slots. VPX PlaySound volume caps at 1.0 (cannot amplify).
**Fix:** Keep only the loudest version, use PlaySound volume parameter to reduce. Remove `_amp` suffixes. Ensure `StopSound` calls match actual filenames.
