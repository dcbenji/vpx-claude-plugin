---
name: vpx-dev-core-vbs
description: "VPX core.vbs API reference cvpmTrough cvpmSaucer cvpmDropTarget cvpmMagnet cvpmMech cvpmFlips2 cvpmImpulseP cvpmVLock SolCallback loadvpm vpmInit"
version: "1.0.0"
last_updated: "2026-03-24"
---

# core.vbs API Reference

## When This Skill Applies

Use when a table script calls functions/classes from `core.vbs` — the VPinMAME driver script that ships with VPX. This is the foundation layer that both VPW tables and GLF tables build on. Current version: 3.61+.

**Source:** `vpinball/scripts/core.vbs` in the VPX installation or repo.

## Initialization

```vbs
' In Table1_Init (ROM tables):
LoadVPM "02700100", "wpc.vbs", 3.57   ' minVersion, romVBS, requiredVer
Set Controller = CreateObject("VPinMAME.Controller")
Controller.GameName = cGameName
Controller.Run

' Then:
vpmInit Table1      ' initializes timer system, flippers, etc.

' In Table1_Exit:
vpmExit             ' cleanup
```

**For original tables:** You still use core.vbs classes (trough, saucer, etc.) but skip the PinMAME controller setup.

---

## cvpmTrough — Ball Trough

Manages ball storage, entry, and exit. The most important class for any table.

### Setup
```vbs
Dim bsTrough
Set bsTrough = New cvpmTrough
bsTrough.Size = 4                                    ' number of slots
bsTrough.InitSwitches Array(swTrough1, swTrough2, swTrough3, swTrough4)
bsTrough.InitExit BallRelease, 90, 5                ' kicker, angle, force
bsTrough.InitExitVariance 3, 2                       ' angle variance, force variance
bsTrough.CreateEvents "bsTrough", Drain              ' name, entry kicker(s)
bsTrough.InitExitSounds "BallRelease", ""
bsTrough.InitEntrySounds "fx_drain", "", ""
bsTrough.Balls = 4                                   ' starting ball count
bsTrough.IsTrough = True                             ' mark as default trough

' Wire solenoids:
SolCallback(sReleaseBall) = "bsTrough.solOut"
SolCallback(sAutoPlunger) = "bsTrough.solOut"
```

### Key Properties/Methods
| Member | Type | Description |
|--------|------|-------------|
| `.Size = N` | Property | Set trough capacity |
| `.InitSwitches(array)` | Sub | Assign switch numbers to slots |
| `.InitExit(kicker, angle, force)` | Sub | Configure exit kicker |
| `.InitExitVariance(angleVar, forceVar)` | Sub | Add kick randomness |
| `.CreateEvents(name, kicker)` | Sub | Auto-wire entry kicker hit events |
| `.Balls = N` | Property | Set/get ball count |
| `.BallsPending` | Property (get) | Balls queued at entrance |
| `.IsTrough = True` | Property | Mark as default trough for orphaned balls |
| `.MaxBallsPerKick = N` | Property | Balls ejected per solOut pulse |
| `.solOut(enabled)` | Sub | Exit solenoid callback |
| `.solIn(enabled)` | Sub | Entry solenoid callback |
| `.AddBall(kicker)` | Sub | Manually add a ball |
| `.EntrySw = swNo` | Property | Entry switch (0 = auto-enter) |

---

## cvpmSaucer — Single Ball Kicker (Scoop/VUK)

Holds one ball, ejects on solenoid command.

### Setup
```vbs
Dim bsScoop
Set bsScoop = New cvpmSaucer
bsScoop.InitKicker Scoop1, swScoop, 270, 30, 5      ' kicker, switch, angle, force, zForce
bsScoop.InitExitVariance 3, 2
bsScoop.InitAltKick 90, 20, 0                        ' alternate exit direction
bsScoop.CreateEvents "bsScoop", Scoop1
bsScoop.InitSounds "fx_kicker", "", ""

SolCallback(sScoopEject) = "bsScoop.solOut"
SolCallback(sScoopAlt) = "bsScoop.solOutAlt"
```

### Key Properties/Methods
| Member | Type | Description |
|--------|------|-------------|
| `.InitKicker(kicker, sw, angle, force, zForce)` | Sub | Configure saucer |
| `.InitAltKick(angle, force, zForce)` | Sub | Alternate exit direction |
| `.InitExitVariance(angleVar, forceVar)` | Sub | Add randomness |
| `.CreateEvents(name, kicker)` | Sub | Auto-wire kicker hit events |
| `.HasBall` | Property (get) | True if ball is captured |
| `.solOut(enabled)` | Sub | Primary exit solenoid |
| `.solOutAlt(enabled)` | Sub | Alternate exit solenoid |
| `.AddBall(kicker)` | Sub | Manually add ball |

---

## cvpmDropTarget — Drop Target Bank

Manages a bank of drop targets with individual/group control.

### Setup
```vbs
Dim dtBank
Set dtBank = New cvpmDropTarget
dtBank.InitDrop Array(dt1, dt2, dt3), Array(swDT1, swDT2, swDT3)
dtBank.CreateEvents "dtBank"
dtBank.InitSnd "fx_droptarget", "fx_resetdrop"
dtBank.AllDownSw = swAllDown                          ' optional: switch when all down
dtBank.AnyUpSw = swAnyUp                              ' optional: switch when any up

SolCallback(sResetDT) = "dtBank.SolDropUp"
```

### Key Properties/Methods
| Member | Type | Description |
|--------|------|-------------|
| `.InitDrop(walls, switches)` | Sub | Configure targets and switches |
| `.CreateEvents(name)` | Sub | Auto-wire hit/drop events |
| `.InitSnd(dropSound, raiseSound)` | Sub | Set sounds |
| `.AllDown` | Property (get) | True when all targets dropped |
| `.AllDownSw = swNo` | Property | Switch for all-down state |
| `.AnyUpSw = swNo` | Property | Switch for any-up state |
| `.Hit(targetNo)` | Sub | Drop target N (1-based) |
| `.SolDropUp(enabled)` | Sub | Reset all targets (solenoid callback) |
| `.SolDropDown(enabled)` | Sub | Drop all targets |
| `.SolHit(targetNo, enabled)` | Sub | Drop single target (solenoid) |
| `.SolUnhit(targetNo, enabled)` | Sub | Raise single target (solenoid) |
| `.LinkedTo = otherBank` | Property | Link banks (one drops, all drop) |

---

## cvpmMagnet — Magnet

Attracts balls within a trigger zone.

### Setup
```vbs
Dim Magnet1
Set Magnet1 = New cvpmMagnet
Magnet1.InitMagnet MagnetTrigger, 15                  ' trigger, strength
Magnet1.GrabCenter = True                             ' snap to center when close
Magnet1.CreateEvents "Magnet1"

SolCallback(sMagnet) = "Magnet1.MagnetOn ="
```

### Key Properties/Methods
| Member | Type | Description |
|--------|------|-------------|
| `.InitMagnet(trigger, strength)` | Sub | Configure magnet |
| `.MagnetOn = True/False` | Property | Enable/disable |
| `.GrabCenter = True/False` | Property | Snap ball to center |
| `.Strength = N` | Property | Magnet power (0-20, >14 = grab) |
| `.Size = N` | Property | Radius of effect |
| `.CreateEvents(name)` | Sub | Auto-wire trigger events |

---

## cvpmMech — Stepper Motor / Mechanism

For ROM-controlled stepper motors (drawbridges, carousels, goalies).

### Setup
```vbs
Dim Mech
Set Mech = New cvpmMech
With Mech
    .Sol1 = 12                    ' direction solenoid 1
    .Sol2 = 13                    ' direction solenoid 2
    .MType = vpmMechLinear + vpmMechTwoDirSol + vpmMechStopEnd
    .Length = 200                  ' travel distance
    .Steps = 200                  ' position steps
    .Acc = 25                     ' acceleration
    .Ret = 10                     ' return speed
    .AddSw swMechHome, 0, 5       ' home switch at positions 0-5
    .AddSw swMechEnd, 195, 200    ' end switch at positions 195-200
End With
Mech.Start

' CRITICAL: Set HandleMech = 0 in table script
' Without this, PinMAME's mech handling interferes
```

### Mech Type Flags (combine with +)
| Flag | Value | Description |
|------|-------|-------------|
| `vpmMechLinear` | &H00 | Linear movement |
| `vpmMechNonLinear` | &H01 | Non-linear movement |
| `vpmMechCircle` | &H00 | Circular path |
| `vpmMechStopEnd` | &H02 | Stop at ends |
| `vpmMechReverse` | &H04 | Reverse direction |
| `vpmMechOneSol` | &H00 | Single solenoid |
| `vpmMechOneDirSol` | &H10 | One-direction solenoid |
| `vpmMechTwoDirSol` | &H20 | Two-direction solenoid |
| `vpmMechStepSol` | &H40 | Stepper solenoid |

---

## cvpmFlips2 — Flipper System

Manages flipper enable/disable, fast flips, and ROM interaction. Created automatically by `vpmInit`.

### Key Members
| Member | Type | Description |
|--------|------|-------------|
| `.FlipL(enabled)` | Sub | Left flipper button |
| `.FlipR(enabled)` | Sub | Right flipper button |
| `.FlipUL(enabled)` | Sub | Upper left flipper |
| `.FlipUR(enabled)` | Sub | Upper right flipper |
| `.Enabled = True/False` | Property | Enable fast flips |
| `.FlipperSolNumber(idx)` | Property | Solenoid mapping (0=LL, 1=LR, 2=UL, 3=UR) |
| `.TiltSol(enabled)` | Sub | Game-on/tilt solenoid |
| `.Solenoid = solNo` | Property | Game-on solenoid number |

**SAM fast flips:** Call `InitVpmFFlipsSAM` (not in core.vbs — in SAM.vbs).

---

## cvpmImpulseP — Impulse Plunger

Electronic/auto plunger with charge-and-release.

### Setup
```vbs
Dim Plunger
Set Plunger = New cvpmImpulseP
Plunger.InitImpulseP PlungerTrigger, 80, 500          ' trigger, strength, chargeTime
Plunger.Random 3                                        ' variance
Plunger.Switch swPlunger
Plunger.CreateEvents "Plunger"

' Fire from SolCallback:
SolCallback(sAutoPlunger) = "Plunger.AutoFire"
' Or manual: Plunger.Pullback / Plunger.Fire
```

---

## cvpmVLock — Visible Ball Lock

Multi-slot lock with visible balls (like medieval castle gates).

### Setup
```vbs
Dim Lock
Set Lock = New cvpmVLock
Lock.InitVLock Array(LockTrig1, LockTrig2, LockTrig3), _
               Array(LockKick1, LockKick2, LockKick3), _
               Array(swLock1, swLock2, swLock3)
Lock.ExitDir = 90
Lock.ExitForce = 30
Lock.CreateEvents "Lock"

SolCallback(sLockRelease) = "Lock.SolExit"
```

---

## cvpmCaptiveBall — Captive Ball in Loop

Ball trapped in a ramp/loop that transfers force to playfield balls.

### Key Properties
| Member | Type | Description |
|--------|------|-------------|
| `.InitCaptive(trig, wall, kickers, ballDir)` | Sub | Setup |
| `.ForceTrans = 0.5` | Property | Force transfer ratio |
| `.MinForce = N` | Property | Minimum exit force |
| `.RestSwitch = swNo` | Property | Ball-at-rest switch |
| `.Start()` | Sub | Create captive ball |

---

## SolCallback Array

The bridge between ROM solenoid commands and table actions.

```vbs
' Assignment (in Table1_Init or ConfigureDevices):
SolCallback(1) = "bsTrough.solOut"           ' route sol 1 to trough exit
SolCallback(15) = "SolScoop"                  ' route sol 15 to custom sub
SolCallback(17) = "SetLamp 117,"              ' flasher sharing lamp number (+100)

' Custom handler:
Sub SolScoop(enabled)
    If enabled Then                            ' CRITICAL: always guard with If enabled
        Scoop1.Kick 270, 30
    End If
End Sub
```

**SolCallback vs SolModCallback:**
- `SolCallback(N)` — binary True/False
- `SolModCallback(N)` — modulated 0-255 (for PWM solenoids, GI dimming)

---

## Global Objects

| Object | Type | Description |
|--------|------|-------------|
| `Controller` | PinMAME | ROM emulation controller |
| `vpmTimer` | cvpmTimer | Timer/callback manager |
| `vpmNudge` | cvpmNudge | Nudge/tilt handler |
| `vpmFlips` | cvpmFlips2 | Flipper manager |
| `vpmTrough` | cvpmTrough | Default trough (set by `.IsTrough = True`) |
| `Lights(260)` | Array | All lamp objects (indexed by lamp number) |
| `SolCallback(68)` | Array | Solenoid callback routing |
| `GICallback` | Function | GI string change handler |
| `GICallback2` | Function | Second GI string handler |
| `LampCallback` | Function | Called after all lamp updates |

---

## Key Functions (Not in Classes)

```vbs
' Ball creation
Set ball = vpmCreateBall(kicker)              ' create ball in kicker

' Switch control
Controller.Switch(swNo) = True/False          ' set switch state directly
vpmTimer.PulseSw swNo                         ' pulse switch briefly

' Lamp state
Lampz.state(lampNo) = 0/1                    ' set lamp (if using Lampz)

' Key constants (available in all table scripts)
LeftFlipperKey, RightFlipperKey, PlungerKey, StartGameKey,
AddCreditKey, LeftTiltKey, RightTiltKey, CenterTiltKey
```

---

## Common Patterns

### Wiring a complete trough + drain
```vbs
' Objects needed on table: Drain (kicker), BallRelease (kicker), Trough1-4 (kickers)
Dim bsTrough: Set bsTrough = New cvpmTrough
bsTrough.Size = 4
bsTrough.InitSwitches Array(31, 32, 33, 34)    ' switch numbers from manual
bsTrough.InitExit BallRelease, 90, 5
bsTrough.CreateEvents "bsTrough", Drain
bsTrough.Balls = 4
bsTrough.IsTrough = True
SolCallback(9) = "bsTrough.solOut"
```

### Wiring a scoop that starts a mode
```vbs
Dim bsScoop: Set bsScoop = New cvpmSaucer
bsScoop.InitKicker Scoop, 41, 270, 30, 5
bsScoop.CreateEvents "bsScoop", Scoop
SolCallback(15) = "bsScoop.solOut"

' Then in your game logic, check switch 41 to start modes
```

---

## Anti-Patterns

### Calling vpmTimer.Add
**Never use it.** Causes cryptic `VBSE_OLE` errors. Use VPX timer objects or queue-based alternatives.

### Missing SolCallback guard
Every custom SolCallback sub MUST have `If enabled Then`. The callback fires for BOTH activation and deactivation.

### Using cvpmBallStack
Older approach that destroys/creates balls. Use physical trough (cvpmTrough) with kickers instead — balls persist, IDs stay stable.
