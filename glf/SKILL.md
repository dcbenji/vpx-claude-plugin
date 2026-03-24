---
name: vpx-dev-glf
description: "VPX Game Logic Framework GLF original table mode event shot multiball scoring ball save tilt light show sound display Flux mpcarr Apophis"
version: "1.0.0"
last_updated: "2026-03-24"
---

# VPX Game Logic Framework (GLF)

## When This Skill Applies

Use when building an **original (non-ROM) VPX table** that needs structured game logic: modes, multiball, scoring, ball save, tilt, shot tracking, lighting, sounds, displays. GLF replaces hand-coded state machines with a configuration-driven, event-based framework.

**NOT for recreations** — ROM tables use PinMAME + the VPW stack (see pinmame-wiring skill).

**GLF coexists with nFozzy/Fleep** — GLF handles game logic; nFozzy handles flipper physics and rubber separation; Fleep handles mechanical sounds. You need all three for a complete table.

## What GLF Is

GLF (by mpcarr/Flux) is a VBScript framework inspired by the Mission Pinball Framework (MPF). Instead of writing raw `_Hit` subs and managing state with Dim variables, you configure **devices** and **modes** that communicate through **events**. GLF handles:

- Ball management (trough, scoops, VUKs, locks)
- Flippers, bumpers, slingshots, drop targets (as managed devices)
- Game modes with priority, start/stop events, and scoped components
- Multiball with shoot-again, grace period, add-a-ball
- Shots with profiles and state progression
- Ball save, tilt, high scores
- Scoring via variable player
- Lights, sounds, shows, displays — all event-driven

**Source:** [github.com/mpcarr/vpx-glf](https://github.com/mpcarr/vpx-glf) (MIT license)
**Showcase table:** Dark Chaos by Apophis (2025)
**Example table repo:** [github.com/mpcarr/vpx-example-glf](https://github.com/mpcarr/vpx-example-glf)

## Setup Requirements

### 1. Include the script
Download `vpx-glf.vbs` and paste/include it in your table script (before your game code).

### 2. VPX objects required
- **Timer:** `Glf_GameTimer` — Enabled, Interval = -1
- **Collections** (create via F8 in VPX):
  - `glf_lights` — all VPX light objects
  - `glf_switches` — all switch triggers/targets
  - `glf_slingshots` — slingshot walls
  - `glf_spinners` — spinner objects

### 3. Global constants
```vbs
Const cGameName = "MyGame"
Const BallSize = 50
Const BallMass = 1
Const tnob = 6           ' total balls in trough
Const lob = 0            ' non-playable balls (captive balls)
Dim gBot
Dim tablewidth: tablewidth = Table1.width
Dim tableheight: tableheight = Table1.height
```

### 4. Hook into table events
```vbs
Sub Table1_Init()
    ConfigureGlfDevices
    Glf_Init(Table1)
End Sub

Sub Table1_Exit()
    Glf_Exit
End Sub

Sub Table1_KeyDown(ByVal keycode)
    Glf_KeyDown keycode
End Sub

Sub Table1_KeyUp(ByVal keycode)
    Glf_KeyUp keycode
End Sub

Sub Table1_OptionEvent(ByVal eventId)
    If eventId = 1 Then DisableStaticPreRendering = True
    Glf_Options eventId
    If eventId = 3 Then DisableStaticPreRendering = False
End Sub
```

### 5. ConfigureGlfDevices sub
All device and mode configuration goes here:
```vbs
Sub ConfigureGlfDevices
    ' Trough, flippers, devices, modes — all configured here
End Sub
```

## Core Concepts

### Events
Everything communicates through named events. Devices emit events; modes and components listen.

```vbs
' Listen for an event
AddPinEventListener "sw01_active", "myKey", "HandleSwitch", 1000, Null

Sub HandleSwitch(args)
    Dim listenerArgs: listenerArgs = args(0)
    Dim dispatchArgs: dispatchArgs = args(1)
End Sub

' Fire an event
DispatchPinEvent "sw01_active", Null

' Remove a listener
RemovePinEventListener "sw01_active", "myKey"
```

**Built-in events from switches:** When a VPX object in the `glf_switches` collection is hit, GLF auto-dispatches `{switchname}_active` and `{switchname}_inactive` events.

### Devices
Physical table mechanisms configured with `CreateGlf*` factory functions. Each device maps VPX objects to GLF's event system and provides callbacks for the physical actions you implement.

### Modes
Containers for game logic. A mode has a priority, start/stop events, and scoped components (shots, timers, counters, lights, sounds, etc.). When a mode stops, all its components are automatically cleaned up.

### Callbacks
GLF tells you WHEN something should happen (via callbacks). You write the HOW (the actual VPX kick/sound/animation code). This separation means GLF doesn't care about your specific table layout.

## Device Reference

### Trough (Ball Device)
```vbs
With CreateGlfBallDevice("trough")
    .BallSwitches = Array("s_trough1", "s_trough2", "s_trough3")
    .EjectCallback = "TroughEjectCallback"
    .EjectTimeout = 2000
    .DefaultDevice = True
End With

Sub TroughEjectCallback(ball)
    BallRelease.Kick 90, 20
End Sub
```

### Scoop / VUK (Ball Device)
```vbs
With CreateGlfBallDevice("scoop")
    .BallSwitches = Array("s_scoop1")
    .EjectCallback = "ScoopEjectCallback"
    .EjectTimeout = 2000
End With

Sub ScoopEjectCallback(ball)
    Scoop1.Kick 270, 30
End Sub
```

**Events emitted:** `balldevice_{name}_ball_entered`, `balldevice_{name}_ball_eject_success`

### Flippers
```vbs
With CreateGlfFlipper("left")
    .Switch = "s_left_flipper"
    .ActionCallback = "LeftFlipperAction"
End With

Sub LeftFlipperAction(enabled)
    If enabled Then
        LeftFlipper.RotateToEnd
        LF.Fire    ' nFozzy
    Else
        LeftFlipper.RotateToStart
        LF.UnFire
    End If
End Sub
```

**Default behavior:** Enabled on `ball_started`, disabled on `ball_will_end`.

### Auto-Fire Devices (Bumpers, Slingshots)
```vbs
With CreateGlfAutoFireDevice("left_sling")
    .Switch = "s_LeftSlingshot"
    .ActionCallback = "LeftSlingshotAction"
End With

Sub LeftSlingshotAction(enabled)
    If enabled Then
        PlaySoundAtVol "sling", LeftSlingshot, VolumeDial
    End If
End Sub
```

### Drop Targets
```vbs
With CreateGlfDropTarget("target1")
    .Switch = "s_target1"
    .ActionCallback = "Target1Action"
    .ResetEvents = Array("reset_targets")
End With

Sub Target1Action(state)
    If state = 1 Then
        ' Target dropped
        DropTarget1.IsDropped = True
    ElseIf state = 0 Then
        ' Target reset
        DropTarget1.IsDropped = False
    End If
End Sub
```

**Events emitted:** `droptarget_{name}_hit`, `droptarget_{name}_reset`

## Mode System

Modes are the core organizational unit. Everything game-logic related lives inside a mode.

### Basic Mode
```vbs
With CreateGlfMode("base", 10)
    .StartEvents = Array("ball_started")
    .StopEvents = Array("ball_ended")

    ' --- Scoring ---
    With .VariablePlayer
        With .EventName("sw_target1_active")
            With .Variable("score")
                .Action = "add"
                .Int = 1000
            End With
        End With
        With .EventName("sw_bumper1_active")
            With .Variable("score")
                .Action = "add"
                .Int = 100
            End With
        End With
    End With
End With
```

### Mode with Shots, Counter, Timer, Lights, Sounds
```vbs
With CreateGlfMode("ramp_challenge", 20)
    .StartEvents = Array("ramp_challenge_start")
    .StopEvents = Array("ramp_challenge_complete", "ball_ended")

    ' Track ramp completions
    With .Counters("ramp_count")
        .CountEvents = Array("sw_left_ramp_active", "sw_right_ramp_active")
        .CountCompleteValue = 5
        .EventsWhenComplete = Array("ramp_challenge_complete")
        .PersistState = False
    End With

    ' 30-second countdown
    With .Timers("mode_timer")
        .StartRunning = True
        .Direction = "down"
        .StartValue = 30
        .EndValue = 0
        .TickInterval = 1000
    End With

    ' Scoring
    With .VariablePlayer
        With .EventName("sw_left_ramp_active")
            With .Variable("score")
                .Action = "add"
                .Int = 25000
            End With
        End With
    End With

    ' Lights
    With .LightPlayer
        With .EventName("mode_ramp_challenge_started")
            With .Lights("left_ramp_arrow")
                .Color = "00ff00"
                .Fade = 300
            End With
        End With
    End With

    ' Sounds
    With .SoundPlayer
        With .EventName("mode_ramp_challenge_started")
            .Sound = "mode_start_callout"
            .Action = "play"
        End With
    End With

    ' Chain events
    With .EventPlayer
        .Add "ramp_challenge_complete", Array("start_victory_show", "award_extra_ball")
    End With
End With
```

## Multiball
```vbs
With CreateGlfMode("multiball_mode", 30)
    .StartEvents = Array("multiball_ready")
    .StopEvents = Array("multiball_ended")

    With .Multiballs("main_multiball")
        .BallCount = 3
        .BallCountType = "total"
        .StartEvents = Array("mode_multiball_mode_started")
        .ShootAgain = 15000
        .GracePeriod = 2000
    End With

    With .BallSaves("mb_save")
        .ActiveTime = 10000
        .EnableEvents = Array("multiball_main_multiball_started")
        .AutoLaunch = True
        .BallsToSave = 3
    End With
End With
```

**Events emitted:** `multiball_{name}_started`, `multiball_{name}_ended`, `multiball_{name}_shoot_again`, `multiball_{name}_ball_lost`

## Ball Save
```vbs
With .BallSaves("new_ball")
    .ActiveTime = 6000
    .GracePeriod = 2000
    .HurryUpTime = 3000
    .EnableEvents = Array("ball_started")
    .AutoLaunch = True
    .BallsToSave = 1
End With
```

**Events emitted:** `ball_save_{name}_enabled`, `ball_save_{name}_saving_ball`, `ball_save_{name}_hurry_up`

## Tilt
```vbs
With .Tilt
    .WarningsToTilt = 3
    .SettleTime = 5000
    .MultipleHitWindow = 1000
    .TiltWarningEvents = Array("sw_tilt_active")
    .TiltSlamTiltEvents = Array("sw_slam_tilt_active")
End With
```

**Events emitted:** `tilt_warning`, `tilt`, `slam_tilt`, `tilt_clear`

## Player State & Scoring

```vbs
' Set/get player variables
SetPlayerState "current_mode", "base"
Dim mode: mode = GetPlayerState("current_mode")

' Monitor score changes (e.g., update DMD)
AddPlayerStateEventListener "score", "score_display", 0, "UpdateScoreDisplay", 1000, Null

Sub UpdateScoreDisplay(args)
    Dim score: score = GetPlayerState("score")
    DMDScore.Text = FormatNumber(score, 0)
End Sub
```

## High Scores
```vbs
With EnableGlfHighScores()
    With .Categories()
        .Add "score", Array("GRAND CHAMPION", "HIGH SCORE 1", "HIGH SCORE 2", "HIGH SCORE 3")
    End With
    With .Defaults("score")
        .Add "FLX", 9000000
        .Add "APO", 7000000
        .Add "VPX", 5000000
        .Add "GLF", 3000000
    End With
    .EnterInitialsTimeout = 65000
End With
```

## Key Events Reference

### Game Lifecycle
| Event | When |
|-------|------|
| `game_starting` | New game begins |
| `game_started` | Game fully initialized |
| `game_ending` | Game ending |
| `game_ended` | Game fully ended |
| `ball_starting` | Ball about to start |
| `ball_started` | Ball in play |
| `ball_will_end` | Ball about to end |
| `ball_ended` | Ball fully ended |
| `player_added` | New player added |

### Switch Events (auto-generated)
For every object in `glf_switches`:
- `{switch_name}_active` — ball hits
- `{switch_name}_inactive` — ball exits (triggers only)

### Device Events
| Device | Event Pattern |
|--------|---------------|
| Ball device | `balldevice_{name}_ball_entered`, `balldevice_{name}_ball_eject_success` |
| Flipper | `flipper_{name}_activated`, `flipper_{name}_deactivated` |
| Drop target | `droptarget_{name}_hit`, `droptarget_{name}_reset` |
| Auto-fire | `auto_fire_coil_{name}_activate` |

### Mode Events
| Event | When |
|-------|------|
| `mode_{name}_started` | Mode started |
| `mode_{name}_stopped` | Mode stopped |

### Component Events
| Component | Event Pattern |
|-----------|---------------|
| Multiball | `multiball_{name}_started`, `multiball_{name}_ended`, `multiball_{name}_ball_lost` |
| Ball save | `ball_save_{name}_enabled`, `ball_save_{name}_saving_ball`, `ball_save_{name}_hurry_up` |
| Timer | `timer_{name}_started`, `timer_{name}_tick`, `timer_{name}_complete` |
| Counter | `counter_{name}_complete` |
| Shot | `{name}_hit`, `{name}_{profile}_hit`, `{name}_{state}_hit` |
| Tilt | `tilt_warning`, `tilt`, `slam_tilt`, `tilt_clear` |

## GLF vs Hand-Coded Comparison

**Without GLF** (hand-coded):
```vbs
Dim ModeActive, ModeTimer, RampCount
Sub StartRampMode()
    ModeActive = True
    ModeTimer = 30
    RampCount = 0
End Sub
Sub LeftRamp_Hit()
    If Not ModeActive Then Exit Sub
    RampCount = RampCount + 1
    Score = Score + 25000
    If RampCount >= 5 Then CompleteRampMode
End Sub
Sub GameTimer_Timer()
    If ModeActive Then
        ModeTimer = ModeTimer - 0.01
        If ModeTimer <= 0 Then ModeActive = False
    End If
End Sub
```

**With GLF** (configured):
```vbs
With CreateGlfMode("ramp_challenge", 20)
    .StartEvents = Array("ramp_challenge_start")
    .StopEvents = Array("ramp_challenge_complete", "timer_mode_timer_complete")
    With .Counters("ramp_count")
        .CountEvents = Array("sw_left_ramp_active")
        .CountCompleteValue = 5
        .EventsWhenComplete = Array("ramp_challenge_complete")
    End With
    With .Timers("mode_timer")
        .StartRunning = True
        .Direction = "down"
        .StartValue = 30
        .EndValue = 0
        .TickInterval = 1000
    End With
    With .VariablePlayer
        With .EventName("sw_left_ramp_active")
            With .Variable("score")
                .Action = "add"
                .Int = 25000
            End With
        End With
    End With
End With
```

The GLF version is more verbose but: the mode auto-starts/stops, the counter auto-resets, the timer auto-cleans-up, player state is auto-tracked per player, and nothing needs manual state management.

## Integration with nFozzy + Fleep

GLF handles game logic only. You still need:
- **nFozzy physics** — flipper tuning, rubber separation, target bouncers (see nfozzy-physics skill)
- **Fleep sounds** — mechanical sounds, ball rolling (see sound skill)
- **Standard VPX timers** — GameTimer with `Cor.Update`, FrameTimer for animations (see conventions skill)

The flipper callback is where these meet:
```vbs
Sub LeftFlipperAction(enabled)
    If enabled Then
        LeftFlipper.RotateToEnd
        LF.Fire              ' nFozzy
    Else
        LeftFlipper.RotateToStart
        LF.UnFire            ' nFozzy
    End If
End Sub
```

## Anti-Patterns

### Mixing GLF and hand-coded state for the same feature
**Wrong:** Using a GLF counter AND a manual `Dim count` variable for the same target bank.
**Fix:** Pick one approach. GLF counters emit events on completion — listen for those.

### Forgetting glf_switches collection
**Wrong:** GLF switch events never fire.
**Fix:** Every VPX trigger/target that should generate events must be in the `glf_switches` collection with "Fire events for this collection" checked.

### Not hooking Glf_KeyDown/Glf_KeyUp
**Wrong:** Flippers don't respond to input.
**Fix:** GLF needs the key events forwarded. Add `Glf_KeyDown keycode` / `Glf_KeyUp keycode` to your table key subs.

### Putting VPX physics code in GLF callbacks
**Wrong:** Trying to handle `Cor.Update` or nFozzy inside GLF device callbacks.
**Fix:** Keep the standard GameTimer/FrameTimer architecture for physics. GLF callbacks are for game logic actions (kick, sound, light), not physics updates.
