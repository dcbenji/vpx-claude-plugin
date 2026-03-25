---
name: vpx-dev-table-format
description: "VPX extracted table JSON format gamedata gameitems materials collections script object file structure vpxtool extract assemble edit modify"
version: "1.0.0"
last_updated: "2026-03-25"
---

# VPX Extracted Table Format

## When This Skill Applies

Use when reading, editing, or creating VPX table data in extracted JSON format. This is how Claude directly modifies table objects, physics, materials, collections, and scripts without the GUI editor.

**Prerequisite:** The user must extract their .vpx file first using `vpxtool`:
```bash
vpxtool extract my-table.vpx          # creates my-table/ directory
# ... Claude edits JSON files ...
vpxtool assemble my-table/            # rebuilds my-table.vpx
```

## Directory Structure

```
extracted_dir/
├── gamedata.json           # Table physics and global settings
├── gameitems.json          # Index of all game objects
├── materials.json          # Physics material definitions
├── collections.json        # Object groupings (dPosts, Fleep sounds, etc.)
├── images.json             # Image asset metadata
├── sounds.json             # Sound asset metadata
├── script.vbs              # Table VBScript (the game logic)
├── info.json               # Table metadata (author, description) — optional
├── renderprobes.json       # Render probes — optional
├── gameitems/              # One JSON file per game object
│   ├── Flipper.LeftFlipper.json
│   ├── Wall.SideWall001.json
│   ├── Bumper.Bumper001.json
│   └── ...
├── images/                 # Image asset files (png, jpg)
└── sounds/                 # Sound asset files (wav, mp3)
```

---

## gamedata.json — Table Properties

Global physics and table settings. Key properties for nFozzy tables:

```json
{
  "name": "My Table",
  "right": 1050,
  "bottom": 1900,
  "gravity": 0.97,
  "friction": 0.075,
  "elasticity": 0.8,
  "scatter": 0.0,
  "global_difficulty": 0.5,
  "nudge_time": 5,
  "angle_tilt_max": 6.0,
  "angle_tilt_min": -6.0,
  "image": "playfield_image",
  "playfield_material": "Playfield",
  "ball_image": "",
  "glass_top_height": 600,
  "glass_bottom_height": 0,
  "light_ambient": "#333333",
  "light_height": 500.0,
  "light_range": 3000.0,
  "bloom_strength": 0.4,
  "exposure": 0.1
}
```

**Common edits:**
- `global_difficulty` — nFozzy tables use 0.5 (maps to difficulty ~50 in VPX)
- `gravity` — standard is 0.97
- `friction` — playfield friction 0.15-0.25 for nFozzy
- `right` / `bottom` — playfield dimensions in VP units (standard ~1050 x 1900)

---

## gameitems.json — Object Index

Array of entries pointing to individual object files:

```json
[
  {
    "file_name": "Flipper.LeftFlipper.json",
    "is_locked": false,
    "editor_layer": 0,
    "editor_layer_name": "Layer 1",
    "editor_layer_visibility": true
  }
]
```

**When adding a new object:** Append an entry here AND create the corresponding file in `gameitems/`.

**When deleting an object:** Remove the entry here AND delete the file from `gameitems/`.

---

## Game Object Files — `gameitems/*.json`

Each file wraps the object data under its type key:

```json
{ "Flipper": { "name": "LeftFlipper", "center": { "x": 230, "y": 1800 }, ... } }
```

**The top-level key IS the type name.** This wrapping format is required — a flat `{ "name": "LeftFlipper" }` will break.

### 21 Object Types

| Type | Position Field | Common Use |
|------|---------------|------------|
| Flipper | `center` {x,y} | Main/upper flippers |
| Bumper | `center` {x,y} | Pop bumpers |
| Wall | `drag_points` [{x,y,z}...] | Walls, slingshots, playfield boundary |
| Trigger | `center` {x,y} | Flipper triggers, lane triggers, switches |
| Kicker | `center` {x,y} | Trough kickers, scoops, VUKs |
| Light | `center` {x,y} | Inserts, GI |
| HitTarget | `position` {x,y,z} | Drop targets, standup targets |
| Gate | `center` {x,y} | One-way gates |
| Spinner | `center` {x,y} | Spinners |
| Ramp | `drag_points` [{x,y,z}...] | Wire/plastic ramps |
| Rubber | `drag_points` [{x,y,z}...] | Rubber bands, rings |
| Primitive | `position` {x,y,z} | 3D meshes, plastics |
| Flasher | `center` {x,y} | Flasher lights |
| Plunger | `center` {x,y} | Ball launcher |
| Timer | `center` {x,y} | GameTimer, FrameTimer, LampTimer |
| Decal | `center` {x,y} | Stickers, markings |
| TextBox | `ver1`/`ver2` {x,y} | Text displays |
| Reel | `ver1`/`ver2` {x,y} | Mechanical reel displays |
| LightSequencer | `center` {x,y} | Light animation sequences |
| PartGroup | `position` {x,y,z} | Object grouping (10.8.1+) |
| Ball | `center` {x,y} | Ball objects |

---

## Key Object Examples

### Flipper

```json
{
  "Flipper": {
    "name": "LeftFlipper",
    "center": { "x": 230, "y": 1800 },
    "base_radius": 21.5,
    "end_radius": 13.0,
    "flipper_radius_max": 130.0,
    "flipper_radius_min": 0.0,
    "height": 50.0,
    "start_angle": 121.0,
    "end_angle": 70.0,
    "strength": 2200.0,
    "return": 0.058,
    "elasticity": 0.8,
    "elasticity_falloff": 0.43,
    "friction": 0.6,
    "ramp_up": 3.0,
    "scatter": 0.0,
    "mass": 1.0,
    "torque_damping": 0.75,
    "torque_damping_angle": 6.0,
    "rubber_thickness": 7.0,
    "rubber_height": 19.0,
    "rubber_width": 24.0,
    "material": "Rubber",
    "rubber_material": "Rubber",
    "surface": "",
    "is_visible": true,
    "is_enabled": true,
    "is_reflection_enabled": true,
    "timer_enabled": false,
    "timer_interval": 0
  }
}
```

**Note:** The `return` property is a JavaScript reserved word. When editing in code, access as `obj["return"]`, not `obj.return`.

### Bumper

```json
{
  "Bumper": {
    "name": "Bumper001",
    "center": { "x": 400, "y": 700 },
    "radius": 45.0,
    "force": 10.0,
    "threshold": 1.6,
    "scatter": 0.0,
    "height_scale": 90.0,
    "orientation": 0.0,
    "ring_speed": 0.5,
    "ring_drop_offset": 0.0,
    "cap_material": "",
    "base_material": "",
    "ring_material": "",
    "socket_material": "",
    "surface": "",
    "is_cap_visible": true,
    "is_base_visible": true,
    "is_ring_visible": true,
    "is_socket_visible": true,
    "is_collidable": true,
    "hit_event": true
  }
}
```

### Wall (with drag points)

```json
{
  "Wall": {
    "name": "LeftSlingshot",
    "drag_points": [
      { "x": 170, "y": 1620, "z": 0, "smooth": false, "is_slingshot": true },
      { "x": 230, "y": 1540, "z": 0, "smooth": false, "is_slingshot": true },
      { "x": 200, "y": 1650, "z": 0, "smooth": false, "is_slingshot": false }
    ],
    "height_bottom": 0.0,
    "height_top": 50.0,
    "elasticity": 0.3,
    "friction": 0.3,
    "scatter": 0.0,
    "slingshot_force": 80.0,
    "slingshot_threshold": 2.0,
    "material": "Playfield",
    "sidecolor": "#ffffff",
    "is_visible": true,
    "is_collidable": true,
    "is_droppable": false,
    "is_bottom_solid": false,
    "hit_event": false,
    "timer_enabled": false
  }
}
```

### Trigger

```json
{
  "Trigger": {
    "name": "LFTrigger",
    "center": { "x": 230, "y": 1800 },
    "radius": 77.0,
    "rotation": 0.0,
    "wire_thickness": 0.0,
    "animation_speed": 1.0,
    "surface": "",
    "material": "",
    "is_visible": false,
    "is_enabled": true,
    "hit_height": 150.0,
    "shape": 0,
    "timer_enabled": false
  }
}
```

### Kicker (trough/scoop/VUK)

```json
{
  "Kicker": {
    "name": "Trough1",
    "center": { "x": 250, "y": 1950 },
    "radius": 25.0,
    "orientation": 0.0,
    "surface": "",
    "material": "",
    "is_enabled": true,
    "hit_accuracy": 0.5,
    "hit_height": 40.0,
    "fall_through": false,
    "legacy_mode": false,
    "timer_enabled": false
  }
}
```

### HitTarget (drop target / standup)

```json
{
  "HitTarget": {
    "name": "DropTarget001",
    "position": { "x": 400, "y": 900, "z": 0 },
    "size": { "x": 32, "y": 32, "z": 32 },
    "rot_x": 0.0,
    "rot_y": 0.0,
    "rot_z": 0.0,
    "target_type": 2,
    "threshold": 2.0,
    "elasticity": 0.5,
    "elasticity_falloff": 0.5,
    "friction": 0.3,
    "scatter": 5.0,
    "is_collidable": true,
    "is_visible": true,
    "is_dropped": false,
    "is_legacy": false,
    "raise_delay": 100,
    "material": "",
    "physics_material": "",
    "use_hit_event": true,
    "timer_enabled": false
  }
}
```

**target_type values:** 0=DropTargetBeveled, 1=DropTargetSimple, 2=DropTargetFlat, 3=HitTargetRound, 4=HitTargetRectangle, 5=HitTargetFatRectangle, 6=HitTargetSlim, 7=DropTargetFlatSimple

### Timer (invisible, no position needed)

```json
{
  "Timer": {
    "name": "GameTimer",
    "center": { "x": 100, "y": 100 },
    "timer_enabled": true,
    "timer_interval": 10
  }
}
```

### Light

```json
{
  "Light": {
    "name": "Light001",
    "center": { "x": 500, "y": 900 },
    "falloff": 50.0,
    "falloff_power": 2.0,
    "intensity": 10.0,
    "state": 0,
    "color": "#ffa957",
    "color2": "#ffa957",
    "mesh_radius": 20.0,
    "fade_speed_up": 0.05,
    "fade_speed_down": 0.02,
    "blink_pattern": "10",
    "blink_interval": 125,
    "surface": "",
    "image": "",
    "is_round_light": true,
    "show_bulb_mesh": false,
    "is_visible": true,
    "drag_points": [
      { "x": 520, "y": 900, "z": 0, "smooth": true },
      { "x": 500, "y": 920, "z": 0, "smooth": true },
      { "x": 480, "y": 900, "z": 0, "smooth": true },
      { "x": 500, "y": 880, "z": 0, "smooth": true }
    ]
  }
}
```

---

## materials.json — Physics Materials

Array of material definitions. Key materials for nFozzy tables:

```json
[
  {
    "name": "z_col_rubberposts",
    "elasticity": 0.9,
    "elasticity_falloff": 0.1,
    "friction": 0.3,
    "scatter_angle": 1.0,
    "roughness": 0.0,
    "is_metal": false,
    "is_opaque_active": false
  },
  {
    "name": "z_col_rubberpostsleeves",
    "elasticity": 0.765,
    "elasticity_falloff": 0.1,
    "friction": 0.3,
    "scatter_angle": 1.0
  },
  {
    "name": "z_col_rubberbands",
    "elasticity": 0.85,
    "elasticity_falloff": 0.13,
    "friction": 0.3,
    "scatter_angle": 0.0
  }
]
```

**To add a material:** Append to this array. **To modify:** Find by `name` and update properties.

---

## collections.json — Object Groupings

```json
[
  {
    "name": "dPosts",
    "items": ["dPost001", "dPost002", "dPost003"],
    "fire_events": true,
    "stop_single_events": false,
    "group_elements": true
  },
  {
    "name": "dSleeves",
    "items": ["dSleeve001", "dSleeve002"],
    "fire_events": true,
    "stop_single_events": false,
    "group_elements": true
  }
]
```

**Common nFozzy collections:** `dPosts`, `dSleeves`, `TargetBounce`, `Rubbers` (hit sounds)
**Common Fleep collections:** `GatesWire`, `Apron`, `Metals`, `Rollovers`, `Walls`
**Common GLF collections:** `glf_lights`, `glf_switches`, `glf_slingshots`, `glf_spinners`

---

## script.vbs — Table Script

Plain text VBScript file. Read/write as UTF-8 text. This is the entire game logic — modes, scoring, physics code (nFozzy), sound (Fleep), display (FlexDMD).

---

## Common Editing Workflows

### Audit nFozzy Rubber Setup
1. Read all objects, filter by material name containing `rubber`
2. Read `collections.json`, check `dPosts` and `dSleeves` members exist as objects
3. Read `materials.json`, verify `z_col_rubberposts`, `z_col_rubberpostsleeves`, `z_col_rubberbands` exist with correct values
4. Cross-reference: every rubber-type object should either be in dPosts, dSleeves, or use bands material

### Set Flipper Physics for Era
1. Read all Flipper objects from `gameitems/`
2. Update `strength`, `return`, `elasticity`, `mass`, `ramp_up` to match era values from nfozzy-physics skill
3. Write back each modified file

### Add Objects to Collection
1. Read `collections.json`
2. Find or create the collection by name
3. Append item names to `items` array
4. Write back

### Replace Table Script
1. Read `script.vbs`
2. Generate or modify VBScript using vpx-dev plugin knowledge
3. Write back to `script.vbs`

---

## Gotchas

- **Object names are case-insensitive** in VPX — `LeftFlipper` and `leftflipper` are the same object
- **The wrapping format is required** — `{ "Flipper": { ... } }`, NOT `{ "name": "LeftFlipper", ... }`
- **Flipper `return` is a reserved word** — access via bracket notation `obj["return"]`, never `obj.return`
- **Drag points define geometry** — Walls, Ramps, and Rubbers need drag_points arrays. Don't create these without knowing the intended shape.
- **gameitems.json must stay in sync** — every file in `gameitems/` needs a matching entry in the index, and vice versa
- **Timer objects are invisible** — position doesn't matter, but `timer_enabled` and `timer_interval` are critical
- **Material names are referenced by string** — if you rename a material in materials.json, update all objects that reference it
