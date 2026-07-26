# CLAUDE.md

Working notes for this repository: project context, conventions, active tasks, and an activity log.
Keep the **Current Tasks** and **Activity Log** sections updated as work progresses.

---

## 1. Project Overview

**rover_led_udp_controller_hw** — KiCad hardware design for an ESP32-S3 based LED controller
for a rover, driven over UDP. This repo contains *hardware only* (schematics, PCB, footprints,
fabrication outputs). No firmware source lives here.

- ECAD tool: **KiCad 10.0** (`C:\Program Files\KiCad\10.0`)
- Fabrication target: **JLCPCB** (via the KiCad Fabrication Toolkit plugin)
- License: see `LICENSE`

## 2. Repository Layout

```
rover_led_udp_controller_hw/          <- repo root
├── CLAUDE.md                         <- this file
├── README.md
├── .mcp.json                         <- KiCad MCP server config (project-scoped)
├── .claude/settings.local.json
└── rover_led_udp_controller_hw/      <- KiCad project directory
    ├── rover_led_udp_controller_hw.kicad_pro   <- project file
    ├── rover_led_udp_controller_hw.kicad_sch   <- ROOT schematic sheet
    ├── TOP.kicad_sch                           <- top-level sheet (title block / hierarchy)
    ├── MCU&LEDS.kicad_sch                      <- main design sheet (all components)
    ├── rover_led_udp_controller_hw.kicad_pcb   <- PCB layout
    ├── rover_led_udp_controller_hw.kicad_dru   <- custom design rules
    ├── Libs/                                   <- project-local symbol/footprint libs
    │   ├── MechatronicsAcademy.kicad_sym / .pretty
    │   └── ESP32-S3-DevKit-LiPo_Rev_B.kicad_sym
    ├── docs/                                   <- component datasheets (PDF)
    ├── gerbers/  jlcpcb/  production/          <- fabrication outputs
    ├── 3d/                                     <- 3D models
    └── fabrication-toolkit-options.json
```

**Schematic hierarchy:** `rover_led_udp_controller_hw.kicad_sch` (root) → `TOP.kicad_sch` → `MCU&LEDS.kicad_sch`.
Nearly all real circuitry lives in `MCU&LEDS.kicad_sch`; edits usually target that file.

## 3. Hardware Architecture

| Block | Parts | Notes |
|---|---|---|
| MCU | `U4` ESP32-S3-WROOM-1-N8R8 | 8 MB flash / 8 MB PSRAM, Wi-Fi for the UDP link |
| Buck 1 | `U2` LMR51450SDRRR + `L3` 2.2 µH | 24 V input rail step-down |
| Buck 2 | `U3` SY8089AAAC (SOT23-5) + `L2` 4.7 µH | secondary / 3V3 rail |
| Level shift | `IC`, `IC2` TXS0108E | 3V3 ↔ 5 V for LED data/clock lines |
| Input protect | `Q1` P-MOS 60 V SOT-23, `F1` 2 A/1206 fuse | reverse-polarity + overcurrent |
| USB | `USB-C`, `USB-C2` USB2.0 Type-C (A40-00119-A52-12) | programming / power |
| Controls | `SW` T1107A tactile, `RST`, `BUT` | reset + boot/user button |
| LED I/O | `J10`–`J17` | two channels: `DO_OUT`/`CO_OUT`/`DI_IN`/`CI_IN` (+ `_2`) — clocked LED strips (data + clock) |
| Power I/O | `J2` +24V_PAD, `J3`/`J4` IN/OUT +5, `J5`–`J9` GND pads | pass-through 5 V and GND |

Datasheets for `SY8089AAAC`, `LMR51450`, and JLC parts `C2829089` / `C2913201` are in
`rover_led_udp_controller_hw/docs/`.

## 4. Tooling

### KiCad MCP server
Configured in `.mcp.json`. Prefer these tools over parsing `.kicad_sch` / `.kicad_pcb` by hand:

- Inspect: `list_schematic_components`, `list_schematic_nets`, `get_schematic_info`,
  `analyze_hierarchical_nets`, `trace_hierarchical_connection`
- Validate: `run_erc`, `get_erc_violations`, `run_drc`, `get_drc_violations`, `detect_pin_conflicts`
- Analysis: `extract_power_domains`, `analyze_pcb_power_integrity`, `analyze_pcb_signal_integrity`
- Output: `generate_netlist`, `export_gerber`

Defaults from `.mcp.json`: detailed summaries, nets + power included, ESP-IDF as the test framework.
It resolves `KICAD_PYTHON`, `KICAD_MCP_SERVER_HOME`, and `KICAD_PROJECT_PATHS` from env with
hardcoded fallbacks for this machine.

### Library tables
- `sym-lib-table`: `ESP32-S3-DevKit-LiPo_Rev_B`, `MechatronicsAcademy` (project-local, `${KIPRJMOD}/Libs/`),
  plus stock `Device` and `Transistor_FET` from `${KICAD10_SYMBOL_DIR}`.
- `fp-lib-table`: `MechatronicsAcademy` only.

New project-local symbols go in `Libs/` and **must** be registered in `sym-lib-table` with a
`${KIPRJMOD}` URI — never an absolute path.

## 5. Conventions & Gotchas

- **KiCad must be closed before editing project files programmatically.** `~*.lck` files in the
  project directory mean KiCad has the project open; writes will conflict or be overwritten.
- `.gitignore` excludes `*.kicad_prl`, `.history/`, `*-backups/`, `*.net`, `*.csv`, `*.xml`, and
  lock files. Don't commit those, and don't rely on exported netlists/BOMs being in the repo.
- Schematic files are large S-expression text. A one-symbol change can produce a multi-thousand-line
  diff (KiCad rewrites UUIDs and ordering) — review diffs by intent, not by line count.
- `MCU&LEDS.kicad_sch` has an `&` in the filename; **quote the path** in every shell command.
- Regenerate `gerbers/`, `jlcpcb/`, and `production/` only via the Fabrication Toolkit so
  `fabrication-toolkit-options.json` settings are applied.
- Run ERC after schematic edits and DRC after layout edits before considering work done.

## 6. Current Tasks

<!-- Update as work starts/finishes. Format: - [ ] task — owner/notes -->

- [ ] Review the in-progress `MCU&LEDS.kicad_sch` rework (large uncommitted diff, see §7) and run ERC.
- [ ] Decide whether `ESP32-S3-DevKit-LiPo_Rev_B.kicad_sym` (untracked) should be committed
      or was only a scratch import — it is registered in `sym-lib-table` either way.
- [ ] Confirm PCB layout is still consistent with the reworked schematic (PCB last touched 2025-06-26,
      schematic touched 2026-07-26); re-run "Update PCB from Schematic" + DRC if not.

## 7. Working Context

**Branch:** `master` (also the main/PR branch).

**Uncommitted work as of 2026-07-26:**
- Modified: `README.md`, `rover_led_udp_controller_hw/MCU&LEDS.kicad_sch` (~6.1k insertions /
  4.2k deletions), `fp-lib-table`, `sym-lib-table`
- Untracked: `.mcp.json`, `Libs/ESP32-S3-DevKit-LiPo_Rev_B.kicad_sym`

The library-table edits and the new ESP32-S3 DevKit symbol were added alongside the schematic
rework — they are one logical change set.

**Commit history:** `2891a78 Rename project` ← `63b0402 Final version` ← `aa50c8e Update`
← `b555acb First commit` ← `78eaea4 Initial commit`.
"Final version" refers to the fabricated 2025-06 revision; work since then is a new revision.

## 8. Activity Log

<!-- Newest first. Format: ### YYYY-MM-DD — summary, then bullets of what changed and why. -->

### 2026-07-26 — Added CLAUDE.md
- Created this file to hold project context, conventions, tasks, and activity history.
- Captured repo layout, schematic hierarchy, hardware block breakdown, and KiCad MCP tooling notes.
