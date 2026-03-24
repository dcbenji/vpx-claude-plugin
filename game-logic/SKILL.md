---
name: vpx-dev-game-logic
description: "VPX VBScript pinball game rules modes multiball scoring ball lock ball save tilt attract wizard mode state machine FlexDMD drain"
version: "1.0.0"
last_updated: "2026-03-24"
---

# Game Logic Patterns

## When This Skill Applies

Use when implementing: game modes, multiball, scoring, ball locks, ball save, tilt handling, attract mode, wizard modes, FlexDMD scenes, drain logic, or player state management.

Always apply conventions skill rules (naming, timers, standalone compatibility) to all generated VBScript.

## Mode State Machine (Start / Active / Complete)

Every mode follows a three-phase lifecycle:

```vbs
Dim ModeActive, ModeTimer

Sub StartMode()
    ModeActive = True
    ModeTimer = 30               ' seconds
    ' Set up: lights, DMD, ball save
    UpdateModeLights True
    ShowScene Scenes(kModeIntro)
End Sub

Sub UpdateMode()                 ' called from GameTimer
    If Not ModeActive Then Exit Sub
    ModeTimer = ModeTimer - 0.01 ' 10ms tick
    If ModeTimer <= 0 Then CompleteMode False
End Sub

Sub ModeShot(shotId)
    If Not ModeActive Then Exit Sub
    AddScore 50000
    ' Track progress, check completion
    If AllShotsHit() Then CompleteMode True
End Sub

Sub CompleteMode(success)
    ModeActive = False
    UpdateModeLights False
    If success Then AddScore 500000
    ' Reset state, transition to next
End Sub
```

**Gotcha:** On tilt, modes do NOT complete through normal drain logic. Force-reset all active modes in the tilt handler.

## Multiball (gBOT Pattern)

Create all balls once at init, never destroy. Track with a global array instead of calling `GetBalls` every frame.

```vbs
Dim gBOT : gBOT = Array()       ' global Ball-On-Table array
Dim BallsInPlay

Sub StartMultiball(count)
    Dim i
    BallsInPlay = count
    BallSaverActive = True
    BallSaverTimer = 15          ' seconds
    For i = 1 To count - 1       ' -1 because one ball already in play
        ReleaseBallFromTrough
    Next
    UpdateGBOT
End Sub

Sub UpdateGBOT()
    ' Rebuild gBOT from trough state + lock state
    ' Called on ball create/release/lock events, NOT every frame
End Sub
```

**NOTE: GetBalls in Cor.Update ONLY.** The `gBOT` variable causes segfaults in standalone VPX when used inside `Cor.Update`. This is the #1 cause of "ball hits flipper -> crash" on standalone. Use `GetBalls` specifically in `Cor.Update`; use `gBOT` everywhere else.

```vbs
Sub GameTimer_Timer()
    Dim BOT: BOT = GetBalls      ' GetBalls HERE, not gBOT
    Cor.Update BOT
    ' ... other physics updates
End Sub
```

## Ball Save

Timer-based with configurable duration and max returns:

```vbs
Dim BallSaverActive, BallSaverTime, BallSaverReturns

Sub ActivateBallSaver(seconds)
    BallSaverActive = True
    BallSaverTime = seconds
    BallSaverReturns = 0
End Sub

Sub Drain_Hit()
    If BallSaverActive And BallSaverReturns < 10 Then
        BallSaverReturns = BallSaverReturns + 1
        ReleaseBallFromTrough
        Exit Sub
    End If
    ' Normal drain logic
    BallsInPlay = BallsInPlay - 1
    If BallsInPlay <= 0 Then EndBall
End Sub

Sub DecrementBallSaver()         ' called from GameTimer
    If Not BallSaverActive Then Exit Sub
    BallSaverTime = BallSaverTime - 0.01
    If BallSaverTime <= 0 Then BallSaverActive = False
End Sub
```

**Gotcha:** For high ball counts (13-15 ball multiball), cap returns at 10. First few balls drain fast; unlimited returns wastes time.

## Ball Lock Variants

### Physical Trough Replacement
Create balls at trough kicker positions using `CreateSizedBallWithMass`. **BallMass must always be 1** — even widebody tables (mass 1.3 breaks all flipper physics). Set controller switches active. Drain kicker at physical bottom. Check manual for "Install X balls" count. See also: conventions skill production checklist for pre-release verification.

```vbs
Sub InitTrough()
    Dim i
    For i = 0 To TroughSize - 1
        Set gBOT(i) = TroughKicker(i).CreateSizedBallWithMass(BallSize, BallMass)
        Controller.Switch(swTrough(i)) = True
    Next
End Sub
```

### Visual Lock (Real Ball in Mechanism)
Place invisible kicker inside lock. Create real ball in kicker when locked. Ball rotates with mechanism naturally. Kicker releases on ROM event. Eliminates texture-swap artifacts.

```vbs
Sub LockBall(lockNum)
    LockKicker(lockNum).CreateSizedBallWithMass BallSize, BallMass
    LockedBalls = LockedBalls + 1
End Sub

Sub SolReleaseLock(enabled)
    If enabled Then
        LockKicker(0).Kick 0, 30
        LockedBalls = LockedBalls - 1
    End If
End Sub
```

### Captive / Newton Ball
Oversized ball (57 wide) at post position. Create when raised, destroy/move when lowered. VPX ball collision physics transfers momentum naturally. Used in: Breakshot center, AC/DC bell, JWPS.

## Scoring (Per-Player State)

Save/restore all per-player variables at ball drain and turn start:

```vbs
Dim PlayerScore(4), PlayerMode(4), PlayerLocks(4), PlayerBonus(4)

Sub SavePlayerState(p)
    PlayerScore(p) = Score
    PlayerMode(p) = ModeProgress
    PlayerLocks(p) = LockedBalls
    PlayerBonus(p) = BonusMultiplier
End Sub

Sub RestorePlayerState(p)
    Score = PlayerScore(p)
    ModeProgress = PlayerMode(p)
    LockedBalls = PlayerLocks(p)
    BonusMultiplier = PlayerBonus(p)
End Sub
```

**Gotcha:** Counters like `rampHitsThisBall` may not reset between games if certain end-game paths skip normal reset. Initialize ALL player arrays to defaults at game start.

## Tilt Handling

### Script-Based Tilt (Original Tables)

```vbs
Dim TiltWarnings, TiltSensitivity
TiltSensitivity = 1000          ' ms ignore window (Stern default)

Sub CheckTilt()
    TiltWarnings = TiltWarnings + 1
    If TiltWarnings >= 3 Then DoTilt
End Sub

Sub DoTilt()
    ' 1. Force-reset ALL active modes
    If ModeActive Then CompleteMode False
    If MultiballActive Then EndMultiball
    ' 2. Clear flipper flags (CRITICAL)
    LFlipperHeld = False
    RFlipperHeld = False
    ' 3. Disable flippers
    LeftFlipper.RotateToStart
    RightFlipper.RotateToStart
    ' 4. Kill ball save
    BallSaverActive = False
    ' 5. Let balls drain naturally
End Sub
```

**Without clearing flipper flags:** next ball's start button press can trigger slam tilt because the held-flag from previous ball was never cleared (Goonies bug).

### Native 10.8.1 Tilt
VPX 10.8.1 has a native tilt plumb simulator in the physics engine (1kHz). Models a 3D Newtonian mass on a bar hitting a ring. LiveUI overlay shows plumb position. Use for recreation tables; script-based for originals needing custom behavior.

## Attract Mode

Initialize FlexDMD scenes once into arrays. Never recreate per-show.

```vbs
Dim Scenes(10), FlexMode
' FlexMode: 1=gameplay, 2=attract/intro, 3=highscores

Sub Flex_Init()
    On Error Resume Next
    Set FlexDMD = CreateObject("FlexDMD.FlexDMD")
    On Error Goto 0
    If FlexDMD Is Nothing Then UseFlexDMD = 0 : Exit Sub
    Set Scenes(1) = CreateGameplayScene()
    Set Scenes(2) = CreateAttractScene()
    Set Scenes(3) = CreateHighScoreScene()
End Sub

Sub StartGame()
    FlexMode = 1
    lightCtrl.StopSyncWithVpxLights  ' stop attract overriding lights
End Sub
```

**Gotchas:**
- FlexDMD update timer must stay enabled entire session. Disabling after intro breaks DMD on subsequent games.
- Detect FlexDMD via error handling, NOT filesystem checks (fails on standalone).
- `StopSyncWithVpxLights` required or attract light patterns bleed into gameplay.

## Wizard Mode

```vbs
Sub StartWizardMode()
    ' 1. Ball saver (8-10s minimum)
    ActivateBallSaver 10
    ' 2. Turn off unrelated shot lights (no "Christmas tree")
    ClearAllShotLights
    ' 3. Start wizard-specific lights
    UpdateWizardLights True
    ' 4. Set mode state
    WizardActive = True
End Sub
```

**Design rules:**
- ALWAYS include ball saver (8-10s minimum)
- Multiball must NOT start wizard -- play out multiball first, light wizard after single ball
- Extra ball stacking must work (earn 1 + earn 1 = have 2)
- Everything resets after wizard fail or complete
- Turn off unrelated shot lights during wizard

**Intro sequence pattern:** Use a physical ball trap (wall) to hold ball during intro. GI off, DMD intro text, character lighting, ~5s pause, wall drops, ball saver activates. Disable flippers during intro if multiball is active so extra balls drain first.

## FlexMode State Machine

```vbs
' FlexMode values:
' 0 = off/not initialized
' 1 = gameplay (score display, mode callouts)
' 2 = attract/intro (cycling scenes)
' 3 = high scores
' 4 = bonus counting

Sub FlexTimer_Timer()
    If UseFlexDMD = 0 Then Exit Sub
    Select Case FlexMode
        Case 1: UpdateGameplayScene
        Case 2: UpdateAttractScene
        Case 3: UpdateHighScoreScene
        Case 4: UpdateBonusCount
    End Select
End Sub
```

**Rule:** Don't put control logic inside the FlexDMD timer. Timer only updates visuals based on variables set elsewhere.

## Ball-in-Narnia Recovery

Detect and recover balls lost through playfield or stuck in impossible positions:

```vbs
Sub CheckNarniaBalls()           ' call from GameTimer periodically
    Dim i
    For i = 0 To UBound(gBOT)
        If gBOT(i).Z < -50 Then ' far below playfield
            gBOT(i).X = VUKPosX
            gBOT(i).Y = VUKPosY
            gBOT(i).Z = 50
            gBOT(i).VelX = 0
            gBOT(i).VelY = 0
            gBOT(i).VelZ = 0
        End If
    Next
End Sub
```

**Detection signs:** Rolling sound continues after ball visually disappears, or z-position far below 0. Place a catcher trigger below the playfield as a safety net.

## Quick Reference

| Pattern | Key Code | Gotcha |
|---------|----------|--------|
| SolCallback | `Sub SolX(enabled)` | **Must** include `If enabled Then` guard |
| gBOT | `Dim gBOT : gBOT = Array()` | Use `GetBalls` in `Cor.Update` ONLY (standalone segfault) |
| Ball save | Timer-based, decrement in GameTimer | Cap at 10 returns for high ball counts |
| Mode reset | Force-reset in tilt handler | Clear flipper-held flags too |
| Player state | `SavePlayerState` / `RestorePlayerState` | Init ALL arrays at game start |
| FlexDMD detect | `On Error Resume Next` + `CreateObject` | Never check filesystem (fails standalone) |
| Attract lights | `lightCtrl.StopSyncWithVpxLights` | Call on game start or attract bleeds through |
| Wizard mode | Ball saver + clear unrelated lights | Multiball gates wizard (single ball only) |
| Ball-in-Narnia | Z < -50 position check | Place catcher trigger below playfield |
| Destroyed balls | `Set ballVar = Nothing` | `IsNull()` does NOT detect destroyed ball refs |

## Anti-Patterns

### Destroying and Recreating Balls
**What it looks like:** `DestroyBall` then `CreateBall` during multiball.
**Why it's wrong:** Ball IDs change, breaking shadow tracking and gBOT references. Real machines never destroy balls.
**Fix:** Use physical trough with gBOT pattern. Create all balls once at init.

### GetBalls in Hot Loops
**What it looks like:** `GetBalls` called every frame in rolling sub or FrameTimer.
**Why it's wrong:** Primary performance bottleneck. Allocates array every call.
**Fix:** Use `gBOT` array, update only on ball create/release/lock events. Exception: `Cor.Update` in GameTimer must use `GetBalls`.

### FlexDMD Scene Recreation
**What it looks like:** Creating new scene objects every time a scene is shown.
**Why it's wrong:** Throws errors, leaks memory.
**Fix:** Initialize all scenes once in `Flex_Init` into an array. Use `ShowScene` wrapper.

### Missing SolCallback Guard
**What it looks like:** `Sub SolScoop(enabled)` without `If enabled Then`.
**Why it's wrong:** Solenoid fires on both activate AND deactivate. Sounds play twice, kicks fire on release.
**Fix:** Always wrap action code in `If enabled Then`.
