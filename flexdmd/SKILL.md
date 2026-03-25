---
name: vpx-dev-flexdmd
description: "VPX FlexDMD DMD LCD display scene label font image video animation action score scoreboard segment UltraDMD"
version: "1.0.0"
last_updated: "2026-03-24"
---

# FlexDMD — DMD & Display System

## When This Skill Applies

Use when building a score display, mode callout screen, attract mode DMD, LCD scoreboard, or any on-screen display in a VPX table. FlexDMD is the standard display engine for both original and ROM-based VPX tables.

**Author:** Vincent Bousquet (vbousquet/Niwak)
**Source:** Built into VPX 10.8+ as native plugin. Also available as standalone COM object for pre-10.8.
**Architecture:** LibGDX Scene2D-inspired — a Stage (root Group) containing Actors (Label, Image, Frame, Video) with an Action system for animations.

## Setup

### Create and Initialize

```vbs
Dim FlexDMD, UseFlexDMD

Sub InitFlexDMD()
    On Error Resume Next
    Set FlexDMD = CreateObject("FlexDMD.FlexDMD")
    On Error Goto 0
    If FlexDMD Is Nothing Then UseFlexDMD = False : Exit Sub
    UseFlexDMD = True

    FlexDMD.GameName = cGameName
    FlexDMD.RenderMode = 2           ' 0=gray2, 1=gray4, 2=RGB color
    FlexDMD.Width = 128
    FlexDMD.Height = 32
    FlexDMD.ProjectFolder = "./FlexDMD/"
    FlexDMD.TableFile = Table1.Filename
    FlexDMD.Clear = True
    FlexDMD.Run = True
End Sub
```

**CRITICAL:** Set Width/Height BEFORE `Run = True`. Changing after causes reinitialization flicker.
**CRITICAL:** Detect via error handling, NEVER filesystem checks (fails on standalone).

### VPX Objects Required

- **Timer:** `FlexDMDTimer` — Interval = -1, Enabled = True
- **Flasher or DMD object:** with "Use Script DMD" enabled to display the output

### Render Loop

```vbs
Sub FlexDMDTimer_Timer()
    If Not UseFlexDMD Then Exit Sub
    Dim DMDp
    DMDp = FlexDMD.DmdColoredPixels
    If Not IsEmpty(DMDp) Then
        DMDWidth = FlexDMD.Width
        DMDHeight = FlexDMD.Height
        DMDColoredPixels = DMDp
    End If
End Sub
```

**NEVER disable this timer.** If disabled, FlexDMD breaks on subsequent games without a table restart.

---

## RenderMode

| Value | Name | Use For |
|-------|------|---------|
| 0 | DMD_GRAY_2 | 4-shade grayscale (classic EM look) |
| 1 | DMD_GRAY_4 | 16-shade grayscale (WPC DMD) — default |
| 2 | DMD_RGB | Full color (modern/LCD style) |
| 3-16 | SEG_* | Segment displays (alphanumeric, see Segment section) |

**DMD color:** `FlexDMD.Color = RGB(255, 88, 32)` tints grayscale output to simulate colored DMD hardware (orange, red, etc.).

---

## Core Concepts

### Stage
The root Group. All visible actors are children of the Stage:
```vbs
FlexDMD.Stage    ' returns the root Group
```

### Thread Safety
**MUST wrap all stage modifications in lock/unlock:**
```vbs
FlexDMD.LockRenderThread
' ... add/remove/modify actors ...
FlexDMD.UnlockRenderThread
```
Excessive lock/unlock calls cause frame drops. Batch modifications into fewer lock blocks.

### Actors
Everything visible is an Actor: Label, Image, Frame, Video, Group. All share base properties.

---

## Actor Base (all actors inherit these)

### Properties
| Property | Type | Description |
|----------|------|-------------|
| `Name` | string | Identifier |
| `X`, `Y` | float | Position relative to parent |
| `Width`, `Height` | int | Size |
| `PrefWidth`, `PrefHeight` | int | Natural/preferred size |
| `Visible` | bool | Show/hide |
| `FillParent` | bool | Auto-size to parent bounds |
| `ClearBackground` | bool | Clear behind this actor |

### Methods
| Method | Description |
|--------|-------------|
| `SetBounds(x, y, w, h)` | Set position and size |
| `SetPosition(x, y)` | Set position only |
| `SetAlignedPosition(x, y, alignment)` | Position with anchor point |
| `SetSize(w, h)` | Set size only |
| `Pack()` | Size to preferred/natural size |
| `Remove()` | Remove from parent |
| `AddAction(action)` | Add animation action |
| `ClearActions()` | Remove all actions |

### Alignment Enum
| Value | Name | Value | Name |
|-------|------|-------|------|
| 0 | TopLeft | 1 | Top |
| 2 | TopRight | 3 | Left |
| 4 | Center | 5 | Right |
| 6 | BottomLeft | 7 | Bottom |
| 8 | BottomRight | | |

```vbs
' Center a label on the DMD
myLabel.SetAlignedPosition 64, 16, 4    ' x=64, y=16, alignment=Center
```

---

## Label

Text display. The most-used actor.

```vbs
Dim scoreFont, scoreLabel
Set scoreFont = FlexDMD.NewFont("FlexDMD.Resources.udmd-f12x10.fnt", RGB(255,88,32), RGB(0,0,0), 0)
Set scoreLabel = FlexDMD.NewLabel("score", scoreFont, "0")
scoreLabel.SetAlignedPosition 64, 16, 4    ' centered on 128x32 DMD
FlexDMD.LockRenderThread
FlexDMD.Stage.AddActor scoreLabel
FlexDMD.UnlockRenderThread
```

### Label Properties
| Property | Type | Description |
|----------|------|-------------|
| `Text` | string | Display text (supports multi-line with vbCrLf) |
| `Font` | Font | Font object |
| `Alignment` | enum | Text alignment within bounds |
| `AutoPack` | bool | Auto-resize to fit text |

### Updating Text
```vbs
Sub UpdateScore(newScore)
    FlexDMD.LockRenderThread
    scoreLabel.Text = FormatNumber(newScore, 0)
    FlexDMD.UnlockRenderThread
End Sub
```

---

## Font

```vbs
Set myFont = FlexDMD.NewFont(fontPath, tint, borderTint, borderSize)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `fontPath` | string | Path to .fnt file (BMFont format) |
| `tint` | uint | Color tint: `RGB(r, g, b)` |
| `borderTint` | uint | Outline color |
| `borderSize` | int | Outline thickness (0 = none) |

### Built-in Fonts (always available)
| Font | Size | Style |
|------|------|-------|
| `FlexDMD.Resources.udmd-f5x5.fnt` | 5x5 | Tiny pixel |
| `FlexDMD.Resources.udmd-f7x5.fnt` | 7x5 | Small pixel |
| `FlexDMD.Resources.udmd-f12x10.fnt` | 12x10 | Medium (good for scores) |

### Font Path Formats
| Prefix | Source |
|--------|--------|
| `FlexDMD.Resources.` | Built-in bitmap fonts |
| `VPX.` | Embedded in the .vpx table file |
| (none) | File relative to `ProjectFolder` |

---

## Image

```vbs
Set myImage = FlexDMD.NewImage("logo", "logo.png")
myImage.SetBounds 0, 0, 128, 32
myImage.Scaling = 0    ' Fit
```

### Scaling Enum
| Value | Name | Behavior |
|-------|------|----------|
| 0 | Fit | Scale to fit, preserve aspect ratio |
| 1 | Fill | Scale to fill, preserve ratio (may crop) |
| 2 | FillX | Match width, proportional height |
| 3 | FillY | Match height, proportional width |
| 4 | Stretch | Stretch to exact size |
| 7 | None | Original size |

### Asset Path Features
Append filters with `&`:
- `"logo.png&dmd=3"` — DMD dot filter, dot size 3
- `"logo.png&region=0,0,64,32"` — Crop region
- `"logo.png&pad=2,2,2,2"` — Add padding

**Supported formats:** PNG, JPG, BMP (images), WMV, AVI, MP4 (video), GIF (animated)

---

## Video / Animation

```vbs
Set myVideo = FlexDMD.NewVideo("intro", "intro.gif")
myVideo.SetBounds 0, 0, 128, 32
myVideo.Loop = False
```

`NewVideo` auto-detects type from the file:
- `.gif` → GIF animation
- `.wmv` / `.avi` / `.mp4` → Video
- Images with `|` separator → Image sequence: `"frame1.png|frame2.png|frame3.png"`

### Properties
| Property | Type | Description |
|----------|------|-------------|
| `Length` | float (read) | Duration in seconds |
| `Loop` | bool | Loop playback (default True) |
| `Paused` | bool | Pause/resume |
| `PlaySpeed` | float | Speed multiplier (default 1.0) |
| `Scaling` | enum | See Scaling enum |
| `Alignment` | enum | See Alignment enum |

### Methods
| Method | Description |
|--------|-------------|
| `Seek(seconds)` | Jump to time position |

---

## Frame (Rectangle/Border)

```vbs
Set myFrame = FlexDMD.NewFrame("border")
myFrame.SetBounds 0, 0, 128, 32
myFrame.Thickness = 2
myFrame.BorderColor = RGB(255, 255, 255)
myFrame.Fill = True
myFrame.FillColor = RGB(0, 0, 0)
```

| Property | Type | Description |
|----------|------|-------------|
| `Thickness` | int | Border width (default 2) |
| `BorderColor` | uint | Border color |
| `Fill` | bool | Fill interior |
| `FillColor` | uint | Fill color |

---

## Group (Container)

```vbs
Set myGroup = FlexDMD.NewGroup("scene1")
myGroup.SetBounds 0, 0, 128, 32

Set bg = FlexDMD.NewImage("bg", "background.png")
bg.FillParent = True
myGroup.AddActor bg

Set title = FlexDMD.NewLabel("title", titleFont, "MULTIBALL!")
title.SetAlignedPosition 64, 16, 4
myGroup.AddActor title
```

| Method | Description |
|--------|-------------|
| `AddActor(actor)` | Add child (last added = on top) |
| `AddActorAt(actor, index)` | Insert at z-order |
| `RemoveActor(actor)` | Remove child |
| `RemoveAll()` | Remove all children |
| `GetLabel(name)` | Get typed child by name |
| `GetImage(name)` | Get typed child by name |
| `GetGroup(name)` | Get typed child by name |
| `HasChild(name)` | Check if child exists |
| `ChildCount` | Number of children |
| `Clip` | bool — clip children to group bounds |

---

## Action System (Animations)

Every actor has an `ActionFactory`. Create actions from the factory, add them to the actor.

```vbs
Dim af : Set af = myLabel.ActionFactory
```

### Available Actions

| Factory Method | Description |
|---------------|-------------|
| `af.Wait(seconds)` | Pause |
| `af.Delayed(seconds, action)` | Run action after delay |
| `af.Show(visible)` | Set visibility |
| `af.Blink(showSecs, hideSecs, count)` | Toggle visibility |
| `af.MoveTo(x, y, duration)` | Tween position (returns action with `.Ease` property) |
| `af.Sequence()` | Sequential container — call `.Add(action)` |
| `af.Parallel()` | Parallel container — call `.Add(action)` |
| `af.Repeat(action, count)` | Repeat N times |
| `af.AddTo(parentGroup)` | Add actor to group |
| `af.RemoveFromParent()` | Remove actor from parent |
| `af.AddChild(actor)` | Add child to group actor |
| `af.RemoveChild(actor)` | Remove child from group |
| `af.Seek(seconds)` | Seek animated actor |

### Animation Example — Slide-in Title
```vbs
Dim af : Set af = titleLabel.ActionFactory
Dim seq : Set seq = af.Sequence()
seq.Add af.Show(True)
Dim moveIn : Set moveIn = af.MoveTo(64, 16, 0.3)
moveIn.Ease = 22    ' SineOut
seq.Add moveIn
seq.Add af.Wait(2.0)
Dim moveOut : Set moveOut = af.MoveTo(64, -20, 0.3)
moveOut.Ease = 21    ' SineIn
seq.Add moveOut
seq.Add af.Show(False)
titleLabel.AddAction seq
```

### Interpolation (Ease) Values
| Value | Name | Value | Name |
|-------|------|-------|------|
| 0 | Linear | 1-3 | ElasticIn/Out/InOut |
| 4-6 | QuadIn/Out/InOut | 7-9 | CubeIn/Out/InOut |
| 10-12 | QuartIn/Out/InOut | 13-15 | QuintIn/Out/InOut |
| 16-18 | SineIn/Out/InOut | 19-21 | BounceIn/Out/InOut |
| 22-24 | CircIn/Out/InOut | 25-27 | ExpoIn/Out/InOut |
| 28-30 | BackIn/Out/InOut | | |

---

## Common Patterns

### Score Display (simplest useful case)
```vbs
Dim scoreFont, scoreLabel

Sub InitDMD()
    On Error Resume Next
    Set FlexDMD = CreateObject("FlexDMD.FlexDMD")
    On Error Goto 0
    If FlexDMD Is Nothing Then Exit Sub

    FlexDMD.GameName = cGameName
    FlexDMD.RenderMode = 2
    FlexDMD.Width = 128 : FlexDMD.Height = 32
    FlexDMD.Clear = True : FlexDMD.Run = True

    Set scoreFont = FlexDMD.NewFont("FlexDMD.Resources.udmd-f12x10.fnt", _
        RGB(255,88,32), RGB(0,0,0), 0)
    Set scoreLabel = FlexDMD.NewLabel("score", scoreFont, "0")
    scoreLabel.SetAlignedPosition 64, 20, 4

    FlexDMD.LockRenderThread
    FlexDMD.Stage.AddActor scoreLabel
    FlexDMD.UnlockRenderThread
End Sub

Sub UpdateDMDScore()
    If FlexDMD Is Nothing Then Exit Sub
    FlexDMD.LockRenderThread
    scoreLabel.Text = FormatNumber(Score, 0)
    FlexDMD.UnlockRenderThread
End Sub
```

### Scene System (Pre-built Scenes)
Create all scenes once at init. Switch between them by adding/removing from Stage.

```vbs
Dim SceneGameplay, SceneAttract, SceneModeIntro

Sub InitScenes()
    ' Gameplay scene
    Set SceneGameplay = FlexDMD.NewGroup("gameplay")
    SceneGameplay.SetBounds 0, 0, 128, 32
    Set scoreLabel = FlexDMD.NewLabel("score", scoreFont, "0")
    scoreLabel.SetAlignedPosition 64, 20, 4
    SceneGameplay.AddActor scoreLabel

    ' Attract scene
    Set SceneAttract = FlexDMD.NewGroup("attract")
    SceneAttract.SetBounds 0, 0, 128, 32
    Set attractLabel = FlexDMD.NewLabel("title", titleFont, "PRESS START")
    attractLabel.SetAlignedPosition 64, 16, 4
    SceneAttract.AddActor attractLabel

    ' Mode intro scene
    Set SceneModeIntro = FlexDMD.NewGroup("mode_intro")
    SceneModeIntro.SetBounds 0, 0, 128, 32
    Set modeTitle = FlexDMD.NewLabel("title", scoreFont, "")
    modeTitle.SetAlignedPosition 64, 16, 4
    SceneModeIntro.AddActor modeTitle
End Sub

Sub ShowScene(scene)
    FlexDMD.LockRenderThread
    FlexDMD.Stage.RemoveAll
    FlexDMD.Stage.AddActor scene
    FlexDMD.UnlockRenderThread
End Sub

' Usage:
ShowScene SceneAttract     ' attract mode
ShowScene SceneGameplay    ' during play
```

### Mode Callout with Animation
```vbs
Sub ShowModeCallout(modeName, duration)
    Dim lbl : Set lbl = SceneModeIntro.GetLabel("title")
    lbl.Text = modeName

    ' Slide in from bottom, hold, slide out
    lbl.ClearActions
    lbl.SetAlignedPosition 64, 40, 4    ' start below screen
    Dim af : Set af = lbl.ActionFactory
    Dim seq : Set seq = af.Sequence()
    Dim moveIn : Set moveIn = af.MoveTo(64, 16, 0.3)
    moveIn.Ease = 22    ' SineOut
    seq.Add moveIn
    seq.Add af.Wait(duration)
    Dim moveOut : Set moveOut = af.MoveTo(64, -10, 0.3)
    moveOut.Ease = 21    ' SineIn
    seq.Add moveOut
    lbl.AddAction seq

    ShowScene SceneModeIntro
End Sub
```

### Two-Line Score + Info Display
```vbs
Sub InitTwoLineDisplay()
    Set SceneGameplay = FlexDMD.NewGroup("gameplay")
    SceneGameplay.SetBounds 0, 0, 128, 32

    ' Small font for top info line
    Set infoFont = FlexDMD.NewFont("FlexDMD.Resources.udmd-f5x5.fnt", _
        RGB(200,200,200), RGB(0,0,0), 0)
    Set infoLabel = FlexDMD.NewLabel("info", infoFont, "BALL 1")
    infoLabel.SetAlignedPosition 64, 5, 4

    ' Large font for score
    Set scoreFont = FlexDMD.NewFont("FlexDMD.Resources.udmd-f12x10.fnt", _
        RGB(255,88,32), RGB(0,0,0), 0)
    Set scoreLabel = FlexDMD.NewLabel("score", scoreFont, "0")
    scoreLabel.SetAlignedPosition 64, 22, 4

    SceneGameplay.AddActor infoLabel
    SceneGameplay.AddActor scoreLabel
End Sub
```

---

## Segment Displays

For EM and early solid-state tables:

```vbs
FlexDMD.RenderMode = 3    ' SEG_2x16Alpha (or other SEG_* value)
FlexDMD.Run = True

' Write segment data (uint16 per character, bits = segment states)
Dim segData(31)
segData(0) = &H7B    ' character segments
FlexDMD.Segments = segData
```

| RenderMode | Display Type |
|-----------|--------------|
| 3 | 2x16 Alpha |
| 4 | 2x20 Alpha |
| 5 | 2x7 Alpha + 2x7 Numeric |
| 6-16 | Various numeric/alpha combos |

---

## UltraDMD Compatibility

For tables written against UltraDMD, FlexDMD provides a drop-in replacement:

```vbs
Set UltraDMD = FlexDMD.NewUltraDMD()
UltraDMD.Init
```

Key UltraDMD methods (work without modification):

| Method | Description |
|--------|-------------|
| `DisplayScoreboard(nPlayers, highlighted, s1, s2, s3, s4, lower1, lower2)` | 4-player scoreboard |
| `DisplayScene00(bg, topText, topBright, bottomText, bottomBright, animIn, pause, animOut)` | Two-line scene |
| `DisplayScene01(id, bg, text, bright, outline, animIn, pause, animOut)` | Single-line scene |
| `ScrollingCredits(bg, text, bright, animIn, pause, animOut)` | Scrolling text |
| `DisplayText(text, brightness, outlineBrightness)` | Simple text |
| `CancelRendering()` | Clear scene queue |
| `IsRendering()` | Check if scene active |

### UltraDMD Animation Types
| Value | Name | Value | Name |
|-------|------|-------|------|
| 0 | FadeIn | 1 | FadeOut |
| 2 | ZoomIn | 3 | ZoomOut |
| 4-5 | ScrollOffLeft/Right | 6-7 | ScrollOnLeft/Right |
| 8-9 | ScrollOffUp/Down | 10-11 | ScrollOnUp/Down |
| 14 | None | | |

---

## Anti-Patterns

### Recreating Scenes Every Time
**Wrong:** `Set scene = FlexDMD.NewGroup(...)` inside a show function.
**Fix:** Create all scenes once in Init, swap via `Stage.RemoveAll` + `Stage.AddActor`.

### Disabling the FlexDMD Timer
**Wrong:** `FlexDMDTimer.Enabled = False` between games.
**Fix:** Keep timer enabled for the entire session. Disabling breaks DMD on subsequent games.

### Control Logic in DMD Timer
**Wrong:** Mode transitions, scoring, game state changes inside `FlexDMDTimer_Timer`.
**Fix:** Timer only reads variables and updates visuals. Control logic lives in GameTimer or event handlers.

### Filesystem Detection
**Wrong:** `If Dir("FlexDMD.dll") <> "" Then ...`
**Fix:** Use `On Error Resume Next` + `CreateObject`. FlexDMD is built into standalone VPX.

### Lock Thrashing
**Wrong:** Lock/unlock for every individual property change.
**Fix:** Batch modifications inside a single lock/unlock pair. Excessive locking causes frame drops.
