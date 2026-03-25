# nFozzy Physics Reference

Comprehensive reference extracted from the nFozzy Physics Dropbox Paper document (the VPW community standard for Visual Pinball physics). All values are authoritative for VPX/VPE physics tuning.

Updated values from rothbauerw's VPW wiki refinements (2025 community consensus) are marked with `[UPDATED - VPW wiki 2025]`. Original nFozzy values are preserved as `[ORIGINAL]` where they differ.

---

## Table of Contents

1. [Ball Properties](#ball-properties)
2. [Table Physics](#table-physics)
3. [Flipper Physics by Era](#flipper-physics-by-era)
4. [FlipperTricks FSM](#flippertricks-fsm)
5. [FlipperCorrection (Polarity/Velocity)](#flippercorrection-polarityvelocity)
6. [LiveCatch](#livecatch)
7. [Rubber COR Curves](#rubber-cor-curves)
8. [Material Presets](#material-presets)
9. [Bumper Settings](#bumper-settings)
10. [Slingshot Settings](#slingshot-settings)
11. [Drop Target Constants](#drop-target-constants)
12. [Flipper Geometry Notes](#flipper-geometry-notes)
13. [LinearEnvelope Function](#linearenvelope-function)
14. [Lane Guide & Technique Notes](#lane-guide--technique-notes)

---

## Ball Properties

| Property | Value | Unit |
|----------|-------|------|
| Diameter | 50 | VP units |
| Mass | 1 | (unitless) |

Standard 1-1/16" steel pinball = 50 VP units diameter, mass 1.

---

## Table Physics

Baseline: Terminator 2 (community reference table).

| Property | Value | Notes |
|----------|-------|-------|
| Gravity Constant | 0.97 | |
| Difficulty | 56 | |
| Min Slope | 6 degrees | |
| Max Slope | 7 degrees | |
| Play Slope | 6.56 degrees | Actual gameplay slope |
| Playfield Friction | 0.15 - 0.25 | Range; tune per table |
| Playfield Elasticity | 0.25 | |
| Playfield Elasticity Falloff | 0 | |
| Playfield Scatter | 0 degrees | Avoid anomalies |
| Default Element Scatter | 2 degrees | Applied to most objects |

---

## Flipper Physics by Era

### Constants by Era

| Parameter | EM | Late 70s - Early 80s | Mid 80s | Late 80s - Early 90s | Mid 90s+ |
|-----------|----|-----------------------|---------|----------------------|----------|
| EOSTnew | 1.0 | 1.0 | 1.0 | 0.8 | 0.8 |
| EOSAnew | 4 | 4 | 6 | 6 | 4 |
| EOSRampup | 0 | 0 | 0 | 0 | 0 |
| SOSRampup | 2.5 | 2.5 | 2.5 | 2.5 | 2.5 |
| Coil Rampup | varies | 2.5 | 2.5 | 2.5 | 2.5 |
| EOS Torque | 0.3 | 0.3 | 0.275 | 0.275 | 0.3 |
| Return Strength `[ORIGINAL]` | 0.055 | 0.045 | 0.045 | 0.035 | 0.025 |
| Return Strength `[UPDATED - VPW wiki 2025]` | **0.048** | 0.045 | 0.045 | 0.035 | **0.018** |
| TimeDelay | 80ms | 80ms | 80ms | 60ms | 60ms |
| SOSEM | 0.815 | 0.815 | 0.815 | 0.815 | 0.815 |
| LiveCatch | 16ms | 16ms | 16ms | 16ms | 16ms |
| LiveElasticity | 0.45 | 0.45 | 0.45 | 0.45 | 0.45 |

Notes on updated return strength:
- EM: `[ORIGINAL]` 0.055, `[UPDATED - VPW wiki 2025]` **0.048**
- Mid 90s+: `[ORIGINAL]` 0.025, `[UPDATED - VPW wiki 2025]` **0.018**

### EOSTNew / EOSReturn Formula-Based Approach

`[UPDATED - VPW wiki 2025]` VPE PR #443 introduced formula-based calculation that scales properly with flipper strength:

```
EOSTNew = FlipperStrength * (-0.00025) + 1.57
EOSReturn = FlipperStrength * (-0.0000119) + 0.06348
```

These formulas produce the same values at standard flipper strengths but scale correctly for non-standard configurations. Use these instead of the era-based lookup when flipper strength is known.

### Flipper Strength Ranges by Era

| Era | Strength Range |
|-----|---------------|
| EM | 500 - 1000 |
| Late 70s - Early 80s | 1400 - 1600 |
| Mid 80s - Early 90s | 2000 - 2600 |
| Mid 90s+ | 3200 - 3300 |

### FlipperTricks Constants (VBScript)

```vbscript
Const EOSTnew = 0.8           ' late 80s onward (1.0 for EM-80s)
Const EOSAnew = 1             ' EOS torque angle (degrees)
Const EOSRampup = 0           ' ramp at EOS
Const SOSRampup = 2.5         ' start-of-stroke rampup
Const LiveCatch = 16          ' millisecond window
Const LiveElasticity = 0.45   ' elasticity during catch
Const SOSEM = 0.815           ' start-of-stroke elasticity multiplier
Const EOSReturn = 0.018       ' return strength (mid-90s+) [UPDATED - VPW wiki 2025]
Const LiveDistanceMin = 30    ' VP units from flipper base
Const LiveDistanceMax = 114   ' VP units (tip protection)
```

---

## FlipperTricks FSM

The FlipperTricks system is a finite state machine that runs on a 1ms timer, continuously adjusting flipper physics properties to simulate real coil behavior.

### Timer Setup

```vbscript
RightFlipper.timerinterval = 1
RightFlipper.timerenabled = True
```

### States

#### State 0: Idle
- Default state, no adjustments active.

#### State 1: Start-of-Stroke (SOS)
- **Entry condition:** `Abs(Flipper.currentangle) > Abs(Flipper.startangle) - 0.05`
- **Actions:**
  - Sets `Flipper.rampup = SOSRampup` (2.5)
  - Adjusts end angle: `Flipper.endangle = FEndAngle - 3*Dir` (3-degree overshoot)
  - Applies elasticity reduction: `Flipper.Elasticity = FElasticity * SOSEM` (0.815 multiplier)
  - Resets `FCount = 0`

#### State 2: End-of-Stroke (EOS)
- **Entry condition:** `Abs(currentangle) <= Abs(endangle) AND FlipperPress = 1`
- **Actions:**
  - Sets `eostorque = EOSTnew`
  - Sets `eostorqueangle = EOSAnew` (1 degree)
  - Zeroes ramp-up: `rampup = 0`
  - Applies EOS torque and angle adjustments to control flipper at max extension

#### State 3: Return / Deactivation
- **Entry condition:** `Abs(currentangle) > Abs(endangle) + 0.01 AND FlipperPress = 1`
- **Actions:**
  - Restores original torque: `eostorque = EOST`
  - Returns elasticity: `Elasticity = FElasticity`
  - Dampens ball velocity in cradle to prevent bouncing
  - Applies return torque reduction

### State Transitions

```
Idle ──────────────────────────────────> State 1 (SOS)
  Condition: Abs(angle) > Abs(startangle) - 0.05

State 1 (SOS) ─────────────────────────> State 2 (EOS)
  Condition: Abs(angle) <= Abs(endangle) AND button pressed

State 2 (EOS) ─────────────────────────> State 3 (Return)
  Condition: Abs(angle) > Abs(endangle) + 0.01 AND button pressed

State 3 (Return) ──────────────────────> Idle
  Condition: Button released (FlipperPress = 0)
```

### Dir Variable

```vbscript
Dir = Flipper.startangle / Abs(Flipper.startangle)
```

- Left flipper: Dir = 1
- Right flipper: Dir = -1

### FElasticity Initialization

```vbscript
FElasticity = LeftFlipper.elasticity
```

Read directly from the flipper object's elasticity property at initialization.

---

## FlipperCorrection (Polarity/Velocity)

The FlipperCorrection system (implemented as `FlipperPolarity` class) adjusts ball trajectory after flipper contact to simulate real-world flipper behavior. Position 0 = flipper base, position 1 = flipper tip.

### Initialization Structure

```vbscript
Dim LF, RF
Set LF = New FlipperPolarity
Set RF = New FlipperPolarity

Sub InitPolarity()
    ' Add polarity points
    LF.AddPt "Polarity", idx, position, adjustment
    ' Add velocity points
    LF.AddPt "Velocity", idx, position, coefficient
    ' Add Ycoef safety points
    LF.AddPt "Ycoef", idx, y_position, coefficient

    ' Link objects
    LF.Object = LeftFlipper
    LF.EndPoint = EndPointLp

    ' Bind triggers
End Sub
```

### Trigger Binding

```vbscript
Sub TriggerLF_Hit()
    LF.Addball activeball
End Sub

Sub TriggerLF_UnHit()
    LF.PolarityCorrect activeball
End Sub
```

### Flipper Activation (replaces RotateToEnd)

```vbscript
LF.fire    ' Instead of LeftFlipper.RotateToEnd
```

### Polarity Curves by Era

#### Late 70s to Early 80s

| Position | Polarity |
|----------|----------|
| 0 | 0 |
| 0.05 | -2.7 |
| 0.33 | -2.7 |
| 0.37 | -2.7 |
| 0.41 | -2.7 |
| 0.45 | -2.7 |
| 0.576 | -2.7 |
| 0.66 | -1.8 |
| 0.743 | -0.5 |
| 0.81 | -0.5 |
| 0.88 | 0 |

#### Mid 80s

| Position | Polarity |
|----------|----------|
| 0 | 0 |
| 0.05 | -3.7 |
| 0.33 | -3.7 |
| 0.37 | -3.7 |
| 0.41 | -3.7 |
| 0.45 | -3.7 |
| 0.576 | -3.7 |
| 0.66 | -2.3 |
| 0.743 | -1.5 |
| 0.81 | -1.0 |
| 0.88 | 0 |

#### Late 80s to Early 90s

| Position | Polarity |
|----------|----------|
| 0 | 0 |
| 0.05 | -5.0 |
| 0.4 | -5.0 |
| 0.6 | -4.5 |
| 0.65 | -4.0 |
| 0.7 | -3.5 |
| 0.75 | -3.0 |
| 0.8 | -2.5 |
| 0.85 | -2.0 |
| 0.9 | -1.5 |
| 0.95 | -1.0 |
| 1.0 | -0.5 |
| 1.1 | 0 |
| 1.3 | 0 |

#### Early 90s and After

| Position | Polarity |
|----------|----------|
| 0 | 0 |
| 0.05 | -5.5 |
| 0.4 | -5.5 |
| 0.6 | -5.0 |
| 0.65 | -4.5 |
| 0.7 | -4.0 |
| 0.75 | -3.5 |
| 0.8 | -3.0 |
| 0.85 | -2.5 |
| 0.9 | -2.0 |
| 0.95 | -1.5 |
| 1.0 | -1.0 |
| 1.05 | -0.5 |
| 1.1 | 0 |
| 1.3 | 0 |

### Velocity Curve (Identical Across All Eras)

| Position | Coefficient |
|----------|-------------|
| 0 | 1.0 |
| 0.16 | 1.06 |
| 0.41 | 1.05 |
| 0.53 | 0.982 |
| 0.702 | 0.968 |
| 0.95 | 0.968 |
| 1.03 | 0.945 |

### Ycoef Safety Points (Identical Across All Eras)

| Ball Y Position | Coefficient |
|-----------------|-------------|
| RightFlipper.Y - 65 | 1 (disabled) |
| RightFlipper.Y - 11 | 1 (enabled) |

Applied when ball position > 0.65 on flipper to prevent tip bounce anomalies.

### Correction Algorithm

**Polarity Correction:**

```vbscript
AddX = LinearEnvelope(BallPos, PolarityIn, PolarityOut) * LR
VelX = VelX + 1 * (AddX * ycoef * PartialFlipcoef)
```

- `LR` = -1 for left flipper, +1 for right flipper
- `BallPos` = normalized position along flipper (0-1)

**Partial Flip Coefficient:**

```vbscript
PartialFlipCoef = ((Flipper.StartAngle - Flipper.CurrentAngle) / _
    (Flipper.StartAngle - Flipper.EndAngle))
PartialFlipCoef = Abs(PartialFlipCoef - 1)
```

**Velocity Correction:**

```vbscript
VelCoef = LinearEnvelope(BallPos, VelocityIn, VelocityOut)
' If partial flip: interpolate toward 1.0
VelX = VelX * VelCoef
VelY = VelY * VelCoef
```

### TimeDelay

Correction is disabled after TimeDelay milliseconds from flipper activation:
- Late 70s - Mid 80s: 80ms
- Late 80s onward: 60ms

---

## LiveCatch

LiveCatch simulates the real-world technique of catching a ball on a held flipper by timing the button press to absorb the ball's momentum.

### Parameters

| Parameter | Value | Unit |
|-----------|-------|------|
| LiveCatch window | 16 | ms |
| LiveElasticity | 0.45 | |
| LiveDistanceMin | 30 | VP units |
| LiveDistanceMax | 114 | VP units |

### Implementation

```vbscript
Sub CheckLiveCatch(ball, Flipper, FCount, parm)
    Dim CatchTime : CatchTime = GameTime - FCount

    If CatchTime <= LiveCatch And parm > 6 And _
       Abs(Flipper.x - ball.x) > LiveDistanceMin And _
       Abs(Flipper.x - ball.x) < LiveDistanceMax Then

        If CatchTime <= LiveCatch * 0.5 Then
            LiveCatchBounce = 0      ' perfect catch (first 8ms)
        Else
            LiveCatchBounce = Abs((LiveCatch / 2) - CatchTime)
        End If

        If LiveCatchBounce = 0 And ball.velx * Dir > 0 Then ball.velx = 0
        ball.vely = LiveCatchBounce * (32 / LiveCatch)
    End If
End Sub
```

### Behavior

- `parm` = collision force; must exceed 6 to trigger
- `FCount` = timestamp of flipper activation
- **Perfect catch** (0-8ms): Zeroes horizontal velocity completely
- **Partial catch** (8-16ms): Progressive bounce reduction based on timing precision
- Distance check ensures catch only works in the cradle zone (30-114 VP units from base), not at the tip

---

## Rubber COR Curves

Coefficient of Restitution (COR) as a function of ball speed. These define how "bouncy" rubber is at different impact velocities.

### RubbersD (Primary — Rubber Posts/Pegs)

| Speed (VP units/s) | COR `[ORIGINAL]` | COR `[UPDATED - VPW wiki 2025]` |
|---------------------|-------------------|-----------------------------------|
| 0 | 1.10 | **1.11** |
| 3.77 | 0.97 | **0.99** |
| 5.76 | 0.967 | **0.942** |
| 15.84 | 0.874 | 0.874 (unchanged) |
| 56 | 0.64 | 0.64 (unchanged) |

Key change: The updated curve has a steeper drop between speeds 3.77 and 5.76 (0.99 to 0.942 vs original 0.97 to 0.967), making medium-speed impacts slightly less bouncy while slow impacts are slightly more bouncy.

### SleevesD (Rubber Sleeves — 85% of RubbersD)

Calculated as 85% of the RubbersD curve.

| Speed (VP units/s) | COR `[ORIGINAL]` | COR `[UPDATED - VPW wiki 2025]` |
|---------------------|-------------------|-----------------------------------|
| 0 | 0.935 | **0.9435** |
| 3.77 | 0.8245 | **0.8415** |
| 5.76 | 0.82195 | **0.8007** |
| 15.84 | 0.7429 | 0.7429 (unchanged) |
| 56 | 0.544 | 0.544 (unchanged) |

### FlippersD (Flipper Elasticity Profile)

| Speed (VP units/s) | COR |
|---------------------|-----|
| 0 | 1.1 |
| 3.77 | 0.99 |
| 6 | 0.99 |

Note: Flipper COR curve is flat above speed 3.77. The flipper's own elasticity is further modified by the FlipperTricks FSM (SOSEM multiplier, EOSTnew, etc.).

### Best-Fit Parameters for VPE Implementation

The COR curves can be approximated using VPE's `elasticity_with_falloff` formula:

| Material | Elasticity | Falloff |
|----------|-----------|---------|
| Rubber Posts (best-fit) | 1.05 | 0.22 |
| Rubber Sleeves (best-fit) | 0.89 | 0.22 |

These fitted parameters applied via VPE's dampening-during-collision approach (not post-collision) produce the final COR directly.

---

## Material Presets

### Complete Material Table

| Material | Elasticity | Friction | Falloff | Scatter |
|----------|-----------|----------|---------|---------|
| Rubber Posts | 0.9 | 0.3 | 0.1 | 1 deg |
| Rubber Pegs | 0.9 | 0.3 | 0.1 | 1 deg |
| Rubber Sleeves | 0.765 | 0.3 | 0.1 | 1 deg |
| Rubber Bands | 0.85 | 0.3 | 0.13 | 0 deg |
| Metal | 0.4 | 0.15 - 0.2 | 0 | 0 deg |
| Metal Ramps | 0.4 | 0.15 | 0 | 0 deg |
| Metal w/ Falloff | 0.45 | 0.2 | 0.05 | 0 deg |
| Plastic Ramps | 0.4 | 0.3 | 0.2 | 0 deg |
| Plastics | 0.42 | 0.3 | 0 | 0 deg |
| Wood | 0.4 | 0.3 | 0.1 | 0 deg |
| Drop Targets | 0.625 | 0.3 | 0.1 | 1 deg |
| Spot Targets | 0.625 | 0.3 | 0 | 1 deg |
| Ramp Entry Protectors | 0.5 | 0.3 | 0.4 | 0 deg |
| Ramp End Vinyl | 0.7 | 0.3 | 0.4 | 0 deg |
| Gates | 0.85 | 0.8 | 0 | 0 deg |

### VPE ColliderComponent.cs Defaults (Authoritative)

Note: VPE has two layers of defaults. The `*Data.cs` files contain old VPX compatibility values (wrong defaults). The `*ColliderComponent.cs` files contain the Unity runtime values (authoritative). Always use ColliderComponent.cs values:

| Component | Elasticity | Friction | Scatter |
|-----------|-----------|----------|---------|
| Wall | 0.8 | 0.0 | 0.0 |
| Flipper | varies | varies | 0.0 |
| Bumper | 0.9 | 0.3 | 1.0 |
| All others | 0.0 scatter | | |

---

## Bumper Settings

| Property | Value | Notes |
|----------|-------|-------|
| Elasticity | 0.9 | VPE default |
| Hit Threshold | 1.6 - 2 | Range |
| Force | 9.5 - 10.5 | Range |
| Scatter Angle | 1 degree | |
| Friction | 0.3 | |

Note: nFozzy document lists bumper elasticity as 0.85 in one section and 0.9 in context. VPE ColliderComponent.cs default is 0.9.

---

## Slingshot Settings

| Property | Value | Notes |
|----------|-------|-------|
| Force | 5 | |
| Elasticity | 0.85 | |
| Hit Threshold | 2 - 3 | Range |
| Scatter Angle | 1 degree | |

---

## Drop Target Constants

| Constant | Value | Unit | Description |
|----------|-------|------|-------------|
| DTDropSpeed | 110 | ms | Time for target to drop |
| DTDropUpSpeed | 40 | ms | Time for target to raise |
| DTDropUnits | 44 | VP units | Drop distance |
| DTDropUpUnits | 10 | VP units | Raises above up position |
| DTMaxBend | 8 | degrees | Max rotation on hit |
| DTDropDelay | 20 | ms | Friction delay before drop |
| DTRaiseDelay | 40 | ms | Post-solenoid delay |
| DTBrickVel | 30 | VP units/s | Velocity threshold for bricking |
| DTEnableBrick | 0 | boolean | Disabled by default |
| DTMass | 0.2 | 0-1 range | Higher = more resistance |

### Brick Detection

```
If perpvel > DTBrickVel And perpvel > 0 And perpvelafter <= 0 Then
    ' target bricks (does not drop)
End If
```

Impact must be within 8 VP units of target center for brick check.

---

## Flipper Geometry Notes

### Standard 3-inch Flipper

> "A 3-inch flipper with rubbers will be about 3.125 inches long. This translates to about 147 VP units. Therefore, the flipper start radius + the flipper length + the flipper end radius should equal approximately 147 VP units."

### Conventions

- Right flipper angle must use **negative** value (e.g., -122 degrees, not +238 degrees)
- `47.06 VP units = 1 inch`

### Trigger Setup

- Triggers should surround flipper shape from start angle to end angle
- Trigger must be exactly **27 VP units** larger than the flipper objects (2024 standard, supersedes original 23 — old margin was less than ball radius)
- Hit height of triggers: **150 VP units**

### Endpoint Primitive

- **2x end radius** for XSize/YSize = 2

---

## LinearEnvelope Function

The core interpolation function used throughout the nFozzy system. Performs piecewise linear interpolation between keyframe pairs.

```vbscript
Function LinearEnvelope(xInput, xKeyFrame, yLvl)
    Dim L
    For ii = 1 To UBound(xKeyFrame)
        If xInput <= xKeyFrame(ii) Then L = ii : Exit For
    Next
    If xInput > xKeyFrame(UBound(xKeyFrame)) Then L = UBound(xKeyFrame)

    Y = pSlope(xInput, xKeyFrame(L-1), yLvl(L-1), xKeyFrame(L), yLvl(L))

    If xInput <= xKeyFrame(LBound(xKeyFrame)) Then Y = yLvl(LBound(xKeyFrame))
    If xInput >= xKeyFrame(UBound(xKeyFrame)) Then Y = yLvl(UBound(xKeyFrame))

    LinearEnvelope = Y
End Function
```

- `xInput`: the value to look up (e.g., ball speed, ball position)
- `xKeyFrame`: array of breakpoint X values
- `yLvl`: array of corresponding Y values
- Returns: linearly interpolated Y value
- Clamps to first/last Y value outside the keyframe range

---

## Lane Guide & Technique Notes

### Post Pass

> "Geometry can be very important when designing a table for flipper tricks. For example, the position of lane guides in relation to the flipper can have significant impacts on the ability to post pass. If the lane guides are too high, the ball may deflect off the guide preventing it from achieving the necessary trajectory."

### Techniques Enabled by nFozzy

The FlipperTricks + FlipperCorrection system enables these real-world techniques in simulation:

- **Post pass**: Ball wraps around the flipper tip post to the opposite flipper
- **Drop catch**: Ball decelerates on a dropping flipper (uses EOSReturn)
- **Tap pass**: Quick flipper tap sends ball across to the other flipper
- **Dead bounce / dead flip**: Ball bounces off a held flipper without active input
- **Live catch**: Timed button press absorbs ball momentum (LiveCatch window)
- **Alley pass**: Ball passes through the inlane to the opposite flipper

---

## Coordinate System Reference

| System | Origin | X | Y | Z |
|--------|--------|---|---|---|
| VP/VPX Internal | Top-left | Right | Down | Toward player |
| Units | | 47.06/inch | 47.06/inch | 47.06/inch |

- Ball diameter: 50 VP units = ~1.0625 inches (standard 1-1/16" ball)
- Standard playfield: ~954 x 2094 VP units (20.25" x 44.5")

---

## Summary of All Updated Values [VPW wiki 2025]

For quick reference, here are all values that differ from the original nFozzy document:

| Parameter | Original | Updated | Source |
|-----------|----------|---------|--------|
| Rubber COR @ speed 0 | 1.10 | **1.11** | VPW wiki 2025 |
| Rubber COR @ speed 3.77 | 0.97 | **0.99** | VPW wiki 2025 |
| Rubber COR @ speed 5.76 | 0.967 | **0.942** | VPW wiki 2025 |
| EM Return Strength | 0.055 | **0.048** | VPW wiki 2025 |
| 90s+ Return Strength | 0.025 | **0.018** | VPW wiki 2025 |
| EOSTNew formula | era lookup | `Strength * (-0.00025) + 1.57` | VPE PR #443 |
| EOSReturn formula | era lookup | `Strength * (-0.0000119) + 0.06348` | VPE PR #443 |
