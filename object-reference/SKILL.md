---
name: vpx-dev-object-reference
description: "VPX object types events properties methods scripting API reference bumper trigger kicker flipper spinner gate wall ramp target collection timer plunger primitive"
version: "1.0.0"
last_updated: "2026-03-24"
---

# VPX Object Reference

## When This Skill Applies

Use when a user needs to know what events, properties, or methods are available on a VPX table object. Essential for non-scripters who can place objects in the editor but don't know how to wire them up in VBScript.

Source: `vpinball.idl` from VPX 10.8.1 source (authoritative COM interface definitions).

## How VPX Scripting Works (For Non-Scripters)

Every object you place on the table can fire **events** — VPX calls your script when something happens. You write a Sub (a block of code) with the object's name + underscore + event name:

```vbs
' Object named "Target1" fires a "Hit" event when the ball hits it
Sub Target1_Hit()
    Score = Score + 1000
End Sub
```

That's it. Name the Sub correctly and VPX calls it automatically. No registration, no wiring.

**Collections** work the same way but pass an `index` parameter telling you WHICH member was hit:

```vbs
Sub TargetBounce_Hit(idx)
    TargetBouncer activeball, 2
End Sub
```

**CRITICAL: The `(idx)` parameter is required for collection events.** Without it, the sub signature doesn't match and silently fails — no error, no event.

---

## Object Event Reference

### Table

The table object itself. Named whatever your table is (usually `Table1`).

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Table loads. Put all initialization code here |
| `KeyDown(keycode)` | keycode (Long) | Any key pressed. Use for flipper input, start button, coin |
| `KeyUp(keycode)` | keycode (Long) | Any key released. Use for flipper release |
| `Exit()` | — | Table closing. Cleanup code |
| `Paused()` | — | Game paused |
| `UnPaused()` | — | Game unpaused |
| `MusicDone()` | — | Background music track finished |
| `OptionEvent(event)` | event (Long) | In-game option menu interaction (10.8.1+) |

**Common key constants:** `LeftFlipperKey`, `RightFlipperKey`, `PlungerKey`, `StartGameKey`, `AddCreditKey`, `LeftTiltKey`, `RightTiltKey`, `CenterTiltKey`

```vbs
Sub Table1_KeyDown(ByVal keycode)
    If keycode = LeftFlipperKey Then LeftFlipper.RotateToEnd
    If keycode = RightFlipperKey Then RightFlipper.RotateToEnd
    If keycode = StartGameKey Then StartGame
End Sub
```

---

### Bumper

Pop bumper. Fires the ball away on contact.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Hit()` | — | Ball hits the bumper |
| `Timer()` | — | Timer tick (if TimerEnabled) |
| `Animate()` | — | Visual update frame |

```vbs
Sub Bumper1_Hit()
    Score = Score + 100
    PlaySoundAtVol "bumper", Bumper1, VolumeDial
End Sub
```

**Key properties:** `Force` (kick strength, 13-15 typical), `Threshold` (minimum hit force to trigger), `Radius` (MUST match rubber skirt, not cap — see troubleshooting #10)

---

### Wall (Surface)

Walls, slingshots, and any flat barrier. Also used as invisible colliders.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Hit()` | — | Ball hits the wall |
| `Slingshot()` | — | Ball triggers slingshot action (wall must have sling properties set) |
| `Timer()` | — | Timer tick (if TimerEnabled) |

```vbs
Sub LeftSlingshot_Slingshot()
    Score = Score + 10
    PlaySoundAtVol "sling", LeftSlingshot, VolumeDial
End Sub
```

**Key properties:** `IsDropped` (True = wall removed from play, False = active), `Collidable`, `SlingshotForce`, `SlingshotThreshold`, `HasHitEvent`

**Gotcha:** `_Slingshot` event is separate from `_Hit`. Use `_Slingshot` for scoring/sound — it only fires on actual sling activation. `_Hit` fires on any contact.

---

### Flipper

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Hit()` | — | Ball touches flipper (any contact) |
| `Collide(velocity)` | velocity (Float) | Ball collides with moving flipper. The velocity parameter is collision strength |
| `Timer()` | — | Timer tick |
| `LimitEOS(angle)` | angle (Float) | Flipper reaches End Of Stroke (fully up) |
| `LimitBOS(angle)` | angle (Float) | Flipper reaches Beginning Of Stroke (rest position) |
| `Animate()` | — | Visual update frame |

**Key methods:** `RotateToEnd` (flip up), `RotateToStart` (return to rest)

**Key properties:** `StartAngle`, `EndAngle`, `Strength`, `EOSTorque`, `EOSTorqueAngle`, `Return`, `RampUp`

**WARNING:** Never create duplicate `_Collide` subs. VBScript silently uses the last one, killing nFozzy physics. If you paste nFozzy code that includes a `_Collide` sub, delete any existing one first.

---

### Trigger

Invisible area that detects ball entry/exit. The workhorse of game logic — use for lane detection, ramp entry/exit, rollover switches, nFozzy flipper triggers.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Hit()` | — | Ball ENTERS the trigger area |
| `UnHit()` | — | Ball EXITS the trigger area |
| `Timer()` | — | Timer tick |
| `Animate()` | — | Visual update frame |

```vbs
' Detect ball entering left inlane
Sub LeftInlane_Hit()
    Score = Score + 500
    PlaySoundAtVol "rollover", LeftInlane, VolumeDial
End Sub

' Detect ball direction on a ramp entry
Sub RampEntry_Hit()
    If activeball.vely < 0 Then    ' ball moving UP (toward backbox)
        WireRampOn True
    End If
End Sub
Sub RampEntry_UnHit()
    If activeball.vely > 0 Then    ' ball moving DOWN (toward drain)
        WireRampOff
    End If
End Sub
```

**Key properties:** `Enabled` (True/False — disable to ignore hits), `HitHeight` (how tall the detection zone is)

**Tip:** Triggers are invisible in play. Place them across lanes, at ramp entrances/exits, and anywhere you need to detect the ball passing through.

---

### Kicker

Holds and ejects balls. Used for scoops, VUKs, trough positions, drain, ball locks.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Hit()` | — | Ball enters the kicker (captured) |
| `UnHit()` | — | Ball exits the kicker (after kick) |
| `Timer()` | — | Timer tick |

```vbs
Sub Scoop1_Hit()
    PlaySoundAtVol "kicker_enter", Scoop1, VolumeDial
    ' Award points, start mode, etc.
    Score = Score + 5000
    ' Kick ball back out after a delay
    ScoopTimer.Interval = 500
    ScoopTimer.Enabled = True
End Sub

Sub ScoopTimer_Timer()
    ScoopTimer.Enabled = False
    Scoop1.Kick 270, 20    ' angle, speed
End Sub
```

**Key methods:**
- `Kick(angle, speed)` — Eject ball in X-Y plane. Angle = direction relative to object's angle
- `KickZ(angle, speed, inclination, heightz)` — Eject with vertical component (for VUKs)
- `CreateBall` / `CreateSizedBallWithMass(size, mass)` — Create a ball in the kicker
- `DestroyBall` — Remove ball (use sparingly — see ball philosophy in toys-mechs skill)

**Key properties:** `Enabled`, `HitAccuracy`, `Scatter` (random angle variation)

---

### HitTarget (Drop Target / Standup Target)

Physical targets that react to ball hits. Drop targets fall when hit; standup targets bounce the ball.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Hit()` | — | Ball hits the target face |
| `Dropped()` | — | Drop target falls down (drop targets only) |
| `Raised()` | — | Drop target resets to up position (drop targets only) |
| `Timer()` | — | Timer tick |
| `Animate()` | — | Visual update frame (used by DoDTAnim/DoSTAnim) |

```vbs
' Standup target
Sub StandupTarget1_Hit()
    Score = Score + 2500
End Sub

' Drop target bank of 3
Sub DropTarget1_Dropped()
    Score = Score + 1000
    CheckDropBank
End Sub

Sub CheckDropBank()
    If DropTarget1.IsDropped And DropTarget2.IsDropped And DropTarget3.IsDropped Then
        Score = Score + 10000
        ' Reset after delay
        ResetDropTimer.Interval = 2000
        ResetDropTimer.Enabled = True
    End If
End Sub

Sub ResetDropTimer_Timer()
    ResetDropTimer.Enabled = False
    DropTarget1.IsDropped = False
    DropTarget2.IsDropped = False
    DropTarget3.IsDropped = False
End Sub
```

**Key properties:** `IsDropped` (True = down, False = up), `Collidable`, `Threshold`

**Gotcha:** Use `_Dropped` (not `_Hit`) for scoring on drop targets — `_Hit` fires on any contact including glancing blows that don't drop the target.

---

### Spinner

Rotating gate that spins when the ball passes through. Scores on each revolution.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Spin()` | — | Each revolution of the spinner |
| `Timer()` | — | Timer tick |
| `LimitEOS(angle)` | angle (Float) | Spinner reaches end of rotation range |
| `LimitBOS(angle)` | angle (Float) | Spinner reaches beginning of rotation range |
| `Animate()` | — | Visual update frame |

```vbs
Sub Spinner1_Spin()
    Score = Score + 200
    PlaySoundAtVol "spinner", Spinner1, VolumeDial
End Sub
```

**Key properties:** `CurrentAngle` (read — use in FrameTimer for visual mesh animation), `Length`, `Damping`, `AngleMax`, `AngleMin`

---

### Gate

One-way gate that lets the ball through in one direction only.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Hit()` | — | Ball passes through the gate |
| `Timer()` | — | Timer tick |
| `LimitEOS(angle)` | angle (Float) | Gate reaches max open angle |
| `LimitBOS(angle)` | angle (Float) | Gate returns to closed position |
| `Animate()` | — | Visual update frame |

```vbs
Sub OutlaneGate_Hit()
    PlaySoundAtVol "gate", OutlaneGate, VolumeDial
End Sub
```

**Key properties:** `Open` (True = always open / disabled), `Damping`, `Collidable`

---

### Ramp

Physical ramp surface. **Very few script events** — most ramp logic uses Triggers placed at entry/exit points.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |

**Key properties:** `Collidable`, `HasWallImage`, `HeightBottom`, `HeightTop`, `WidthBottom`, `WidthTop`

**Tip:** Ramps themselves don't have Hit events. Place Trigger objects at ramp entrances and exits to detect ball movement. Use the ball's `vely` (negative = moving up) to determine direction.

---

### Primitive

3D mesh objects. Used for visual elements, toys, and custom collision geometry.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Hit()` | — | Ball hits the primitive (requires `HasHitEvent = True` AND `Collidable = True`) |

```vbs
Sub MyToy_Hit()
    PlaySoundAtVol "plastic_hit", MyToy, VolumeDial
End Sub
```

**Key properties:** `Collidable`, `HasHitEvent`, `IsToy` (True = never collidable, visual only), `Visible`, `Opacity`, `RotX`/`RotY`/`RotZ`, `ObjRotX`/`ObjRotY`/`ObjRotZ`, `BlendDisableLighting`, `Image`

**Gotcha:** Both `HasHitEvent` AND `Collidable` must be True for `_Hit` to fire. `IsToy = True` overrides Collidable.

---

### Light

VPX light object. Controls illumination and can be linked to lightmap primitives.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Timer()` | — | Timer tick |
| `Animate()` | — | Visual update frame |

**Note:** Lights have NO hit event. They are controlled entirely from script or Light State Controller. Use `State` to set on/off/blink, `IntensityScale` for brightness, `Color` for color.

**Key properties:** `State` (0=off, 1=on, 2=blink), `IntensityScale` (0.0-1.0), `Color`, `Fader` (None, Linear, Incandescent)

---

### Plunger

Ball launcher mechanism.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Timer()` | — | Timer tick |
| `LimitEOS(position)` | position (Float) | Plunger reaches full pull |
| `LimitBOS(position)` | position (Float) | Plunger returns to rest |

**Key properties:** `MechStrength`, `ParkPosition`, `MechPlunger` (True = analog plunger input)

---

### Timer

Standalone timer object. The backbone of game logic timing.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Timer()` | — | Each interval tick |

**Key properties:** `Interval` (ms, or -1 for per-frame), `Enabled` (True/False)

**CRITICAL rules from conventions skill:**
- Set `Interval` BEFORE `Enabled = True`
- Never use interval = 1 (fires 10+ times per frame)
- Interval -1 timers fire ONE extra time after disabling

---

### Rubber

Rubber ring/band objects.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Hit()` | — | Ball bounces off the rubber |
| `Timer()` | — | Timer tick |

**Key properties:** `Elasticity`, `ElasticityFalloff`, `Friction`, `Scatter`, `Collidable`

**Note:** With nFozzy rubber separation, visible rubbers are usually set non-collidable. Invisible dPosts/dSleeves handle the physics. The visible rubber's `_Hit` event may not fire. Use the collection hit events (`dPosts_Hit`, `Rubbers_Hit`) instead.

---

### Flasher

2D billboard element for flasher effects (not a light).

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Timer()` | — | Timer tick |

**Key properties:** `Opacity` (0-100), `Visible`, `Color`, `ImageA`/`ImageB`

**Note:** Flashers have no Hit event. They are purely visual. Control via SolCallback timers — see the lighting skill's flasher decay pattern.

---

### Textbox

On-screen text display (legacy — FlexDMD preferred for modern tables).

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Timer()` | — | Timer tick |

**Key properties:** `Text`, `FontColor`, `BackColor`, `Visible`

---

### DispReel

Mechanical score reel display.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Timer()` | — | Timer tick |
| `Animate()` | — | Visual update frame |

**Key methods:** `AddValue(value)`, `SetValue(value)`, `ResetToZero`

---

### LightSeq

Light sequencer for coordinated animations (attract mode patterns, etc.).

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Object created |
| `Timer()` | — | Timer tick |
| `PlayDone()` | — | Queued animation sequence completed |

**Key methods:** `Play(sequence, tailLength, repeat, pause)`, `StopPlay`

---

### Collection

A group of objects. Fire events for any member with a single Sub. **CRITICAL for reducing code — instead of writing 20 separate `_Hit` subs, put all targets in a collection.**

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Collection created |
| `Hit(index)` | index (Long) | Ball hits any member. Index = position in collection (0-based) |
| `UnHit(index)` | index (Long) | Ball exits any trigger member |
| `Slingshot(index)` | index (Long) | Slingshot fires on any wall member |
| `Spin(index)` | index (Long) | Any spinner member spins |
| `Dropped(index)` | index (Long) | Any drop target member drops |
| `Raised(index)` | index (Long) | Any drop target member raises |
| `Timer(index)` | index (Long) | Timer tick on any member |

```vbs
' All standup targets in one collection
Sub Standups_Hit(idx)
    Score = Score + 1000
    TargetBouncer activeball, 2    ' if using TargetBouncer
End Sub
```

**CRITICAL:** The `(idx)` parameter is REQUIRED even if you don't use it. Without it, the event silently never fires.

**Tip:** "Fire events for this collection" must be checked in VPX collection properties. This is the #1 reason collection events don't work.

---

### Ball

The ball itself. Rarely used directly in event subs.

| Event | Parameters | When It Fires |
|-------|-----------|---------------|
| `Init()` | — | Ball created |
| `Timer()` | — | Timer tick |

**Key properties (read from `activeball` in any hit event):**
- `x`, `y`, `z` — Position
- `velx`, `vely`, `velz` — Velocity (vely negative = moving toward backbox)
- `AngMomX`, `AngMomY`, `AngMomZ` — Angular momentum
- `Mass`, `Radius`
- `Color`, `Image`

---

## Global Event

### OnBallBallCollision

Not tied to any object. Fires whenever two balls collide anywhere on the table.

```vbs
Sub OnBallBallCollision(ball1, ball2, velocity)
    ' velocity = collision speed
    ' Used for FlipperCradleCollision dampening (see nfozzy-physics skill)
    ' Used for multiball collision sound throttling (see sound skill)
End Sub
```

---

## The `activeball` Variable

Inside any `_Hit`, `_UnHit`, `_Spin`, `_Slingshot`, `_Collide`, or `_Dropped` event, VPX provides the `activeball` variable — the ball that caused the event. Use it to read velocity, position, or modify the ball:

```vbs
Sub Ramp1Entry_Hit()
    ' Only score if ball is moving upward (entering ramp, not rolling back)
    If activeball.vely < 0 Then
        Score = Score + 5000
    End If
End Sub
```

---

## Common Patterns for Non-Scripters

### "I placed a target, how do I score points?"
Name the target in VPX (e.g., `MyTarget`), then:
```vbs
Sub MyTarget_Hit()
    Score = Score + 1000
End Sub
```

### "I want something to happen after a delay"
Place a Timer object, name it (e.g., `DelayTimer`), set Enabled = False:
```vbs
' Start the delay (call from wherever)
Sub StartDelay()
    DelayTimer.Interval = 2000    ' 2 seconds
    DelayTimer.Enabled = True
End Sub

Sub DelayTimer_Timer()
    DelayTimer.Enabled = False    ' Fire once only
    ' Do the delayed thing here
End Sub
```

### "I want to detect the ball entering/leaving an area"
Place a Trigger object over the area:
```vbs
Sub MyArea_Hit()
    ' Ball entered
End Sub
Sub MyArea_UnHit()
    ' Ball left
End Sub
```

### "I want a drop target bank that resets when all are down"
See the HitTarget section above for the complete pattern.

### "I want to kick the ball out of a hole after a pause"
See the Kicker section above for the scoop + timer pattern.
