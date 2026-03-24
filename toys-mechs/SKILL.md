---
name: vpx-dev-toys-mechs
description: "VPX VBScript pinball toy mechanism diverter magnet VUK kicker spinner motor drop target captive ball Newton ball carousel stepper cvpmMech trough"
version: "1.0.0"
last_updated: "2026-03-24"
---

# Toys & Mechanisms

## When This Skill Applies

Use when implementing: diverters, magnets, VUKs/kickers, spinners, drop targets, motor mechanisms (carousels, goalies, drawbridges), captive/Newton balls, or any mechanical table toy. Also applies to ball creation/destruction philosophy.

Always apply conventions skill rules (naming, timers, standalone compatibility) to all generated VBScript.

## Ball Philosophy

Real machines don't create or destroy balls. Create all balls once at initialization, track with `gBOT`, never destroy. Multiball = release balls from physical trough kickers, not `CreateBall`.

**Source:** VPW best practice from TFTC, Road Show

## Diverter

SolCallback activates/deactivates a wall or kicker to redirect ball path. Track position state for sound and DOF.

```vbs
SolCallback(21) = "SolDiverter"

Sub SolDiverter(enabled)
    If enabled Then
        DiverterWall.IsDropped = False
        PlaySoundAtVol "Diverter_On", DiverterWall, VolumeDial
    Else
        DiverterWall.IsDropped = True
        PlaySoundAtVol "Diverter_Off", DiverterWall, VolumeDial
    End If
End Sub
```

**Gotchas:**
- **Diverters are an exception to the `If Enabled Then` guard pattern.** They need action on BOTH activation (open) and deactivation (close). The `If Enabled Then` guard rule applies to momentary solenoids where deactivation should be silent (bumpers, kickers, etc.)
- Diverter solenoids should NOT have DOF contactor effects -- they hold position and can burn physical cabinet solenoids. Only momentary solenoids (bumpers, slings, kickers) get contactors
- Use `IsDropped` for walls, `Enabled` for kickers depending on mechanism type

**Source:** F-14 Tomcat, Hook, best-practices.md

## Magnet

VPX magnet objects capture balls with `GrabCenter`. Enable after a delay to simulate gradual pull.

```vbs
Sub SolMagnet(enabled)
    If enabled Then
        Magnet1.Enabled = True
        Magnet1.GrabCenter = False
        MagnetGrabTimer.Interval = 750
        MagnetGrabTimer.Enabled = True
    Else
        Magnet1.Enabled = False
        Magnet1.GrabCenter = False
        MagnetGrabTimer.Enabled = False
    End If
End Sub

Sub MagnetGrabTimer_Timer()
    MagnetGrabTimer.Enabled = False
    Magnet1.GrabCenter = True
End Sub
```

For toys that grab and move the ball (data glove, etc.), use a 1ms timer that continuously positions the ball at the toy location. Do NOT destroy/create balls -- causes visual pops.

**Gotchas:**
- `GrabCenter = True` immediately causes unnatural "wiggle" -- delay 500-1000ms
- VPX magnets cannot accurately simulate multi-ball magnet interaction (balls fire off unpredictably). Offer different ROM versions as workaround
- Stern SAM magnet boards have onboard processors -- PinMAME sends binary on/off only, not PWM. Script must approximate real hardware timing

**Source:** Metallica (GrabCenter timing), Twilight Zone (multi-ball limitation), Johnny Mnemonic (ball-follow pattern)

## VUK (Vertical Up Kicker)

```vbs
Sub SolVUK(enabled)
    If enabled Then
        PlaySoundAtVol "VUK_Up", VUK1, VolumeDial
        VUK1.KickZ angle, speed, inclination, heightz
    End If
End Sub
```

**KickZ parameters:** `KickZ(angle, speed, inclination, heightz)` -- angle = direction in X-Y plane relative to red arrow, speed = ball velocity, inclination = angle above X-Y plane (90 = straight up), heightz = Z translation before kick. A small non-zero `heightz` is sometimes required for the method to work.

**Kick direction:** Use Y-axis for forward kick onto playfield, NOT Z-axis (Z kicks straight up).

Add variance for realism:
```vbs
Function KickoutVariance(aNumber, aVariance)
    KickoutVariance = aNumber + ((Rnd*2)-1)*aVariance
End Function
' Usage: VUK1.Kick KickoutVariance(angle, 3), KickoutVariance(speed, 2)
```

**Gotchas:**
- Release ball variable after kicking -- holding it causes floating/stuck balls on ball search
- Multiple balls in a scoop can stack and jam -- detect with `InRect` + z-height, kick top ball first

**Source:** Road Show (KickZ params), Twilight Zone (variance), Hook (Y vs Z axis), Guns N Roses (stacking)

## Spinner

Two baked primitive meshes at 0 and 180 degrees, opacity cross-faded in FrameTimer:

```vbs
Sub FrameTimer_Timer()
    Dim spinAngle: spinAngle = Spinner1.CurrentAngle
    Dim t: t = (1 + Cos(spinAngle * 3.14159 / 180)) / 2
    SpinnerPrim0.Opacity = t
    SpinnerPrim180.Opacity = 1 - t
End Sub
```

**Gotchas:**
- 180-degree mesh must be slightly larger (e.g., scale 0.575 vs 0.570) to prevent z-fighting during cross-fade
- Both meshes go in VLM.Movables collection

**Source:** Harlem Globetrotters

## Drop Targets

Roth animation system: primitive mesh faces drain at RotZ=0. DoDTAnim runs in FrameTimer (tween-based, safe at frame rate).

```vbs
' In FrameTimer:
Sub FrameTimer_Timer()
    DoDTAnim
    DoSTAnim
End Sub
```

**Gotchas:**
- RotZ=0 MUST face drain. Using ObjRotZ instead of RotZ breaks all hit angle calculations (brick check evaluates perpendicular velocity using RotZ)
- Standup targets need backstop primitives (suffix "o", e.g., "sw24o") -- collidable but not in script
- Drop target layback of 4-7 degrees produces realistic bounce
- Angled sub-playfields: DoDTAnim resets RotX to zero. Add playfield angle offset using z-height check to identify affected targets

**Source:** Goonies (RotZ=0), Judge Dredd (backstops), Countdown (angled playfields), physics-tuning.md (layback)

## Motor Mechanisms

Use `cvpmMech` for ROM-controlled stepper motors (drawbridges, carousels, goalies). Wire Sol1/Sol2 for direction, limit switches for position feedback.

**Critical:** Set `HandleMech=0` in table script to prevent PinMAME mech handling interference. Without this, stepper motors stop responding during gameplay (but work in self-test mode). Initialize limit switches in `Table1_Init` AFTER PinMAME timer starts.

**Non-linear motion:** For mechs that pause at extremes (goalie), use a manually-defined position array stepped by timer ticks instead of linear interpolation.

**Carousel collision:** VPX collidable primitives cannot move at runtime. Use an array of VPX flipper objects arranged in a circle -- flippers are natively collidable AND movable.

**Motor sounds:** Use `PlaySoundAtLevelStaticLoop` -- loops until explicitly stopped. Start on SolCallback True, stop on False. Avoid repeated `PlaySound` calls (audible gaps).

**Motor drift:** ROM can send erratic position values after homing. Use a boolean flag in the SolCallback -- when solenoid deactivates and position > threshold, force park position.

**Source:** Medieval Madness (HandleMech, drawbridge init), World Cup Soccer (goalie arrays), Indiana Jones (flipper carousel), Black Rose / Austin Powers (motor sounds/drift)

## Captive / Newton Ball

Physical mechanism: oversized ball (57 wide, vs standard 50) at a post position transfers force via Newton's cradle effect. VPX ball collision physics handle the force transfer naturally.

```vbs
' Create captive ball when post is raised
Sub RaiseCaptivePost()
    Set CaptiveBall = CaptiveKicker.CreateSizedBallWithMass(57, 1)
    CaptiveBall.Collidable = True
End Sub

' Remove when post is lowered
Sub LowerCaptivePost()
    If Not CaptiveBall Is Nothing Then
        CaptiveBall.DestroyBall
        Set CaptiveBall = Nothing
    End If
End Sub
```

**NOTE:** This skill covers the physical mechanism (mesh, sizing, position). Game-logic skill owns when to lock/release (mode scripting, lock counting). No overlap -- this is "how the captive ball works physically," game-logic is "when to activate it."

**Gotchas:**
- CorTracker crashes on captive balls with zero distance to post boundary -- add zero-distance guard in trajectory calculation
- Ball philosophy exception: captive balls ARE created/destroyed because they represent a fixed mechanism, not a ball in play

**Source:** Breakshot (57-wide, Newton cradle), Indiana Jones (CorTracker crash), AC/DC + JWPS (same technique)

## Anti-Patterns

### Creating Balls for Multiball
**What it looks like:** `CreateBall` / `DestroyBall` to add/remove balls during multiball
**Why it's wrong:** Real machines don't create balls. Ball IDs change on recreate, breaking shadow tracking. `cvpmBallStack` approach sometimes fails.
**Fix:** Physical trough with kickers. Create all balls at init, never destroy. Track with `gBOT`.

### Motor Timer at 1ms
**What it looks like:** Mech animation timer with `Interval = 1`
**Why it's wrong:** 1ms fires 1000x/sec -- massive CPU waste. Halved FPS on Road Show.
**Fix:** Set mech timers to 20ms with adjusted speed values.
