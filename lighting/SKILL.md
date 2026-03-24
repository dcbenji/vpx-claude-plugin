---
name: vpx-dev-lighting
description: VPX VBScript pinball light GI flasher insert lamp fade lighting BlendDisableLighting Lampz Light State Controller PWM incandescent room brightness blink dim combine wash boost cross-dimming
version: "1.0.0"
last_updated: "2026-03-24"
---

# VPX Lighting

## When This Skill Applies

Load when configuring GI strings, flasher effects, insert lighting, light fading, Light State Controller, room brightness, or PWM integration in a VPX table.

## Quick Reference

| Setting | Value |
|---------|-------|
| Incandescent fade up | 40ms (native 10.8+) or 1/40 (Lampz) |
| Incandescent fade down | 120ms (native 10.8+) or 1/120 (Lampz) |
| LED fade up | 1/2 (Lampz) |
| LED fade down | 1/8 (Lampz) |
| GI color temperature | 2700K (AGX/Tony McMapFace) or 3000-3200K (Filmic) |
| Flasher decay formula | `FlashLevel = FlashLevel * 0.93 - 0.01` then `flashx3 = FlashLevel^3` |
| Minimum dark color | #111111 (never #000000) |
| Insert bloom size | 1/3 of what feels natural on first placement |
| BDL range | 0+ (never negative — breaks rendering) |
| GI string numbering | String 1 = `gi0lvl`, String 2 = `gi1lvl` (off-by-one) |

---

## Patterns

### GI String Management

**When to use:** Any table with multiple GI circuits.

**Identification methodology:**
1. Wire color tracing in under-playfield photos
2. Video frame-stepping (`,`/`.` in YouTube) during attract mode
3. ROM built-in GI test menu on real machine
4. Cross-reference multiple machines (account for blown bulbs)

**Per-string dimming:** Each GI string gets its own level variable (`gi0lvl`, `gi1lvl`, etc.). Control via `SolModCallback` for ROM tables or direct scripting for originals.

**BlendDisableLighting for GI elements:** Tie BDL to GI level so plastics, ramps, and parts respond to GI state:
```vbs
lramp.blenddisablelighting = 0.01 * LM_giTop_Parts.opacity
' Or with minimum visibility:
Primitive_MainRamp.blenddisablelighting = gilevel * 0.12 + 0.03
```

**Cross-dimming boost:** When two GI strings of different colors are normally both on, they must be tuned dim enough to not blow out when combined. Create a boost light collection:
```vbs
' Red + white GI example (Black Rose pattern)
IntensityScale = (1 - gi2lvl) * gi1lvl
```
Boost lights operate outside ModLampz, controlled directly via `IntensityScale`.

**Platform differences:**
- SAM (Stern): single on/off relay, not dimmable zones
- Data East: some tables use inverted solenoid logic (high = GI off)
- SAM lamp matrix GI: driven by lamp matrix indices (130, 132, 134, 136)
- System 11: "GI Blink" solenoid dims GI when activated (inverted)

**Gotchas:**
- String numbering is off-by-one: hardware String 1 = software `gi0lvl`
- Render GI as pure white in Blender, apply color in VPX script
- Ball brightness should track GI state (dim ball when GI is off)

**Source:** VPW gi-and-flashers guide (Black Rose, Johnny Mnemonic, Iron Man)

---

### Flasher Decay

**When to use:** Every table with flasher solenoids.

**Template:**
```vbs
' Each flasher gets its own SolCallback + timer
Sub SolFlash1(enabled)
    If enabled Then
        FlashLevel1 = 1
        FlashTimer1.Enabled = True
    End If
End Sub

Sub FlashTimer1_Timer()
    FlashLevel1 = FlashLevel1 * 0.93 - 0.01
    Dim flashx3 : flashx3 = FlashLevel1 * FlashLevel1 * FlashLevel1
    Flasher1.opacity = 110 * flashx3
    Light1.IntensityScale = 3 * flashx3
    If FlashLevel1 < 0 Then FlashTimer1.Enabled = False
End Sub
```

**Per-element variation:** Each flasher gets slightly different fade speed, intensity, and RGB shift during decay (shift toward amber as intensity drops = incandescent cooling). Prevents the "Christmas tree" uniform look.

**GI-aware opacity:** Scale max flasher brightness by GI level:
```vbs
Flasher.opacity = (100 - 30*gilvl) * ObjLevel(nr)
```
When GI ON, max flasher opacity = 70; when OFF, max = 100. Prevents wash-out.

**SolCallback vs SolModCallback:** SolCallback = binary 0/1. SolModCallback = modulated 0-255 for PWM solenoids. Even with modulation, use per-flasher timers with exponential decay rather than driving brightness directly from solenoid values (ROM output is erratic).

**Gotchas:**
- Fadedown too slow causes stacking — rapid fire events compound rendering load
- Flasher texture preloading: briefly display each texture at init (invisible opacity) to prevent first-fire stutter
- Flasher naming: `f` + `1` + solenoid number (e.g., sol 9 = `f109`, sol 25 = `f125`)

**Source:** VPW gi-and-flashers guide (Congo, Metallica, Black Rose)

---

### Insert Lighting

**When to use:** Setting up playfield insert lamps.

**Toolkit standard (VPX 10.8+):**
1. Each insert needs two lights in Blender (bottom glow + top spread) but only one VPX light
2. Set insert collection to "Split" for individually addressable lightmaps
3. Bake inserts into shallow cups/trays (flat inserts look wrong in VR)
4. Position layer separator between playfield and UnderPF inserts
5. VPX Lightmap property on primitives auto-controls opacity from linked light — no script needed

**Pre-toolkit (Flupper 3D inserts) — ON/OFF/BULB naming:**
- **OFF layer:** Depth bias 1000, DL=1, Z=0. Visible when unlit. Responds to GI.
- **ON layer:** Depth bias 1100, DL=0, Z=0. Lit state. Controlled via BDL from script.
- **BULB layer:** Depth bias 1200, DL=1, DLFromBelow=1, Z=-3. VPX light for ball reflection only.

**Color rules:** Never use pure RGB (e.g., 255,0,0). Use mixed values (e.g., 255,32,0) so DisableLighting ramp produces warm highlights. Red inserts need lower intensity than blue/white. In Blender, never set any channel to exactly 0 or 1. Use AgX tonemapper for red insert hotspots.

**Insert brightening when GI off:** Multiply insert IntensityScale when GI is off to simulate eye adaptation.

**Source:** VPW insert-lighting guide (Haunted House, Godzilla, Tales from the Crypt, F-14)

---

### Light Fading

**When to use:** Making lights fade realistically instead of instant on/off.

**Primary: VPX 10.8+ native Incandescent fader**
Set fader type to `Incandescent` on VPX light objects. Working values: 40ms up, 120ms down. This is the preferred approach — no script needed.

For flashers already receiving PWM modulation, set fader to `None` (PWM handles the curve). Can be set from script via the `Fader` property.

**Fallback: Lampz FadeSpeed**
For pre-10.8 compatibility or advanced control:
- Incandescent: fade up `1/40`, fade down `1/120` (asymmetric — filaments heat faster than they cool)
- LED: fade up `1/2`, fade down `1/8`

Use `Lampz.state()` instead of `LampState()` for lampz-enabled tables.

**Per-material fade curves** (for GI ON/OFF primitives):
- Cabinet/wood: `aLvl^5` (dramatic)
- Plastics: `aLvl^3` (moderate)
- Metal: `aLvl^2` (subtle)
- Bulb glass: `aLvl^0.5` (minimal)

**Source:** VPW gi-and-flashers guide (Johnny Mnemonic, Iron Man, TFTC)

---

### Light State Controller

**When to use:** Original (non-ROM) tables needing complex light behaviors: multi-state cycling, synchronized blink groups, spatial sequences.

**Setup requirements:**
1. Include the Light Controller class in table script
2. Initialize during `Table_Init`
3. **Enable `LampTimer`** — most common setup mistake. Without it, no light state changes process
4. On game start, call `lightCtrl.StopSyncWithVpxLights()` to stop attract patterns overriding game states

**Multi-state cycling:** A single light can have multiple stacked states (e.g., solid green + flashing red). The controller cycles between them with priority management.

**Blink groups (v0.9.1+):**
```vbs
AddLightToBlinkGroup "hurryUp", Light1
AddLightToBlinkGroup "hurryUp", Light2
UpdateBlinkGroupPattern "hurryUp", pattern
UpdateBlinkGroupInterval "hurryUp", interval
StartBlinkGroup "hurryUp"
' Later: StopBlinkGroup "hurryUp"
```

**Panlandia — coordinate-based sequences:** Uses light XY positions on the playfield for spatial effects: sweeps, waves, falling blocks. Effects created in 5-10 minutes by mapping light coordinates and triggering groups based on geometric patterns.

**Lightmap color sync:** Controller auto-syncs color changes to attached lightmaps. The lightmap primitive name must include the light name (e.g., `_l01_` in name for light `l01`), OR put the light name in the light's `UserValue` property.

**Gotchas:**
- ROM-based tables should use Lampz/vpmMapLights, not Light State Controller
- v0.9+ removed Lampz dependency — standalone system for originals
- Works alongside VPX 10.8+ `light_animate` system

**Source:** VPW light-state-controller reference (flux5009/mpcarr)

---

### Room Brightness

**When to use:** Controlling ambient light level on baked/lightmapped tables.

**The "one method only" rule:** Use `SetRoomBrightness` OR `primitive.color` — never both. Double-darkening occurs when both methods are active simultaneously.

```vbs
' In Table1_Init
SetRoomBrightness LightLevel/100
```

**Critical rules:**
- Always bake with room lighting ON, then darken in script. Cannot brighten beyond baked level.
- `DisableStaticPrerendering` (VPX 1666+) allows adjusting room brightness via in-game options menu without restart
- Day/Night slider affects ball brightness and VPX lights but NOT toolkit-baked primitive textures

**Source:** VPW gi-and-flashers guide (Godzilla, Johnny Mnemonic)

---

### PWM Integration

**When to use:** PinMAME 3.6+ tables with true lamp/flasher modulation.

**VPX 10.8+ native approach:**
1. Set `UseVPMModSol = 2` (new physics output; `=1` for backward compat)
2. PWM signals from core.vbs are float 0..1 range
3. Set light fader to `Incandescent` for lamps, `None` for flashers receiving PWM
4. Remove SolMask array initializations

**Niwak's PinMAME PWM:** Provides true-to-life slow fades, quick flashes, simultaneous fade-and-blink. Migration takes <5 minutes on lightmapped VPW tables.

**Platform needs:**
- WPC: needs PWM for GI, solenoids, sometimes lamps
- Whitestar/SAM: needs it for everything
- System 11 / Data East: less obvious benefit (GI uses relays)

**GI relay sounds with PWM:** Use threshold-based triggering, not on/off:
```vbs
If lvl < 0.01 And lvl <> lastLvl Then
    Sound_GI_Relay 0, Bumper1
ElseIf lvl > 0.99 And lvl <> lastLvl Then
    Sound_GI_Relay 1, Bumper1
End If
```
Without thresholds, relay sound plays on every PWM update (rapid ticking).

**Source:** VPW gi-and-flashers guide (Big Bang Bar, Godzilla, Jurassic Park)

---

## Anti-Patterns

### Pure Black (#000000) in Materials
**What it looks like:** Playfield areas or materials set to RGB(0,0,0).
**Why it's wrong:** Causes divide-by-zero in light bouncing calculations. Insert lightmaps get pruned (below 0.01 threshold). Broken lighting on dark playfield areas.
**Fix:** Use #111111 as minimum dark value.

### Pure RGB Insert Colors
**What it looks like:** Insert textures using 255,0,0 or 0,255,0 or 0,0,255.
**Why it's wrong:** Flat unrealistic appearance. No warm highlights when BDL increases. Red inserts lack visible hotspots.
**Fix:** Mix channels — e.g., RGB(255,32,0) instead of RGB(255,0,0). Never set any Blender channel to exactly 0 or 1 (use 0.1 minimum).

### Lampz Fading at Frame Rate
**What it looks like:** Lampz timer set to interval -1 (FrameTimer rate).
**Why it's wrong:** Fade speed becomes frame-rate-dependent. Fast machines get fast fades, slow machines get slow fades. Inconsistent visual behavior across systems.
**Fix:** Use a fixed-interval LampTimer for Lampz. Or use VPX 10.8+ native Incandescent fader (frame-rate-independent).
