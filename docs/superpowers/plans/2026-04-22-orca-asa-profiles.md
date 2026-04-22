# OrcaSlicer ASA Profiles Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Clean up the user's OrcaSlicer printer profile and add a Polymaker Polylite ASA filament profile + a 0.20mm structural ASA process profile tuned for Voron parts on a Prusa Mk3 + Revo HF + Klipper setup in a moderately-enclosed tent.

**Architecture:** Three small JSON edits/additions under `~/Library/Application Support/OrcaSlicer/user/default/`. Each new profile ships as a `.json` settings file and a `.info` sidecar (Orca's sync metadata). Everything inherits from existing Orca system profiles; we override only what's specific to this printer, this filament, and Voron-style functional prints. Pressure advance and machine limits stay in Klipper; Orca is told not to fight.

**Tech Stack:** OrcaSlicer 2.3.2.60, Prusa system profiles as inheritance base, JSON config, macOS filesystem.

**Spec:** `docs/superpowers/specs/2026-04-22-orca-asa-profiles-design.md`

**Note on "tests":** This plan edits config files rather than writing code, so the TDD structure adapts:
- "Failing test" → documented expected state that's currently false (e.g., "profile does not exist yet").
- "Passing test" → verifying the file / G-code output / profile discovery matches the design.

---

## File Structure

Files that will be created or modified (all under `~/Library/Application Support/OrcaSlicer/user/default/`):

| Path | Responsibility | Action |
|---|---|---|
| `machine/Stephen 0.4 Klipper CHT.json` | Printer profile — machine limits, start/end G-code, sheet size | **Modify** (1 field) |
| `machine/Stephen 0.4 Klipper CHT.json.bak-YYYYMMDD` | One-shot backup copy | **Create** (Task 1) |
| `filament/Polymaker Polylite ASA @Stephen MK4IS HF0.4.json` | Filament profile — temps, flow, fan, PA-disable | **Create** |
| `filament/Polymaker Polylite ASA @Stephen MK4IS HF0.4.info` | Orca sync metadata sidecar | **Create** |
| `process/0.20mm Structural ASA @Stephen 0.4 Klipper CHT.json` | Print process — layer/infill/brim/speed/accel for Voron ASA | **Create** |
| `process/0.20mm Structural ASA @Stephen 0.4 Klipper CHT.info` | Orca sync metadata sidecar | **Create** |

OrcaSlicer user profiles live outside the git repo. The design doc and this plan are tracked in the `Klipper-Input-Shaping-MK3S-Upgrade` repo.

---

## Task 1: Back up the current printer profile

**Files:**
- Copy from: `~/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json`
- Copy to: `~/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json.bak-20260422`

- [ ] **Step 1: Verify the original exists**

Run:
```bash
ls -la "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json"
```

Expected: single line output, file present with non-zero size.

- [ ] **Step 2: Create timestamped backup copy**

Run:
```bash
cp "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json" \
   "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json.bak-20260422"
```

- [ ] **Step 3: Verify backup**

Run:
```bash
diff "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json" \
     "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json.bak-20260422"
```

Expected: no output (files identical).

---

## Task 2: Drop unused `PRINTER_MODEL="MK4IS"` from START_GCODE

**Files:**
- Modify: `~/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json` (line 75, `machine_start_gcode`)

- [ ] **Step 1: Document expected state**

Current value of `machine_start_gcode` in the profile:

```
PRINT_START EXTRUDER_TEMP={first_layer_temperature} BED_TEMP={first_layer_bed_temperature} PRINTER_MODEL="MK4IS"
```

Target value:

```
PRINT_START EXTRUDER_TEMP={first_layer_temperature} BED_TEMP={first_layer_bed_temperature}
```

The user's `PRINT_START` macro in `macros.cfg` (lines 2–37) does not read a `PRINTER_MODEL` parameter, so dropping the argument is a safe no-op at runtime and removes dead code.

- [ ] **Step 2: Confirm current state**

Run:
```bash
grep '"machine_start_gcode"' "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json"
```

Expected output (exact):
```
    "machine_start_gcode": "PRINT_START EXTRUDER_TEMP={first_layer_temperature} BED_TEMP={first_layer_bed_temperature} PRINTER_MODEL=\"MK4IS\"",
```

- [ ] **Step 3: Apply the edit**

Use the Edit tool with:
- `file_path`: `/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json`
- `old_string`: `"machine_start_gcode": "PRINT_START EXTRUDER_TEMP={first_layer_temperature} BED_TEMP={first_layer_bed_temperature} PRINTER_MODEL=\"MK4IS\"",`
- `new_string`: `"machine_start_gcode": "PRINT_START EXTRUDER_TEMP={first_layer_temperature} BED_TEMP={first_layer_bed_temperature}",`

- [ ] **Step 4: Verify the edit**

Run:
```bash
grep '"machine_start_gcode"' "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json"
```

Expected output (exact):
```
    "machine_start_gcode": "PRINT_START EXTRUDER_TEMP={first_layer_temperature} BED_TEMP={first_layer_bed_temperature}",
```

- [ ] **Step 5: Verify JSON is still valid**

Run:
```bash
python3 -m json.tool "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json" > /dev/null && echo "JSON OK"
```

Expected: `JSON OK` on stdout, no errors.

---

## Task 3: Create the Polymaker Polylite ASA filament profile

**Files:**
- Create: `~/Library/Application Support/OrcaSlicer/user/default/filament/Polymaker Polylite ASA @Stephen MK4IS HF0.4.json`
- Create: `~/Library/Application Support/OrcaSlicer/user/default/filament/Polymaker Polylite ASA @Stephen MK4IS HF0.4.info`

- [ ] **Step 1: Document expected state**

After this task: two new files exist in the filament directory, JSON is valid, and the profile inherits from `Prusa Generic ASA @MK4S HF0.4`.

- [ ] **Step 2: Confirm the profile does not exist yet**

Run:
```bash
ls "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/filament/"
```

Expected: directory is empty (no existing user filament profiles).

- [ ] **Step 3: Write the filament `.json` file**

Use the Write tool to create `/Users/stephen/Library/Application Support/OrcaSlicer/user/default/filament/Polymaker Polylite ASA @Stephen MK4IS HF0.4.json` with this exact content:

```json
{
    "type": "filament",
    "name": "Polymaker Polylite ASA @Stephen MK4IS HF0.4",
    "inherits": "Prusa Generic ASA @MK4S HF0.4",
    "from": "User",
    "filament_id": "Polymaker Polylite ASA",
    "instantiation": "true",
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "filament_vendor": [
        "Polymaker"
    ],
    "filament_type": [
        "ASA"
    ],
    "filament_density": [
        "1.07"
    ],
    "filament_cost": [
        "30"
    ],
    "nozzle_temperature": [
        "250"
    ],
    "nozzle_temperature_initial_layer": [
        "255"
    ],
    "nozzle_temperature_range_low": [
        "240"
    ],
    "nozzle_temperature_range_high": [
        "260"
    ],
    "hot_plate_temp": [
        "100"
    ],
    "cool_plate_temp": [
        "100"
    ],
    "eng_plate_temp": [
        "100"
    ],
    "hot_plate_temp_initial_layer": [
        "100"
    ],
    "cool_plate_temp_initial_layer": [
        "100"
    ],
    "eng_plate_temp_initial_layer": [
        "100"
    ],
    "filament_max_volumetric_speed": [
        "15"
    ],
    "fan_min_speed": [
        "15"
    ],
    "fan_max_speed": [
        "30"
    ],
    "overhang_fan_threshold": [
        "25%"
    ],
    "overhang_fan_speed": [
        "40"
    ],
    "close_fan_the_first_x_layers": [
        "3"
    ],
    "slow_down_layer_time": [
        "15"
    ],
    "slow_down_min_speed": [
        "10"
    ],
    "enable_pressure_advance": [
        "0"
    ],
    "filament_start_gcode": [
        "; Pressure advance managed by Klipper printer.cfg. Add SET_PRESSURE_ADVANCE ADVANCE=x.xx here after a PA tower for this filament."
    ],
    "version": "2.3.2.60"
}
```

- [ ] **Step 4: Verify the `.json` is valid JSON**

Run:
```bash
python3 -m json.tool "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/filament/Polymaker Polylite ASA @Stephen MK4IS HF0.4.json" > /dev/null && echo "JSON OK"
```

Expected: `JSON OK`.

- [ ] **Step 5: Write the filament `.info` sidecar**

First get current epoch:

```bash
date +%s
```

Note the value (e.g., `1776898550`) — use it as `updated_time` below.

Use the Write tool to create `/Users/stephen/Library/Application Support/OrcaSlicer/user/default/filament/Polymaker Polylite ASA @Stephen MK4IS HF0.4.info` with this exact content (replace `<EPOCH>` with the value you just captured):

```
sync_info = create
user_id = 
setting_id = 
base_id = GFSA04
updated_time = <EPOCH>
```

The `base_id = GFSA04` matches the `setting_id` of the parent `Prusa Generic ASA @MK4S HF0.4` profile. This mirrors the pattern of the user's existing machine profile `.info` (which has `base_id = GM001` pointing to `MyKlipper 0.4 nozzle`).

- [ ] **Step 6: Verify both files are present**

Run:
```bash
ls -la "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/filament/"
```

Expected: both `Polymaker Polylite ASA @Stephen MK4IS HF0.4.json` and `Polymaker Polylite ASA @Stephen MK4IS HF0.4.info` listed with non-zero size.

---

## Task 4: Create the `0.20mm Structural ASA` print process profile

**Files:**
- Create: `~/Library/Application Support/OrcaSlicer/user/default/process/0.20mm Structural ASA @Stephen 0.4 Klipper CHT.json`
- Create: `~/Library/Application Support/OrcaSlicer/user/default/process/0.20mm Structural ASA @Stephen 0.4 Klipper CHT.info`

- [ ] **Step 1: Document expected state**

After this task: two new files in the process directory, valid JSON, inherits from `0.20mm STRUCTURAL @MK4S 0.4` (note the uppercase `STRUCTURAL` — that is the exact system profile name).

- [ ] **Step 2: Confirm the profile does not exist yet**

Run:
```bash
ls "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/process/"
```

Expected: directory is empty (no existing user process profiles).

- [ ] **Step 3: Write the process `.json` file**

Use the Write tool to create `/Users/stephen/Library/Application Support/OrcaSlicer/user/default/process/0.20mm Structural ASA @Stephen 0.4 Klipper CHT.json` with this exact content:

```json
{
    "type": "process",
    "name": "0.20mm Structural ASA @Stephen 0.4 Klipper CHT",
    "inherits": "0.20mm STRUCTURAL @MK4S 0.4",
    "from": "User",
    "instantiation": "true",
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "layer_height": "0.20",
    "initial_layer_print_height": "0.20",
    "line_width": "0.45",
    "wall_loops": "4",
    "top_shell_layers": "5",
    "bottom_shell_layers": "5",
    "sparse_infill_pattern": "grid",
    "sparse_infill_density": "40%",
    "top_surface_pattern": "monotonic",
    "bottom_surface_pattern": "monotonic",
    "brim_type": "auto_brim",
    "brim_width": "5",
    "brim_object_gap": "0.1",
    "raft_layers": "0",
    "initial_layer_speed": "25",
    "initial_layer_infill_speed": "40",
    "outer_wall_speed": "40",
    "inner_wall_speed": "60",
    "sparse_infill_speed": "80",
    "internal_solid_infill_speed": "80",
    "top_surface_speed": "40",
    "gap_infill_speed": "40",
    "travel_speed": "250",
    "outer_wall_acceleration": "2000",
    "inner_wall_acceleration": "3000",
    "top_surface_acceleration": "2000",
    "initial_layer_acceleration": "1000",
    "default_acceleration": "3000",
    "travel_acceleration": "4000",
    "seam_position": "nearest",
    "wall_infill_order": "inner wall/outer wall/infill",
    "infill_wall_overlap": "15%",
    "version": "2.3.2.60"
}
```

- [ ] **Step 4: Verify the `.json` is valid JSON**

Run:
```bash
python3 -m json.tool "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/process/0.20mm Structural ASA @Stephen 0.4 Klipper CHT.json" > /dev/null && echo "JSON OK"
```

Expected: `JSON OK`.

- [ ] **Step 5: Write the process `.info` sidecar**

Get current epoch:

```bash
date +%s
```

Use the Write tool to create `/Users/stephen/Library/Application Support/OrcaSlicer/user/default/process/0.20mm Structural ASA @Stephen 0.4 Klipper CHT.info` with this exact content (replace `<EPOCH>`):

```
sync_info = create
user_id = 
setting_id = 
base_id = 
updated_time = <EPOCH>
```

`base_id` is intentionally empty here — the parent `0.20mm STRUCTURAL @MK4S 0.4` does not expose a `setting_id` field in its JSON. Orca will treat this as a user-created profile and generate/sync IDs on first load.

- [ ] **Step 6: Verify both files are present**

Run:
```bash
ls -la "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/process/"
```

Expected: both `.json` and `.info` files present with non-zero size.

---

## Task 5: End-to-end verification in OrcaSlicer

**Files:** None modified. This task verifies the previous four.

- [ ] **Step 1: Document expected state**

After this task: OrcaSlicer loads all three profiles without error, the filament and process profiles appear in their dropdowns specifically when `Stephen 0.4 Klipper CHT` is selected, and a sliced test part produces the expected G-code.

- [ ] **Step 2: Close and reopen OrcaSlicer**

Manual step. Fully quit OrcaSlicer (⌘Q) and relaunch. OrcaSlicer only reads user profiles at startup.

- [ ] **Step 3: Verify the printer profile still loads**

Manual step in OrcaSlicer UI:
- Open the Printer dropdown (top of the right-hand settings panel).
- Select `Stephen 0.4 Klipper CHT`.
- Expected: no error dialog; printer loads normally.

- [ ] **Step 4: Verify the filament profile is visible and selectable**

Manual step:
- Open the Filament dropdown.
- Expected: `Polymaker Polylite ASA @Stephen MK4IS HF0.4` appears in the list.
- Select it.
- Expected: no error, nozzle temperature field on the Filament settings tab shows **250**, initial-layer nozzle temperature shows **255**, bed temp (textured PEI column) shows **100**, max volumetric speed shows **15**.

- [ ] **Step 5: Verify the process profile is visible and selectable**

Manual step:
- Open the Process dropdown.
- Expected: `0.20mm Structural ASA @Stephen 0.4 Klipper CHT` appears in the list.
- Select it.
- Expected: no error. In Quality settings, Layer Height shows **0.20** and Wall Loops shows **4**. In Strength settings, Sparse Infill Pattern shows **Grid** and Density shows **40%**. In Others / Support, Brim type shows **Auto brim**.

- [ ] **Step 6: Slice a test part and inspect G-code**

Manual step:
- Load any small STL (a 20mm calibration cube is ideal).
- With `Stephen 0.4 Klipper CHT` + the new ASA filament + the new ASA process selected, click **Slice**.
- Click **Export G-code** and save somewhere temporary, e.g., `/tmp/asa-test.gcode`.

- [ ] **Step 7: Verify the G-code matches the design**

Run these checks against the exported G-code:

```bash
head -50 /tmp/asa-test.gcode | grep -E "PRINT_START|M572|M142|SET_PRESSURE_ADVANCE|M900"
```

Expected:
- A `PRINT_START EXTRUDER_TEMP=255 BED_TEMP=100` line with **no** trailing `PRINTER_MODEL=` argument.
- **No** `M572` lines (linear advance should not appear).
- **No** `M142` lines (heatbreak target, Prusa-firmware-specific).
- **No** `SET_PRESSURE_ADVANCE` or `M900` lines emitted before `PRINT_START` (Klipper manages PA internally).

Then confirm infill pattern:

```bash
grep -c "infill" /tmp/asa-test.gcode
```

Expected: non-zero count (there will be many infill comments). Visually inspect a mid-print infill section by opening the file in a text editor and looking for `;TYPE:Sparse infill`. The infill G-code should alternate short X and Y segments rather than long straight sweeps — indicative of grid.

- [ ] **Step 8: Document the verified outcome**

No commit required (config files live outside the repo). The design doc and this plan are already committed.

---

## Out of Scope (Follow-up work)

These are intentionally deferred per the spec:

- Running an Orca **Max Volumetric Speed** calibration tower and raising `filament_max_volumetric_speed` beyond 15.
- Running an Orca **Pressure Advance** tower and adding `SET_PRESSURE_ADVANCE ADVANCE=x.xx` to `filament_start_gcode`.
- Cloning the 0.20mm process into 0.15mm fine / 0.25mm draft variants.
- Adding a thermometer inside the enclosure for a real ambient measurement; tuning fan speeds based on the reading.
- Applying the `enable_pressure_advance: 0` + empty `filament_start_gcode` pattern to PLA and PETG filaments too.

---

## Self-Review Notes

1. **Spec coverage:** Every section of the spec maps to a task — printer cleanup (Task 2), filament profile (Task 3), process profile (Task 4), plus verification (Task 5) and safety backup (Task 1). No gaps.
2. **Placeholder scan:** `<EPOCH>` placeholders appear in Tasks 3 and 4 — these are intentional and have explicit substitution instructions in the preceding `date +%s` step.
3. **Type / name consistency:** Parent profile names match the exact Orca system profile filenames (`Prusa Generic ASA @MK4S HF0.4`, `0.20mm STRUCTURAL @MK4S 0.4`). New profile names are consistent between the plan and the design doc.
