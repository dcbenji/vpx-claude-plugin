# VPX Dev Plugin for Claude Code

A Claude Code plugin that enables AI-assisted Visual Pinball X table development. Helps designers create game rules, wire PinMAME ROMs, configure nFozzy physics, manage lighting, and script table logic — without needing to know VBScript.

## What It Does

When you're working on a VPX table and ask Claude for help, the relevant skill(s) automatically load based on your question:

- **"Help me wire up T2 solenoids"** → PinMAME wiring skill activates
- **"Configure flippers for a 90s WPC table"** → nFozzy physics skill activates
- **"My ball passes through the ramp"** → Troubleshooting skill activates
- **"I'm starting a new table"** → Master skill provides the full workflow + quick start template

## Skills Included

| Skill | Purpose |
|-------|---------|
| **Master** | Build workflow, quick start template, skill routing |
| **object-reference** | Every VPX object type: events, properties, methods, usage examples |
| **glf** | Game Logic Framework for original tables: modes, shots, multiball, scoring, events |
| **core-vbs** | core.vbs API: cvpmTrough, cvpmSaucer, cvpmDropTarget, cvpmMagnet, cvpmMech, SolCallback |
| **flexdmd** | FlexDMD display system: scenes, labels, fonts, images, videos, animations, scoreboards |
| **conventions** | Naming, layers, timers, performance, standalone compatibility |
| **nfozzy-physics** | Flipper tuning by era, rubber separation, targets, materials |
| **game-logic** | Modes, multiball, scoring, ball locks, tilt, attract, wizard mode |
| **pinmame-wiring** | ROM switches, coils, lamps, SolCallback, FastFlips, trough |
| **lighting** | GI strings, flasher decay, inserts, Light State Controller |
| **sound** | Fleep setup, ball rolling, ramp transitions, SSF |
| **toys-mechs** | Diverters, magnets, VUKs, spinners, drop targets, motors |
| **troubleshooting** | 28+ symptom-to-fix patterns for common VPX issues |

## Installation

Copy the plugin directory to your Claude Code skills folder:

```bash
cp -r . ~/.claude/skills/vpx-dev/
```

Or clone directly:

```bash
git clone https://github.com/dcbenji/vpx-claude-plugin.git ~/.claude/skills/vpx-dev/
```

Skills activate automatically via Claude Code's description-keyword matching — no manual invocation needed.

## Requirements

- Claude Code (CLI)
- VPX 10.8+ (standalone or Windows)
- nFozzy physics (included by default in all generated scripts)

## Source

All knowledge distilled from the [VPW (Virtual Pinball Workshop)](https://discord.gg/vpinworkshop) community — 88 Discord channels, 166+ documented patterns, consolidated into the [VPW Wiki](https://skirmishofwit.com/vpin-wiki/).

## License

MIT
