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

Open schematic findings from the 2026-07-26 review, highest value first. Full write-up in §9.

- [ ] **CRITICAL — the sheet hierarchy UUIDs are corrupt.** `MCU&LEDS.kicad_sch` declares the *same*
      file UUID as `TOP.kicad_sch` (`a123a8db-a151-4387-aa00-5e9926b56231`), and every symbol
      instance path in it is rooted at `a123a8db-…` instead of the real root sheet
      (`b53082d8-eb8a-450c-893c-89cd915c3254`). The sheet-symbol UUIDs (`ed65724d-…` for TOP,
      `dc69146b-…` for MCU&LEDS) are correct — only the root element and the duplicate file UUID are
      wrong. **Consequence:** `kicad-cli sch export netlist` yields only **14 nets / 149 nodes** for
      the whole design — every pin-to-pin wire-only net is missing, and only label- and
      power-symbol-scoped nets survive. ERC reports 36 `wire_dangling`. Pre-existing as of
      `adb640e`. Fix before trusting any exported netlist, BOM, or PCB update.
      **Do not fix by re-annotating** — repair the UUIDs.
- [x] ~~`R14` across +3.3V→GND with `IC2` `OE` hard-tied~~ — **closed 2026-07-26** as a side effect
      of the 74AHCT125 swap; the 3.3 V side of the translator no longer exists, so `R14` and the OE
      node were removed.
- [x] ~~TXS0108E is the wrong translator family~~ — **done 2026-07-26**, replaced by `U5` 74AHCT125.
- [ ] **All bulk electrolytic was removed from the +24V input** (commit `036b3ef`): now `C7`,`C8`,`C9`
      4.7 µF/50 V X7R 1206 + `C10` 100 nF + `C11` 1 nF, with the 100 µF electrolytic gone. Ceramic is
      the right choice for ripple current, but an undamped all-ceramic input rings against the supply
      harness inductance on hot-plug. `D3` clamps, so it is not defenceless — but every hot-plug now
      dumps into the TVS. Restore bulk, or add an R+C damping leg.
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
- `U2.PG` (power-good) left NC — a free supply-health signal into a spare GPIO, unused.
- **Repo portability**: the *global* `sym-lib-table` maps `MechatronicsAcademy` into a **different
  project's** folder (`rover_bms_ble_controller_hw`). The project-local table shadows it so this
  machine is fine, but a fresh clone inherits a wrong-version library. Same class of problem as the
  hardcoded paths in `.mcp.json`.
