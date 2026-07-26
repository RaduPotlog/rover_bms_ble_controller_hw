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

**Input supply — write this down, it decides part choices.** The `+24V_PAD` is fed from a
**7S Li-ion pack: 25.9 V nominal, 29.4 V fully charged** (4.2 V/cell). It is *not* a regulated
24.0 V rail. This matters more than it looks:

- any TVS on this rail needs a standoff **≥30 V** — which is why `D3` is SMBJ33A and why the
  lower-clamping alternatives are unusable (F-21)
- `U2` LMR51450 has a 38 V absolute maximum, so a charged pack leaves only ~8.6 V of headroom
- if `PWR_SENS` is ever moved from +5V to the input rail (F-03), the divider must survive 29.4 V

Assuming "24 V means 24.0 V" already produced one wrong recommendation. Don't repeat it.

Designators below are the **post-`adb640e` annotation**. That commit renumbered every part, so any
older note, netlist, or the existing PCB refers to different designators — see §7.

| Block | Parts | Notes |
|---|---|---|
| MCU | `U3` ESP32-S3-WROOM-1-N8R8 | 8 MB flash / 8 MB PSRAM, Wi-Fi for the UDP link |
| Buck 1 (24 V→5 V) | `U2` LMR51450SDRRR + `L2` 4.7 µH | `R3`/`R8` FB divider, `R4`+`C17` feedforward, `C2` bootstrap, `R5` 31.6k on RT |
| Buck 2 (5 V→3.3 V) | `U4` SY8089AAAC (SOT23-5) + `L3` 2.2 µH | `R9`/`R10` FB divider |
| Level shift | `IC2` TXS0108E (one only) | 3V3 ↔ 5 V, but only 2 of 8 channels used (A1/B1 = CLK, A2/B2 = MOSI) |
| Input protect | `Q2` P-MOS 60 V SOT-23, `F2` 2 A/1206, `D2` 15 V zener, `R6` 100k, `D3` SMBJ33A TVS | reverse-polarity + overcurrent + surge; TVS and bulk sit on the protected side |
| USB | `USB-C2` USB2.0 Type-C (one only), `R15`/`R16` 5.1k CC pulldowns, `D4` 1N5819 VBUS OR-ing | native USB on GPIO19/20 — no UART bridge, no auto-reset circuit needed |
| Controls | `RST2` → `ESP_EN`, `BUT2` → GPIO0 via `R13` 100 Ω | reset + boot button |
| On-board LEDs | `D5`–`D14`, 10× SK9822 (`LED:APA102` symbol) | two chains of 5, see below |
| LED I/O | `J10`–`J17` | `DO_OUT`/`CO_OUT`/`DI_IN`/`CI_IN` (+ `_2`) |
| Power I/O | `J2` +24V_PAD, `J3`/`J4` IN/OUT +5, `J5`–`J9` GND pads | pass-through 5 V and GND |

**LED chain topology** (confirmed intentional, not a wiring bug):

```
MCU GPIO12 ─┐                  ┌─> D5→D6→D7→D8→D9 ──> J10/J11 (DO_OUT/CO_OUT)
            ├─> IC2 (3V3→5V) ──┤                              │
MCU GPIO11 ─┘                  └─ LED_SPI_CLK / LED_SPI_MOSI  │ external jumper
                                                              v
             J15/J17 (DI_IN_2/CI_IN_2) ──> D14→D13→D12→D11→D10 ──> J14/J16 (_2 outs)
```

Chain 2 has **no on-board driver by design** — it is a pass-through daisy-chained externally from
chain 1's output. Do not "fix" it.

Datasheets for `SY8089AAAC`, `LMR51450`, and JLC parts `C2829089` / `C2913201` are in
`rover_led_udp_controller_hw/docs/`. Note both regulator PDFs have CID-encoded body text and do
**not** yield to naive text extraction — V_ref and abs-max figures must be read by eye.

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

**Do not trust the MCP server's ERC.** `run_erc` / `get_erc_violations` reported 36 `wire_dangling`
errors with no locations on a schematic that has **zero** dangling wire ends. They are false
positives from a crude heuristic. Likewise `extract_power_domains` returns "no power components
found" and `analyze_hierarchical_nets` returns "no nets" on this project, and
`list_schematic_nets` on a sub-sheet only sees labelled nets. The component lister is reliable;
the analysis tools are not.

### Getting real connectivity
Two options, in order of preference:

1. **KiCad's own ERC** (Eeschema → Tools → Electrical Rules Checker). Requires the GUI; the project
   must not be locked.
2. **Parse the schematic directly.** This works headlessly and is what the July 2026 reviews used.
   The recipe, which is non-obvious and worth repeating:
   - Pin coordinates come from `lib_symbols` pin offsets pushed through the instance's
     `(at x y angle)` + `(mirror x|y)`: `screen = (X + px·cos−py·sin, Y − (px·sin+py·cos))`,
     mirror applied *before* rotation. Note schematic Y is inverted relative to library Y.
   - **Sub-symbols named `NAME_0_*` are unit 0 = common to all units.** Filtering on
     `unit == instance_unit` alone silently drops every pin of `U2`, `U3` and most of `IC2`.
     Accept `unit == instance_unit or unit == 0`.
   - Union-find over wire endpoints, plus endpoint-on-segment tests for T-junctions.
   - Then merge by **label text** and by **power-symbol value** — same-named local labels connect
     across the sheet, and geometry alone will not join them.
   - Sanity check: pins landing exactly on a wire endpoint should equal (total pins − NC count).
     Currently **279 on-endpoint + 44 NC = 323**. If that identity breaks, the parse is wrong.
   - `python -c`/heredoc note: backslashes get collapsed in transit, so build regexes containing
     `\` via `chr(92)` rather than writing them literally.

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

Open schematic findings, highest value first. Full write-up in §9.

- [x] ~~**CRITICAL — sheet hierarchy UUIDs corrupt**~~ — **fixed 2026-07-26 (third attempt, verified).**
      **Ground truth first:** KiCad's own demo `share/kicad/demos/complex_hierarchy` shows the correct
      format — a subsheet symbol's path is `/<ROOT FILE uuid>/<sheet-symbol uuid>…`, and the
      subsheet's *own* file uuid appears in **no** path. That proved `a123a8db`
      (`TOP.kicad_sch`'s file uuid) being the root element was definitively wrong, and that changing
      TOP's file uuid is harmless.
      **What was different this time:** the first two attempts only rewrote the paths, leaving
      `a123a8db` alive as TOP's file uuid for KiCad to resurrect. This time TOP also got a fresh uuid
      (`38556dce`), so **`a123a8db` no longer exists anywhere in the project**.
      **Result: ERC 36 → 0 errors; netlist 14 → 84 nets / 264 nodes.** LED chain connected, all 13
      `U2` pins, all 14 `U5` pins, GPIO0 boot net and the +24V bank all resolve.
      **Still to confirm:** it survived `kicad-cli`, but both earlier regressions came from a KiCad
      **GUI** save, which cannot be tested here. Open the project, save, and re-run
      `kicad-cli sch export netlist` — it must still say 84 nets.

- [ ] ~~superseded~~ **CRITICAL — sheet hierarchy UUIDs corrupt. (history of the two failed attempts)**
      **A file-level patch does NOT hold.** Applied and verified clean (ERC 0 / 84 nets) twice; both
      times a KiCad session rewrote the instance paths back to the `/a123a8db-…` root, and the second
      regression is committed in `e6e960b`. Signature both times: all three sheets **plus
      `.kicad_pro`** rewritten within seconds — and no script here writes `.kicad_pro`.
      `kicad-cli sch erc` / `export netlist` were tested directly and do **not** revert it, so the
      agent is the KiCad GUI.
      **Untested hypothesis for a durable fix:** `a123a8db` is `TOP.kicad_sch`'s own file UUID, and
      the dead `(project "BM")` entry suggests TOP was once a project root. Giving **TOP** a fresh
      UUID as well — so `a123a8db` exists nowhere — may stop KiCad resurrecting it. Must be verified
      across a real KiCad session, not just with `kicad-cli`.
      **The check, after every KiCad session:** `kicad-cli sch export netlist` must report **84 nets**.
- [ ] **Verify the PCB was not updated from the broken netlist.** `e6e960b` rewrote 4 878 lines of
      `.kicad_pcb` while the design exported only 14 nets. If any of that came from "Update PCB from
      Schematic", the LED chain, the buck's SW/BOOT/FB/RT nodes, the MOSFET gate network and the USB
      CC resistors arrived unconnected. Check the ratsnest before more layout work.

- [x] ~~**CRITICAL — sheet hierarchy UUIDs corrupt**~~ — first repair attempt 2026-07-26. Root cause was
      commit `2891a78 "Rename project"`: the root sheet symbol carried *two* instance entries —
      `(project "BM")` with the correct root UUID `b53082d8-…`, and
      `(project "rover_led_udp_controller_hw")` with the wrong one `a123a8db-…` (which is
      `TOP.kicad_sch`'s file UUID). KiCad looked up the live project name, got the wrong root, and
      hierarchy resolution collapsed. Repair: dropped the dead `"BM"` entry, re-rooted all 150
      instance paths onto `b53082d8-…`, and gave `MCU&LEDS.kicad_sch` a unique file UUID (it had
      been a duplicate of TOP's). **ERC 36 → 0 violations; netlist 14 → 84 nets / 256 nodes.**
- [x] ~~`R14` across +3.3V→GND with `IC2` `OE` hard-tied~~ — **closed 2026-07-26** as a side effect
      of the 74AHCT125 swap; the 3.3 V side of the translator no longer exists, so `R14` and the OE
      node were removed.
- [x] ~~TXS0108E is the wrong translator family~~ — **done 2026-07-26**, replaced by `U5` 74AHCT125.
- [x] ~~**HIGH — the +24V input has lost all ceramic bulk**~~ — **fixed 2026-07-26.** Rebuilt as a
      hybrid bank: `C7`,`C25`,`C26` = 4.7 µF/50 V X7R 1210 (`C_1210_3225Metric`) carry the ~2 A RMS
      ripple, `C6` = 100 µF/50 V (`CP_Elec_8x10.5`) is the damping element, `C8` 100 nF + `C9` 1 nF
      unchanged. `C25`/`C26` placed on the existing 8.89 mm cap pitch at x = 88.90 / 80.01, extending
      the bank left of `C6` along the same rails. **Layout:** ceramics closest to `U2` VIN/PGND,
      electrolytic behind; `C6` grows from 4×5.7 mm to 8×10.5 mm.

- [ ] **⚠ OPERATIONAL — the F-16 hierarchy repair was silently reverted once mid-session.** After it
      was applied and verified, all three sheets reverted to the broken `a123a8db-…` root and the
      netlist collapsed back to 14 nets. Tell-tale: `.kicad_pro` was rewritten at the same instant,
      and **no script here writes that file** — so a KiCad process did it. A controlled test showed
      `kicad-cli sch erc` / `export netlist` do **not** revert it, so the likely cause is a KiCad GUI
      session flushing stale in-memory state. **Close KiCad before any scripted edit**, and after any
      KiCad session re-check with `kicad-cli sch export netlist` — it must say **84 nets**, not 14.
      The repair is uncommitted; committing makes it durable.
- [x] ~~Polarized capacitors on a non-polarized symbol~~ — **fixed 2026-07-26.** `C6`,`C7`,`C12`,
      `C13`,`C18` now use `Device:C_Polarized_Small`, chosen because its pins sit at exactly
      `(0, ±2.54)` — identical to the old `C` symbol, so the swap moved nothing. KiCad's netlist
      confirmed pin 1 was already on the positive rail for all five, so polarity is correct.
- [x] ~~`C19`/`C22` ceramic-vs-tantalum contradiction~~ — **fixed 2026-07-26**, re-footprinted to
      `Capacitor_SMD:C_0603_1608Metric` to match their `22uF/10V/X5R/0603` value and the identical
      `C3`/`C17` elsewhere. They are non-polarized again, so they need no polarized symbol.
- [x] ~~Re-enable the `footprint_filter` ERC rule~~ — **done 2026-07-26.** Final severities:
      `footprint_filter` = **warning**, `single_global_label` = **warning**,
      `four_way_junction` = **ignore** (the decoupling-bank topology inherently creates 4-way
      junctions; 11 permanent warnings would mask real ones), `simulation_model_issue` = **ignore**
      (no simulation models in this design). Enabling `footprint_filter` produced **zero** capacitor
      violations — the F-18/F-19 work holds.
- [ ] **Known ERC baseline: 18 `footprint_link_issues`** — surfaced by enabling `footprint_filter`.
      All are *intentional* footprint choices on generic symbols: `J2`–`J17` are test pads on a
      `Conn_01x01_Pin` symbol whose filter is `Connector*:*_1x??_*`, plus `D4` and `U4` on OLIMEX
      footprints. Not defects, but 18 standing warnings will mask a real #19. Proper fix is a
      test-point symbol for `J2`–`J17`.
- [x] ~~**F-21 — the TVS does not protect `U2` within its ratings**~~ — **assessed 2026-07-26,
      accepted, no change. `D3` stays SMBJ33A.**
      Facts: `U2` V_IN abs max **38 V** (recommended 4.0–36 V); `D3` SMBJ33A = 33 V standoff,
      ~36.7 V breakdown, **53.3 V clamping at full rated surge**.
      **The supply is a 7S Li-ion pack — 29.4 V fully charged** (see §3). That requires a standoff
      ≥30 V, so the lower-clamping parts an earlier note suggested are all unusable:

      | Part | Standoff | vs 29.4 V pack | V_C |
      |---|---|---|---|
      | SMBJ24A | 24.0 V | conducts hard — pack is ~3 V above its 26.7 V breakdown | 38.9 V |
      | SMBJ26A / 28A | 26 / 28 V | below pack maximum — conduct | 42.1 / 45.4 V |
      | SMBJ30A | 30.0 V | only 0.6 V (2 %) margin — too thin | 48.4 V |
      | **SMBJ33A** | 33.0 V | **3.6 V (12 %) margin — correct** | 53.3 V |

      **Why no substitution can close the gap:** the SMBJ clamping ratio is ~1.6×, so clamping under
      38 V needs a standoff below ~23.5 V — far under the pack voltage. **Do not re-litigate this
      with another SMBJ part.** Closing it properly needs *series impedance* between the TVS and
      `U2` VIN, which is deliberately out of scope.
      **Residual risk, accepted:** V_C 53.3 V is specified at the full rated 11.3 A pulse. Ordinary
      hot-plug and switching events are attenuated by `F2`, harness inductance and the 100 µF bulk
      (`C7`, added under F-17), and clamp well below 53 V. Only a genuinely large surge is exposed.
- [ ] **Revisit F-21 if motors share the 24 V bus.** Regen and back-EMF can push the bus well above
      29.4 V — exactly the case the TVS cannot cover. If the drives are on this bus, the
      series-impedance option is worth reopening.
- [ ] **`PWR_SENS` ADC divider**: `R11` 2.2M / `R12` 470k → ~387 kΩ source impedance into GPIO5,
      with no filter cap. Add 100 nF to GND and drop the divider ~10×. Also decide whether it should
      tap +24V instead of the regulated +5V, which carries little information.
- [ ] **No ESD protection on USB-C** (D+/D−, CC1/CC2, VBUS all bare).
- [ ] **Verify `U2.EN` abs-max** — EN is tied directly to +24V. Read `docs/lmr51450.pdf` by eye
      (text extraction does not work); add a divider if EN is rated below V_IN.
- [ ] **`Q2` has no MPN** (`P-MOS 60V/SOT-23`), so R_DS(on) is unknown and SOT-23 thermal headroom
      at ~1.2 A input is unverified.
- [ ] GPIO0 button has no external pull-up / debounce cap — relies on the internal pull-up only.
- [ ] Net names contain `\` and `()` (`GPIO11\FSPID(SPI3_MOSI)` etc.) — can break CAM/BOM tooling.
- [ ] PCB is now **fully desynced** — `adb640e` renumbered every designator. Deliberately out of
      scope as of 2026-07-26; decide when to re-sync.

## 7. Working Context

**Branch:** `master` (also the main/PR branch). Working tree clean as of 2026-07-26.

**Commit history:** `adb640e Polarity protection and TSV protection` ← `2891a78 Rename project`
← `63b0402 Final version` ← `aa50c8e Update` ← `b555acb First commit` ← `78eaea4 Initial commit`.
"Final version" refers to the fabricated 2025-06 revision; everything after is a new revision.

**About `adb640e`** — the commit message is misleading. It swept up the entire previously-uncommitted
rework (schematic, both lib tables, `.mcp.json`, the ESP32-S3 DevKit symbol, `CLAUDE.md`) *and*
performed a full re-annotation *and* changed the sheet from A3 to A2. The polarity/TVS circuit it
names was already present in the working tree beforehand. **Electrically the commit changed
nothing** — a net-by-net comparison before and after is identical (94 nets, 323 pins, 286 wires,
67 junctions, 44 NC on both sides).

**Designator remap introduced by `adb640e`** — needed to read any older note or the existing PCB:

| Old | New | | Old | New |
|---|---|---|---|---|
| `U3` SY8089 | `U4` | | `D1` zener | `D2` |
| `U4` ESP32-S3 | `U3` | | `D2` Schottky | `D4` |
| `L2` ↔ `L3` swapped | | | `D13` TVS | `D3` |
| `Q1` / `F1` | `Q2` / `F2` | | `D3`–`D12` LEDs | `D5`–`D14` |
| `R12` (OE resistor) | `R14` | | `R15` (GPIO0) | `R13` |

Every prefix now starts at 2 (`C2…C24`, `R2…R16`, `D2…D14`, `U2…U4`, `J2…J17`). Uniform, but there
is no `U1`/`C1`/`R1`/`D1` — fine if deliberate.

## 8. Activity Log

<!-- Newest first. Format: ### YYYY-MM-DD — summary, then bullets of what changed and why. -->

### 2026-07-26 — F-05 closed, F-20 and F-06 fixed
- **F-05 CLOSED — no change needed.** The LMR51450 PDF *is* extractable after all: its fonts are CID
  encoded, so raw stream text is garbage, but decoding via each font's **`/ToUnicode` CMap**
  (`beginbfchar` / `beginbfrange`) recovers clean text. Recipe worth reusing for `SY8089AAAC.pdf`.
  Verdict, quoted: abs max **"EN to PGND −0.3 to V_IN +0.3 V"**, recommended **"EN to PGND: V_IN"**,
  and the pin description says **"Can be connected to VIN."** Tying EN to +24V is sanctioned.
  User independently confirmed.
- Same decode yielded `V_IN` abs max **38 V** and recommended range **4.0–36 V** → new TVS finding
  in §6.
- **F-20 fixed** — ERC rules retuned, see §6.
- **F-06 fixed** — added `R16` 10 kΩ to +3.3V and `C24` 100 nF to GND on the GPIO0 node, ahead of the
  existing `R13` 100 Ω series resistor. Final topology, verified:
  `+3.3V ─[R16 10k]─┬─ GPIO0 (U3.27)`, `├─[C24 100nF]─ GND`, `└─[R13 100Ω]─ BUT2 ─ GND`.
- **Two placement traps hit while adding parts** — both worth remembering:
  1. Cloning a symbol block and rewriting `(at …)` with a regex that expects `(unit` to follow
     **fails when the source has `(mirror x)` in between** — the clone silently keeps the original's
     position and its pins merge with it. Always re-verify the clone's computed pin coordinates.
  2. The `R` symbol is **horizontal** in library coords (pins at `(±3.81, 0)`); a vertical resistor
     needs angle **270**, not 0.
- State after: **78 components, 323 pins, 289 on endpoints + 34 NC, 289 wires, 84 nets, parity OK.**
  ERC = 18 `footprint_link_issues` (known baseline) + 36 `wire_dangling` (F-16, still open).

### 2026-07-26 — F-16 FIXED (third attempt) — netlist complete for the first time
- **Method that worked: get ground truth before patching.** KiCad's bundled demo
  `share/kicad/demos/complex_hierarchy` is a 2-level hierarchy and shows the canonical format:
  `path = /<ROOT FILE uuid>/<sheet-symbol uuid>/…`, and a **subsheet's own file uuid never appears in
  any path**. That single observation settled a question two earlier attempts had guessed at.
- **Why the earlier attempts failed:** they rewrote the paths but left `a123a8db` alive as
  `TOP.kicad_sch`'s file uuid. A KiCad GUI save resurrected it. This time TOP was given a fresh uuid
  (`38556dce`) so the stale value exists **nowhere** in the project.
- **Verified:** ERC **36 → 0** errors (only the 18 known `footprint_link_issues` remain);
  netlist **14 → 84 nets / 264 nodes**; LED chain connected; 13/13 `U2` pins; 14/14 `U5` pins;
  GPIO0 boot net `{C24.1, R13.2, R14.2, U3.27}`; +24V bank `C5…C10` all present.
- **Open question:** only `kicad-cli` could be tested. Both earlier regressions came from a GUI save.
  Confirm by opening in KiCad, saving, and re-running `kicad-cli sch export netlist` → 84 nets.

### 2026-07-26 — Seventh review (post-`e6e960b`): F-16 reopened, circuit work intact
- **F-16 regressed a second time and is committed.** ERC back to 36, netlist back to 14 nets. See §6.
  My earlier "closed" claim was wrong — the correct status was "patched, unverified across a KiCad
  session". Do not mark a file-level metadata repair as closed until it survives one.
- **All circuit-level fixes survived**, verified designator-independently (KiCad re-annotated again,
  so `C25`/`C26` folded into the C-series): 24 V bank = 3× 4.7 µF X7R 1210 (`C5`,`C6`,`C8`) +
  100 µF `CP_Elec_8x10.5` (`C7`) + 100 nF + 1 nF; polarized audit **zero mismatches** (4 polarized
  footprints, all on `C_Polarized_Small`); no tantalum lands remain; no bad net names; `U5`
  74AHCT125; `L2` = `4.7uH/MWSA1003S`; library tables still pinned.
- Drawing is sound: **317 pins, 283 on endpoints + 34 NC, 282 wires, 84 nets, 0 dangles.**
- **Audit designator-independently from now on** — this project has been re-annotated four times.
  Cite values, footprints and net membership, not `Cnn` numbers, when checking whether a fix held.

### 2026-07-26 — F-17 fixed: 24 V input rebuilt as a hybrid bank
- `C6` → 100 µF/50 V `CP_Elec_8x10.5` (damping); `C7` → 4.7 µF/50 V X7R 1210 and back to the
  non-polarized symbol; **new `C25`, `C26`** = 4.7 µF/50 V X7R 1210. `C8`/`C9` untouched.
- Rails re-segmented at x = 80.01 and 88.90 with junctions, matching the existing 8.89 mm cap pitch.
- Verified: **ERC 0 violations, netlist 84 nets / 260 nodes** (+4 nodes = two new caps × two pins),
  LED chain intact, all 13 `U2` pins present. 80 components total (76 on the main sheet + 4 holes).
- **Mid-session the F-16 repair reverted** and had to be re-applied — see the ⚠ item in §6. This is
  the single most important operational fact in this file right now.

### 2026-07-26 — Safe-set fixes applied (F-16, F-18, F-19, F-08, F-10, F-14)
- **F-16 hierarchy repair** — see §6. **ERC 36 → 0; netlist 14 → 84 nets / 256 nodes.**
- **F-18 / F-19** — five electrolytics moved to `Device:C_Polarized_Small`; `C19`/`C22` returned to
  ceramic 0603.
- **F-08** — six net names sanitised (12 label occurrences):
  `GPIO0_BUT1`, `GPIO5_PWR_SENS`, `GPIO11_FSPID_SPI3_MOSI`, `GPIO12_FSPICLK_SPI3_CLK`,
  `GPIO19_USB_DM`, `GPIO20_USB_DP`. No backslashes or parentheses remain in any user net name.
- **F-10** — `L2` value → `4.7uH/MWSA1003S` (series taken from its own footprint). **Current rating
  still unspecified** — needs the real part.
- **F-14** — pinned the KiCad-stock libraries explicitly in the project tables: `74xx`, `LED` in
  `sym-lib-table`; `Package_SO`, `Capacitor_SMD`, `Resistor_SMD`, `LED_SMD`, `Diode_SMD`, `Fuse`,
  `Inductor_SMD`, `Package_TO_SOT_SMD` in `fp-lib-table`. The repo no longer depends on the machine's
  global table for stock parts. **Still machine-dependent:** the `OLIMEX_*` libraries live in
  `C:/Users/potlo/Desktop/olimex_kicad_libs/`, outside the repo — vendoring them into `Libs/` is a
  separate call.
- **Verification** — the pre-cosmetic and post-cosmetic netlists were compared node-set by node-set:
  **84 nets, 84 identical, 0 added, 0 removed.** Connectivity provably unchanged by F-18/19/08/10.
- Not touched: the PCB.

### 2026-07-26 — Fourth schematic review (post-`22ae4da` "PCB redesign")
- Current state: **74 components, 313 pins, 84 nets drawn, 277 wires, 34 NC, 0 geometric dangles.**
  Parity 279 + 34 = 313 holds.
- **The hierarchy UUID defect is unchanged for a third consecutive review** — and `22ae4da` starts a
  PCB redesign, so it is now actively blocking: "Update PCB from Schematic" would import **14 nets**.
- +24V bank swung back to all-electrolytic; two new capacitor findings (polarized-symbol mismatch,
  ceramic-vs-tantalum contradiction). All three recorded in §6.
- `U5` 74AHCT125 re-verified intact; F-01 / F-02 / F-11 remain closed.

### 2026-07-26 — Third schematic review (post-`78e8561`)
- Current state: **75 components, 316 pins, 84 nets drawn, 280 wires, 34 NC, 0 geometric dangles.**
  Parse parity 282 + 34 = 316 holds, so the drawing is coherent.
- **The hierarchy UUID corruption survived a KiCad GUI save and two commits.** Opening and saving
  does not repair it. KiCad still exports only **14 nets / 145 nodes**, ERC still 36 `wire_dangling`.
- `036b3ef` rebuilt the +24V bank to all-ceramic and dropped the 100 µF bulk — new finding, see §6.
- F-01 and F-02 confirmed closed; F-11 (spare translator channels) obsolete — a quad buffer with two
  spare gates replaced an 8-bit translator with six.
- Every remaining finding re-verified against the current designators: `R9`/`R12` are now the
  PWR_SENS divider, `R13` the GPIO0 series, `R14`/`R15` the USB CC pair, `R10`/`R11` the SY8089 FB
  divider, `R3`/`R8` the LMR51450 FB divider.

### 2026-07-26 — F-02 implemented: TXS0108E → 74AHCT125
- Replaced `IC2` (TXS0108E, TSSOP-20) with **`U5` 74AHCT125** (`74xx:74AHCT125`,
  `Package_SO:SOIC-14_3.9x8.7mm_P1.27mm`), a single-supply 5 V quad buffer with TTL inputs.
- Gate A = CLK, gate B = MOSI, both mirrored so the input sits on the right and the existing label
  wires are reused unchanged. Unused gates C/D have `OE` and `A` tied to GND and `Y` no-connected —
  no floating CMOS inputs. Power unit: VCC → +5V, GND → GND.
- `R14` and the OE-to-+3.3V node deleted with it, closing the F-01 finding.
- The old VCCA decoupling (`C23` 22 µF, `C24` 100 nF) was **kept on +3.3V** as rail decoupling
  rather than deleted, so the BOM count is unchanged. For layout, one 100 nF belongs at `U5` pin 14.
- Verified two ways: the geometric parse (parity 286 + 34 NC = 320 pins, 0 dangling) **and**
  `kicad-cli sch export netlist`, which resolves all 14 `U5` pins onto the intended nets.
- ERC is unchanged by the edit: **36 violations before, 36 after**, all pre-existing `wire_dangling`.
- Not touched: the PCB, by explicit instruction.

### 2026-07-26 — Correction: the dangling-wire reports were real
- Running KiCad's own ERC via `kicad-cli` on the **unmodified** commit `adb640e` produced exactly
  **36 `wire_dangling` errors** — the same count the MCP server reported. Calling them false
  positives in the first two reviews was **wrong**.
- Root-caused to the hierarchy UUID collision now tracked at the top of §6. The geometric parse and
  KiCad disagreed because the parse works inside a single sheet file and never consults the sheet
  path, so it cannot see hierarchy corruption. Both were right about different things — and neither
  alone is sufficient. **Cross-check every future connectivity claim against
  `kicad-cli sch export netlist`, not just the parse.**

### 2026-07-26 — Second schematic review (post-`adb640e`)
- Re-derived the netlist after the re-annotation. **Result: no electrical change** vs. the first
  review; only designators, sheet size, and file ordering moved.
- Every finding from the first review still stands, remapped to the new designators (see §6).
- Corrected an error in the first review: the +5V and +3.3V rails **do** carry ceramic bulk
  (`C19`, `C22` and `C3`, `C23`, all 22 µF X5R). The earlier claim of "no ceramic bulk on +5V" was
  wrong — it was built from a partial reading rather than from net membership. The residual point is
  much narrower (X5R DC-bias derating, and 47 µF/50 V electrolytics as the drawn output caps).

### 2026-07-26 — First schematic review
- Reconstructed the netlist by direct parse (see §4) because the MCP ERC proved unreliable.
- Verified correct: input protection topology, LMR51450 vs. TI reference design, USB-C sink
  advertisement, ESP32-S3 reset RC and decoupling, translator rail orientation, strapping pins.
- Raised the findings now tracked in §6.

### 2026-07-26 — Added CLAUDE.md
- Created this file to hold project context, conventions, tasks, and activity history.
- Captured repo layout, schematic hierarchy, hardware block breakdown, and KiCad MCP tooling notes.

## 9. Review Reference

Longer-form write-ups, kept out of the sections above so they stay scannable.

### Verified-correct circuitry (checked 2026-07-26, unchanged by `adb640e`)

- **Input protection.** `J2 → F2 (2 A) → Q2.D`, `Q2.S → +24V`, gate to GND via `R6` 100k, clamped by
  `D2` 15 V zener (cathode on the source side). Drain-on-input / source-on-load is the correct
  orientation; a 15 V clamp suits a ±20 V V_GS part. `D3` (SMBJ33A) and all bulk capacitance sit on
  the protected side, downstream of the fuse — so a TVS failure blows `F2` rather than the source.
- **LMR51450 matches TI's reference topology**, including the `R4` (1k) + `C17` (33 pF) feedforward
  that corresponds to `R_FF`/`C_FF` in the datasheet's typical-application figure. FB divider
  `R3` 100k / `R8` 19.1k → V_out = V_ref × 6.236 (**4.99 V for a 0.8 V reference**).
- **SY8089** FB divider `R9` 220k / `R10` 49.9k → V_ref × 5.409 (**3.245 V for a 0.6 V reference**),
  ~1.7 % low but well inside the ESP32-S3's 3.0–3.6 V window.
- **USB-C sink**: `R15` and `R16` give CC1 and CC2 their own separate 5.1k to GND (not a shared
  resistor). VBUS is OR'd into +5V through `D4`, so USB-only operation works without back-feeding
  the buck.
- **ESP32-S3 support**: EN = `R7` 10k + `C18` 1 µF + `RST2`; decoupling `C3` 22 µF + `C4` 100 nF at
  the 3V3 pin; GND on pins 1, 40 and EPAD 41. Strapping pins acceptable (GPIO0 button, GPIO3 pulled
  up via `R2`, GPIO45/46 floating on internal pull-downs).
- **Translator rail orientation correct**: VCCA = +3.3V (MCU side), VCCB = +5V (LED side).
- **Connectivity clean**: 0 dangling pins, 0 dangling wire ends, 44 NC flags placed.

### Lower-priority notes

- Capacitance per rail, as wired (schematic-level, ignoring placement):
  - `+24V` — `C7` 100 µF elec, `C8`/`C9` 4.7 µF elec, `C5`/`C6` 4.7 µF X7R 1210, `C10` 100 nF, `C11` 1 nF
  - `+5V` — `C14`/`C15` 47 µF elec, `C19`/`C22` 22 µF X5R, `C12`/`C16`/`C21` 100 nF, `C13` 1 nF
  - `+3.3V` — `C20` 47 µF elec, `C3`/`C23` 22 µF X5R, `C4`/`C24` 100 nF
  Ceramic bulk is present on every rail. Worth watching: 22 µF/10 V X5R 0603 loses a large fraction
  of its value at 5 V DC bias, and the caps drawn *at* the buck outputs are the electrolytics.
- Over-rated / inconsistent passives: `C20` is 47 µF/**50 V** on a 3.3 V rail; `C8`/`C9` are
  electrolytic where neighbouring `C5`/`C6` are X7R; `L2`'s value is a bare `4.7uH` with no current
  or DCR rating while `L3` carries a full spec string.
- 6 of 8 translator channels unused (`IC2` A3–A8 / B3–B8 all NC).
- **F-12 — `U2.PG` (power-good) left NC. ACCEPTED 2026-07-26, no change.** It is an open-drain
  output, so floating is electrically safe. Raised only because a supply-health signal into a spare
  GPIO would have been free. Appears in the netlist as `unconnected-(U2-PG-Pad5)`; that is expected
  and not a defect. Do not re-raise.
- **Repo portability**: the *global* `sym-lib-table` maps `MechatronicsAcademy` into a **different
  project's** folder (`rover_bms_ble_controller_hw`). The project-local table shadows it so this
  machine is fine, but a fresh clone inherits a wrong-version library. Same class of problem as the
  hardcoded paths in `.mcp.json`.
