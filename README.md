# TinyBBS

A Meshtastic BBS (Bulletin Board System) module with games, mail, geocoding, and emergency survival guide — built as a [MeshForge](https://github.com/meshenvy/mesh-forge) project.

## Overview

TinyBBS adds a full-featured BBS to any Meshtastic node. Users interact via direct message using a menu-driven interface:

- **Bulletins** — multiple boards (General, Info, News, Urgent)
- **Private mail** — node-to-node messaging with subjects
- **QSL board** — radio contact logging with signal quality
- **Games** — Wordle, Vault-Tec word hacking, Chess by mail, Fallout Wasteland RPG
- **Survival guide** — emergency reference (water, fire, shelter, first aid, navigation…)
- **Reverse geocoding** — location lookups from GPS coordinates

## This is a MeshForge project

**Flash and data installation are handled automatically** by the [MeshForge web flasher](https://meshforge.app).

1. Open the MeshForge flasher
2. Select your device and the TinyBBS firmware build
3. Click **Flash** — the flasher writes the firmware and then automatically uploads the data files (Wordle dictionary, geocoding database, survival guide) to your device's external flash

No manual steps, no scripts to run, no serial terminal required.

## Firmware

The firmware lives in the `firmware/` submodule — a fork of the Meshtastic firmware with the TinyBBS module integrated directly into `src/modules/`.

To build locally:

```bash
cd firmware
pio run -e <target>   # e.g. pio run -e rak4631
```

PlatformIO will automatically:
1. Generate the data files (`bbs-data/*.bin`) via `extra_scripts/gen_meshforge_data.py`
2. Compile the firmware with the `meshenvy/meshforge-sideload` library included

## Hardware

TinyBBS targets nRF52840-based Meshtastic boards with external QSPI flash:

| Board | PlatformIO env |
|-------|---------------|
| RAK4631 | `rak4631` |
| T-Echo | `t-echo` |
| Heltec Mesh Node T114 | `heltec_mesh_node_t114` |

The game and geocoding data (>500 KB total) is stored on the external QSPI flash chip, not the internal flash. Boards without external flash will fall back gracefully (games and geocoding disabled).

## Data files

Data files are generated from source during the PlatformIO build and sideloaded by MeshForge after flashing. They are declared in `firmware/meshforge.yaml`:

```yaml
meshforge:
  data:
    - bbs-data/*.bin:/ext/bbs/kb
```

| File | Description | Size |
|------|-------------|------|
| `wordle.bin` | 12,972-word dictionary for binary search | ~65 KB |
| `geo_us.bin` | US city geocoding index (GeoNames) | ~380 KB |
| `survival.bin` | Emergency survival guide | ~20 KB |

To regenerate data files manually:

```bash
cd firmware
python3 scripts/gen_wordle_packed.py
python3 scripts/gen_geo_packed.py
python3 scripts/gen_survival_packed.py
```

## Commands

Send any of these as a direct message to your TinyBBS node:

```
(empty or ?)    Main menu
B               Bulletin board menu
M               Mail menu
Q               QSL board
G               Games menu
S               System stats
H               Help
```

Within the bulletin board:
```
L [page]        List bulletins
R <id>          Read bulletin
P <text>        Post bulletin
D <id>          Delete your bulletin
```

## Architecture

```
firmware/                   ← Meshtastic fork (the actual project)
  src/modules/
    BBSModule_v2.cpp/.h     ← Main BBS module (Meshtastic SinglePortModule + OSThread)
    BBSWordle.h             ← Wordle word list (2000 curated words for daily puzzle)
    BBSSurvival.h           ← Survival guide reader
    BBSGeoLookup.h          ← Reverse geocoding (reads geo_us.bin from external flash)
    BBSExtFlash.h/.cpp      ← Adafruit LittleFS on QSPI flash
    ...
  scripts/
    gen_wordle_packed.py    ← Generates wordle.bin from BBSWordle.h word list
    gen_geo_packed.py       ← Generates geo_us.bin from cities1000.txt
    gen_survival_packed.py  ← Generates survival.bin (tips embedded in script)
    cities1000.txt          ← GeoNames US city database (~28 MB source)
  extra_scripts/
    gen_meshforge_data.py   ← PlatformIO pre: script, runs gen_*.py at build time
  meshforge.yaml            ← Declares data files for MeshForge sideloader
  platformio.ini            ← lib_deps includes meshenvy/meshforge-sideload
```

## MeshForge sideload library

TinyBBS uses the [`meshforge-sideload`](https://registry.platformio.org/libraries/meshenvy/meshforge-sideload) PlatformIO library for post-flash data delivery. The library:

- Listens on Serial for `0xBB`-framed file transfer frames
- Routes `/ext/` paths to external QSPI flash, `/int/` paths to internal flash
- Is polled from `BBSModule::runOnce()` — no separate thread, no Meshtastic changes needed

The MeshForge web flasher drives this protocol automatically after flashing firmware.
