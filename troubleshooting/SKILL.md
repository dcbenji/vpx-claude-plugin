---
name: vpx-dev-troubleshooting
description: "VPX VBScript pinball debug fix error crash ball through flipper wrong physics broken frame rate fps performance standalone z-fighting NVRAM FlexDMD sounds AudioPan timer stutter ramp stuck bumper trapping"
version: "1.0.0"
last_updated: "2026-03-24"
---

# VPX Troubleshooting

Symptom-first diagnosis for VPX table bugs. Each entry: what you see, what's wrong, how to fix it.

## When This Skill Applies

User reports a bug, crash, visual glitch, physics problem, or performance issue in a VPX table.

## Physics

### 1. Ball passes through walls/ramps
**Check:** Normals direction, collidable flag, mesh thickness, duplicate collidable primitives.
**Fix:** Flip normals so they face outward on all hittable surfaces. Two-sided collision requires mesh thickness (not a flat plane). Chamfer sharp 90-degree edges — fast balls can pass through between physics frames.
**Source:** troubleshooting.md — Jokerz, Metallica

### 2. Ball sinks through playfield
**Check:** Collidable primitives used as rest surfaces, holes cut for rollover switches.
**Fix:** Use VPX walls (not primitives) for rest surfaces. Add under-playfield blocker wall at Z=-0.01. Never cut geometry holes for rollover switches — use transparent texture areas instead.
**Source:** troubleshooting.md — Batman DE

### 3. Flippers feel wrong
**Check:** (a) Duplicate `_Collide` subs — VBScript silently overrides, killing nFozzy physics. (b) Flipper trigger sizing — should be 27 VP. (c) Era curve mismatch. (d) Ball mass != 1. (e) Global Physics Sets conflict. (f) EOS torque corruption from locale settings (comma vs dot decimal).
**Fix:** (a) Merge into one sub. (b) Set 27 VP margin. (c) Match era. (d) Set mass=1. (e) Remove GPS. (f) Hard-code flipper params in script.
**Source:** troubleshooting.md — Austin Powers, Ghostbusters, Game of Thrones, Iron Maiden

### 4. Physics materials zeroed out
**Check:** F11 dotted ball test — if ball spins abnormally, materials are zeroed. Happens after certain VPX operations strip .vpp data.
**Fix:** Re-import the .vpp physics preset file.
**Source:** troubleshooting.md — physics-tuning.md

### 5. Flipper jam after nFozzy integration
**Check:** Sub-pixel flipper alignment — 1-2 pixel positioning error.
**Fix:** Move flipper by 1 pixel in any direction.
**Source:** troubleshooting.md — Ghostbusters

### 6. Ball hovers near flipper
**Check:** Playfield mesh polygon edges intersecting flipper swing arc.
**Fix:** Remove loop cuts and simplify mesh geometry in the flipper swing zone.
**Source:** troubleshooting.md — Congo, Batman DE

### 7. Ball stutters/jitters on surfaces
**Check:** Duplicate collidable primitives overlapping (visual + physics both collidable).
**Fix:** Only ONE primitive collidable per location. Visual primitive: "Is Toy" checked or collidable unchecked.
**Source:** troubleshooting.md — Fish Tales

### 8. Ramp ball stuck
**Check:** (a) Friction too high — 0.8 is velcro. (b) Collidable geometry gaps at joints. (c) Geometry bumps in ramp profile.
**Fix:** (a) Unify ramp friction to ~0.2. (b) Check physics visualization for gaps. (c) Smooth fractional bumps. For persistent stuck spots, use scripted velocity nudge: check position+height+low velocity, add small directional vely nudge.
**Source:** troubleshooting.md — Austin Powers, Diner, Jokerz, Godzilla

### 9. Ramp exit corkscrew
**Check:** Ramp exit spin multiplier (VPX default is 70, too high).
**Fix:** Reduce to 50.
**Source:** physics-tuning.md

### 10. Pop bumper trapping (20+ seconds)
**Check:** Bumper radius — must match rubber skirt/ring, NOT visual hat/cap size.
**Fix:** Reduce radius (e.g., BBB: 43 to 38). Broadly applicable correction.
**Source:** troubleshooting.md — Big Bang Bar, RBION

### 11. Outlane ball curling over post
**Check:** Post/wall collision geometry allows ride-up trajectory.
**Fix:** Adjust geometry to redirect ball upward instead of allowing curl-over path.
**Source:** troubleshooting.md — Iron Maiden, Iron Man

## Rendering

### 12. Z-fighting / overlapping primitives flicker
**Check:** Depth bias values. Counter-intuitive: TOP element = biggest negative (-10000), BOTTOM = positive.
**Fix:** ON inserts = 0 (top), OFF inserts = 30 (behind). Ball shadows: Z height 1.01 VP above playfield. Drop target flickering: uncheck "Unshaded Additive Blend" on CabLit Layer. Depth bias can silently reset after crashes/upgrades — always verify.
**Source:** troubleshooting.md — Batman DE, Earthshaker, Big Bang Bar

### 13. Frame rate drops
**Check:** (a) Timer interval 1 (change to -1). (b) GetBalls in rolling sub (use gBOT). (c) Render probe count + roughness (set to 0 = up to 40% FPS gain). (d) High-poly collidable primitives. (e) Transparency overdraw (biggest performance killer). (f) debug.print in hot paths.
**Fix:** Address in order of impact: probes, transparency, timers, GetBalls, poly count.
**Source:** troubleshooting.md — Fish Tales, Bad Cats, Radical

### 14. Timer catch-up spiral (escalating stutter)
**Check:** Fixed-rate timers at small intervals (1ms, 5ms) creating catch-up loop.
**Fix:** Use -1 timers (per-frame) for visuals. Move cor.update to frame timer (recovered 20-30 FPS on Hook). Only use fixed intervals >30ms with lightweight computation. Note: -1 timers execute one extra time after disable.
**Source:** troubleshooting.md — Bram Stoker's Dracula, Hook, Fish Tales

### 15. Lightmap pruning on dark playfields
**Check:** Toolkit prunes lightmaps below 0.01 threshold. Near-black playfield = near-zero influence.
**Fix:** Never use pure black (#000000) — minimum #111111. Add gamma node or noise to dark areas.
**Source:** troubleshooting.md — Beat the Clock, Johnny Mnemonic

### 16. 32-bit VPX texture memory (blurry GI)
**Check:** Running 32-bit VPX — lightmap nestmaps auto-downsampled.
**Fix:** Use 64-bit VPX. If stuck on 32-bit: downsize EXR lightmaps to 4K + DWAA compression in Blender.
**Source:** troubleshooting.md — Fish Tales

### 17. Render probe roughness VR artifacts
**Check:** Non-zero roughness on playfield reflection probes causes white flashing + ghosting in VR.
**Fix:** Set roughness to 0. Use two probes: one with roughness (desktop), one without (VR).
**Source:** troubleshooting.md — Bad Cats, Breakshot, Fish Tales

### 18. Scene excessively dark (double-darkening)
**Check:** Both `primitive.color` AND `SetRoomBrightness` applied simultaneously.
**Fix:** Use only one method — prefer `SetRoomBrightness` (current standard).
**Source:** troubleshooting.md — CFTBL

## Scripting

### 19. cor.BallVel error
**Check:** Missing GameTimer with `Cor.Update` call.
**Fix:** Add GameTimer (10ms interval) containing `Cor.Update`.
**Source:** troubleshooting.md — conventions

### 20. Sounds play twice
**Check:** Missing `If Enabled Then` guard in SolCallback.
**Fix:** Add `If Enabled Then` at top of ALL SolCallback subs. Callback fires for both True (activate) and False (deactivate).
**Source:** troubleshooting.md — Medieval Madness

### 21. Standalone crash on flipper hit
**Check:** `gBOT` used in `Cor.Update` — causes segfault in standalone. This is 95% of "flipper crash" reports.
**Fix:** Replace `gBOT` with `GetBalls` in Cor.Update ONLY. gBOT is correct everywhere else.
**Source:** troubleshooting.md — Defender

### 22. Options menu breaks rendering (10.8.1+)
**Check:** `DisableStaticPreRendering` spammed on every option change.
**Fix:** Set `True` once on menu entry, `False` once on exit. 10.8.1 uses counter semantics — spamming True breaks it.
**Source:** requirements.md — Req 8.12

### 23. FlexDMD broken
**Check:** (a) Detection via filesystem instead of error handling. (b) Scene arrays recreated per show instead of initialized once. (c) Update timer disabled between games.
**Fix:** (a) Use `On Error Resume Next` + object creation test. (b) Init scene arrays once. (c) Keep timer enabled.
**Source:** troubleshooting.md — game-rules-mechanics.md

### 24. vpmTimer.Add broken
**Check:** Cryptic `VBSE_OLE` errors with no line reference, ~1000+ vpmtimer calls.
**Fix:** Never use vpmTimer.Add. Convert to queue/tick timers. Also check for globalplugin.vbs namespace collisions — prefix custom functions.
**Source:** troubleshooting.md — Iron Maiden, MF DOOM

### 25. Fleep AudioPan crash on standalone
**Check:** Walls passed to AudioPan — walls lack X/Y properties.
**Fix:** Only pass objects with positional properties (primitive, kicker, target) to AudioPan. Guard with property check before panning.
**Source:** troubleshooting.md — Breakshot, Fish Tales

### 26. Ball-in-Narnia (fell through geometry)
**Check:** Ball z-position far below 0.
**Fix:** Loop through gBOT checking z-position. Return balls with z far below 0 to a scoop/VUK.
**Source:** troubleshooting.md — Batman DE

### 27. Bumpers/slings permanently broken after tilt+crash
**Check:** VPX saves table state on exit — tilt sets thresholds to 100 (disabled), crash preserves that.
**Fix:** Re-download table, or set all bumper/sling thresholds at table init to prevent persisted corruption.
**Source:** troubleshooting.md — Jokerz

### 28. Ball collision sounds excessive in multiball
**Check:** Every ball-ball collision fires sound with no throttling.
**Fix:** Track ball IDs + use timer to throttle — only fire sound if >N ms since last collision sound for that ball pair.
**Source:** best-practices.md

## ROM / PinMAME

### NVRAM corrupted (erratic behavior)
**Check:** Switches not registering, scores not tracking, mechanisms misfiring despite correct script.
**Fix:** Delete `.nv` file from VPinMAME nvram directory (e.g., `cc_12.nv`). Delete `.cfg` file too if needed. Always try NVRAM deletion before debugging script.
**Source:** troubleshooting.md — Cactus Canyon, Monster Bash

### ROM won't accept credits (Bally 6803)
**Check:** Missing PulseTimer for ROM communication.
**Fix:** Add PulseTimer with correct interval. Test coin-up as part of initial setup.
**Source:** troubleshooting.md — Flash Gordon
