---
name: vpx-dev-nfozzy-physics
description: "VPX VBScript pinball physics nFozzy flipper rubber bumper slingshot target elasticity friction material tuning cradle polarity velocity curve trigger separation dPosts dSleeves bands TargetBouncer FlipperCradleCollision era WPC Stern"
version: "1.0.0"
last_updated: "2026-03-24"
---

# nFozzy Physics Configuration

## When This Skill Applies
Use when configuring flipper physics, rubber separation, target bounce, slingshot correction, or any physics material tuning on a VPX table. Also applies when diagnosing flipper feel, ball behavior, or physics-related bugs.

## Where to Get the Code

All nFozzy code comes from the **VPW Example Table**:
- **Download:** VPUniverse (search "VPW Example Table")
- **Extracted source:** `github.com/francisdb/vpw-example-table-extracted` — use vpxtool to extract, or read the script directly from the repo
- **What to copy:** The entire nFozzy physics section as one block (CorTracker class, polarity curves, velocity curves, FlipperTricks, CheckLiveCatch, trajectory correction). Paste before your game code.

**Fleep mechanical sounds** (separate from nFozzy but typically added together):
- **Sound functions VBS:** Dropbox link in VPW Discord (Fleep Sound Functions updated)
- **Sound samples by era:** Dropbox link in VPW Discord (Fleep Sounds folder)
- **Tutorial:** "Adding Fleep Sound Package" by rothbauerw

**v1.7+ "Basic" Example Table:** A stripped-down variant with just nFozzy + Fleep + generic layout — proper starting point for new tables (the full Example Table is too complex for beginners).

## 9-Step Implementation Checklist

1. **Make flipper meshes** -- visual flipper primitives matching era dimensions
2. **Make RDampen timer** -- (legacy reference; no longer needed in current implementations)
3. **Paste Key down/up sub code** -- flipper input handling
4. **Add LF Fire / RF Fire** -- to flipper key subs
5. **Paste all nFozzy code as one chunk** -- polarity curves, velocity curves, FlipperTricks, live catch, trajectory correction
6. **Check for duplicate flipper Collide subs** -- VBScript silently overrides duplicates (last definition wins)
7. **Sort out rubbers** -- create dPosts, dSleeves collections, assign physics materials
8. **Import era-correct .vpp** -- in Table > Physics Options. Re-import if material dropdowns are empty
9. **Test** -- F11 dotted ball: no abnormal inlane spin. If ball reverses on flipper contact, re-import .vpp

## Flipper Tuning by Era

| Era | Strength | SOSRampup | EOSTorque | EOSReturn | Notes |
|-----|----------|-----------|-----------|-----------|-------|
| System 11 (mid-80s) | 2300-2600 | 2.5 | 0.275 | 0.35 | Weaker solenoids |
| Early WPC (1991) | 2400 | 2.5 | 0.275 | 0.35 | Red coil FL-11630 |
| Mid WPC (1992+) | 2600 | 2.5 | 0.275 | 0.35 | Blue coil FL-11629 |
| Late WPC / strong coil | 2800-3500 | 2.5 | 0.375 | 0.4 | Indy, No Fear |
| Modern Stern | 3000-3500 | 2.5 | 0.375 | 0.4 | Higher EOS values |

- EOSTorque 0.375 + EOSReturn 0.4 together improve flipper trick realism (tap pass, flick pass)
- SOSRampup must stay at 2.5 -- higher values (8.5) enable tap passes but break flipper timing
- Upper flippers: typically 2400-2500, lower than main flippers

Select the **AddPt polarity/velocity curve set** matching the table's era. Curves are NOT universal. When shot angles feel wrong (too much backhand), try a different era's polarity curve.

## Flipper Trigger Sizing

```
Trigger margin:     27 VP units from flipper (2024 standard, supersedes 23)
End angle sizing:   Add 3 degrees beyond actual end angle, create trigger, restore angle
Hit height:         150
PolarityCorrect:    Must be in drain_hit sub
```

**Why 27:** The old 23-unit margin was less than a ball radius (25 units) -- velocity correction from cradle was silently broken.

```vbs
' drain_hit must include:
Sub drain_hit()
    RF.PolarityCorrect Activeball: LF.PolarityCorrect Activeball
    ' ... drain logic
End Sub
```

Without PolarityCorrect in drain_hit, multiball drain causes flipper trigger unhit events to not fire, producing recurring division-by-zero errors in COR tracking.

## Upper Flipper Simplification

- **Disable trajectory correction** -- the correction adjusts X-velocity relative to the table, not the flipper surface. Works for horizontal main flippers, pushes ball wrong direction for angled upper flippers
- **Keep flipper tricks** -- still needed for realistic upper flipper feel
- Remove the correction call from the upper flipper's solenoid callback only

## Rubber Separation

### Three Components

| Type | Collection | Material | Elasticity | Falloff | Friction | Scatter |
|------|-----------|----------|------------|---------|----------|---------|
| Posts (bare) | dPosts | z_col_rubberposts | 0.9 | 0.1 | 0.2 | 0 |
| Sleeves | dSleeves | z_col_rubberpostsleeves | 0.765 | 0.1 | 0.2 | 0 |
| Bands | (none) | z_col_rubberbands | 0.85 | 0.1 | 0.2 | 0 |

- **dPosts**: invisible collidable cylinders where rubber wraps around bare posts. Collection applies dampening on contact
- **dSleeves**: same as dPosts but with more damping (less bouncy). Sleeves should be ~20x20x20 VP units (real sleeve measurement). Oversized sleeves make the table too hard
- **Bands**: invisible collidable rectangles where rubber stretches between posts. NOT in any dampening collection
- Original visible rubbers stay visible but set **non-collidable**
- The pre-existing `Rubbers` collection (hit sounds) stays as-is -- dPosts/dSleeves are additional

### 5-Object Rubber Band Rigging

Each rubber band assembly requires 5 objects:
1. Post primitive (left end)
2. Post primitive (right end)
3. Rubber band wall (left side)
4. Rubber band wall (right side)
5. Middle section wall (rubber between posts)

Short rubber material should use values between post elasticity and standard band elasticity.

## TargetBouncer

Adds realistic vertical bounce to standup target hits. Redistributes ball velocity into a Z component.

```vbs
Const TargetBouncerEnabled = 1
Const TargetBouncerFactor = 0.7

' Add targets/posts to a "TargetBounce" collection
Sub TargetBounce_Hit(idx)
    TargetBouncer activeball, 2
End Sub
```

**Critical: the `(idx)` parameter is REQUIRED.** Without it, the sub signature does not match and the bounce code silently fails -- no error message, no bounce.

- Random multiplier: 0.2-0.5 (varies bounce height naturally)
- Only fires when `ball.z < 30` (on playfield)
- Factor 0.7 intentionally adds some energy (debated but accepted as standard)

### Exclusion Zones
Do NOT apply TargetBouncer to objects near:
- **Pop bumpers** -- repeated velocity additions between bumper kicks and target bounces compound, launching ball through geometry
- **Scoops** -- balls bounce upward instead of rolling into the scoop

## Slingshot Correction

VPX slings apply force perpendicular to the wall surface. Real slings deflect at an angle. Add 4-7 degrees of rotational correction.

```vbs
' Place in _Slingshot event subs, NOT SolCallback
' (SolCallback has built-in cooldown that suppresses rapid re-fires)
Sub LeftSlingshot_Slingshot
    dim ball, angle, origVelx, origVely
    set ball = activeball
    angle = 4 ' degrees, range 4-7
    origVelx = ball.velx
    origVely = ball.vely
    ball.velx = origVelx + dcos(angle) * origVelx - dsin(angle) * origVely
    ball.vely = origVely + dsin(angle) * origVelx + dcos(angle) * origVely
End Sub
```

Requires `dsin` and `dcos` helper functions in the script:
```vbs
Function dsin(degrees) : dsin = sin(degrees * 3.14159265 / 180) : End Function
Function dcos(degrees) : dcos = cos(degrees * 3.14159265 / 180) : End Function
```

**Threshold tuning:** Both sling threshold AND hit threshold must match (1.5-2.0 sweet spot). If they differ, animation/sounds trigger without physics or vice versa.

## FlipperCradleCollision

During multiball, ball-to-ball collisions on a held flipper produce unrealistically high velocities. Apply COR dampening when either ball is on a flipper held at end angle.

```vbs
Sub OnBallBallCollision(ball1, ball2, velocity)
    Dim DesiredBallCOR
    DesiredBallCOR = 0.4
    If FlipperTrigger(ball1.x, ball1.y, LeftFlipper) Or _
       FlipperTrigger(ball1.x, ball1.y, RightFlipper) Or _
       FlipperTrigger(ball2.x, ball2.y, LeftFlipper) Or _
       FlipperTrigger(ball2.x, ball2.y, RightFlipper) Then
        ball1.velx = ball1.velx * DesiredBallCOR
        ball1.vely = ball1.vely * DesiredBallCOR
        ball1.velz = ball1.velz * DesiredBallCOR
        ball2.velx = ball2.velx * DesiredBallCOR
        ball2.vely = ball2.vely * DesiredBallCOR
        ball2.velz = ball2.velz * DesiredBallCOR
    End If
End Sub
```

All three velocity components (x, y, z) of both balls must be dampened.

## Global Settings

| Setting | Value | Why |
|---------|-------|-----|
| Difficulty | 50-56 | NOT default 20. All tuning at wrong difficulty will be wrong |
| Ramp exit spin multiplier | 50 | Default 70 causes corkscrew exits. Global setting, affects all ramps |
| Ball mass | 1 | Always. Even widebody tables. Mass != 1 breaks all flipper physics |
| BallSize | 50 | Not legacy 25 |
| Playfield friction | 0.15-0.25 | Default 0.02 is far too low |
| Min friction (any surface) | 0.1 | Zero friction causes unpredictable behavior |
| Bumper strength | 13-15 | Default is too weak. Radius = rubber skirt, not cap |

## Anti-Patterns

### Global Physics Sets
**Never use with nFozzy.** Physics sets silently override nFozzy's tuned values. The editor does NOT grey out overridden values -- the conflict is invisible. Remove all GPS before nFozzy tuning.

### Ball Mass != 1
"Ball mass of 1.3 will throw all of the flipper stuff off." The real ball is the same weight on all tables. If mass 1 doesn't feel right, the issue is elsewhere (flipper angles, ramp friction, geometry).

### Zeroed Physics Materials
Certain VPX operations silently zero out imported .vpp material values. **Detection:** F11 dotted ball shows abnormal inlane spin; ball reverses direction touching flipper. **Fix:** Re-import the .vpp file.

### Friction = 0
Never on any surface. Use 0.1 minimum. Zero friction causes balls to slide without rolling and breaks physics calculations.

### Ramp Friction Too High
Friction 0.8 = "velcro ramp." Unify ramp physics with a single material around 0.2 friction.

**Source:** VPW physics-tuning guide, VPW Example Table, VPW best-practices guide
