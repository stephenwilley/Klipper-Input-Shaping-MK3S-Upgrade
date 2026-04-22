# OrcaSlicer ASA profile setup for Mk3 + Revo HF + Klipper

Date: 2026-04-22
Status: Design approved, awaiting user review

## Context

The user runs a Prusa Mk3 upgraded with a Revo 6 hotend and an Obxidian high-flow nozzle, inside a simple foil-lined tent enclosure. The printer is driven by Klipper with input shaping configured, pressure advance set in `printer.cfg` (0.05), and a custom `PRINT_START` macro that handles adaptive bed mesh and dual-sheet Z offsets.

The user is migrating from PrusaSlicer to OrcaSlicer. They have one existing custom machine profile (`Stephen 0.4 Klipper CHT`) that inherits from the `MyKlipper 0.4 nozzle` base and is tagged as `PRINTER_MODEL_MK4IS` + `HF_NOZZLE` so it can pick up MK4S high-flow filament and process profiles. They have no custom filament or process profiles yet.

The user has bought Polymaker Polylite ASA and wants to print Voron parts with it. They know ASA is warp-prone, want to apply best practices (grid infill to avoid long unidirectional lines, brim, conservative first layer, etc.), and expect the enclosure to reach roughly 40°C soaked (measured ~45–50°C near the bed, ~30°C near the front panel).

## Goals

1. Polish the existing printer profile to remove a small amount of cruft without disrupting what works.
2. Create a Polymaker Polylite ASA filament profile tuned for this printer and the moderately-enclosed environment.
3. Create a 0.20mm structural ASA print process profile aimed at Voron-style functional parts.

## Non-goals

- Rebuilding the printer profile from scratch.
- Tuning Orca's `machine_max_*` fields to exactly match `printer.cfg` (they only drive Orca's time estimate; Klipper enforces real limits).
- Creating 0.15mm fine / 0.25mm draft process variants. We create one 0.20mm profile; variants can be cloned later once the base is verified.
- Doing a pressure advance or flow calibration run. The user prefers to do PA towers per filament and will add `SET_PRESSURE_ADVANCE` to `filament_start_gcode` after calibrating.
- Adding per-part settings like fuzzy skin, ironing, or adaptive layer height.

## Design

### 1. Printer profile cleanup — `Stephen 0.4 Klipper CHT`

File: `~/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json`

**Changes:**
- `machine_start_gcode`: drop the unused `PRINTER_MODEL="MK4IS"` parameter. The user's `PRINT_START` macro in `macros.cfg` does not read it. New value:
  ```
  PRINT_START EXTRUDER_TEMP={first_layer_temperature} BED_TEMP={first_layer_bed_temperature}
  ```

**Deliberately unchanged:**
- `retraction_length`: stays at 0.7mm. Stock Mk3 is 0.8mm, MK4S HF is 0.7mm. The Revo 6 HF melt zone is closer to MK4S than stock Mk3 v6 geometry, and the user has not reported stringing. Revisit if Polylite strings.
- `machine_max_*` fields: left alone. They drive Orca's time estimate only; Klipper enforces real limits from `printer.cfg`.
- MK4IS / HF_NOZZLE compatibility tags in `printer_notes`: required for the new filament and process profiles to match.
- All other fields (bed size, extruder clearance, thumbnails, inheritance, Z-hop, wipe, deretraction_speed): unchanged.

### 2. Filament profile — `Polymaker Polylite ASA @Stephen MK4IS HF0.4`

File: `~/Library/Application Support/OrcaSlicer/user/default/filament/Polymaker Polylite ASA @Stephen MK4IS HF0.4.json`

**Inherits:** `Prusa Generic ASA @MK4S HF0.4` → `Prusa Generic ASA @MK4S` → `fdm_filament_asa`.

**Overrides:**

| Field | Value | Rationale |
|---|---|---|
| `filament_vendor` | `"Polymaker"` | Vendor tag |
| `filament_type` | `"ASA"` | Inherited; set explicitly for clarity |
| `filament_density` | `1.07` | Polylite ASA datasheet |
| `filament_cost` | `30` | User paid CAD 29.78 |
| `nozzle_temperature` | `250` | Middle of Polylite's 240–260°C range |
| `nozzle_temperature_initial_layer` | `255` | Slight bump for first layer bond |
| `nozzle_temperature_range_low` | `240` | Polylite min |
| `nozzle_temperature_range_high` | `260` | Polylite max |
| `hot_plate_temp`, `cool_plate_temp`, `eng_plate_temp` | `100` | Textured PEI + ASA; Polymaker recommends 90–100 |
| Same three `_initial_layer` variants | `100` | First layer same as subsequent |
| `filament_max_volumetric_speed` | `15` | Conservative starting point between MK3.5 (11) and MK4S HF (26). User will run Orca's Max Volumetric Speed calibration tower and raise. |
| `fan_min_speed` | `15` | Moderate enclosure, some cooling always on |
| `fan_max_speed` | `30` | Below MK4S HF's 35; enclosure is cooler than a CoreXY MK4 |
| `overhang_fan_threshold` | `25%` | Default |
| `overhang_fan_speed` | `40` | Overhangs benefit from cooling even enclosed |
| `close_fan_the_first_x_layers` | `3` | Zero fan for layers 1–3; locks print to bed |
| `slow_down_layer_time` | `15` | Seconds; prevents small-feature overheating |
| `slow_down_min_speed` | `10` | mm/s floor when slowed |
| `enable_pressure_advance` | `0` | **Critical.** Klipper owns PA via `printer.cfg`. If Orca also sets it they fight. |
| `filament_start_gcode` | `[""]` (single-element array with empty string) | Override parent's `M572` (linear advance) and `M142` (heatbreak temp). Neither applies to this Klipper setup. User will add `SET_PRESSURE_ADVANCE ADVANCE=x.xx` here after PA tower. Stored as an array to match Orca's schema for this field. |

**Compatibility:** The parent profile uses `compatible_printers: ["Prusa MK4S HF0.4 nozzle"]` — an exact-name match that does not cover the user's custom printer. Override with `compatible_printers: ["Stephen 0.4 Klipper CHT"]` so this filament shows up specifically for that printer profile.

### 3. Process profile — `0.20mm Structural ASA @Stephen 0.4 Klipper CHT`

File: `~/Library/Application Support/OrcaSlicer/user/default/process/0.20mm Structural ASA @Stephen 0.4 Klipper CHT.json`

**Inherits:** `0.20mm STRUCTURAL @MK4S 0.4` (Prusa's MK4S structural profile — tuned for functional parts; we add ASA-specific and Voron-specific overrides). Note the uppercase `STRUCTURAL` in the parent profile name — this is the exact name in Orca's profiles.

**Layer / perimeter geometry:**

| Field | Value |
|---|---|
| `layer_height` | `0.20` |
| `initial_layer_print_height` | `0.20` |
| `line_width` | `0.45` |
| `wall_loops` | `4` |
| `top_shell_layers` | `5` |
| `bottom_shell_layers` | `5` |

**Infill (addresses user's "no long straight lines" requirement):**

| Field | Value |
|---|---|
| `sparse_infill_pattern` | `grid` |
| `sparse_infill_density` | `40%` |
| `top_surface_pattern` | `monotonic` |
| `bottom_surface_pattern` | `monotonic` |

**Adhesion / warping mitigation:**

| Field | Value |
|---|---|
| `brim_type` | `auto_brim` |
| `brim_width` | `5` |
| `brim_object_gap` | `0.1` |
| `raft_layers` | `0` |

**Speeds (mm/s):**

| Field | Value |
|---|---|
| `initial_layer_speed` | `25` |
| `initial_layer_infill_speed` | `40` |
| `outer_wall_speed` | `40` |
| `inner_wall_speed` | `60` |
| `sparse_infill_speed` | `80` |
| `internal_solid_infill_speed` | `80` |
| `top_surface_speed` | `40` |
| `gap_infill_speed` | `40` |
| `travel_speed` | `250` |

**Accelerations (mm/s²):**

| Field | Value |
|---|---|
| `outer_wall_acceleration` | `2000` |
| `inner_wall_acceleration` | `3000` |
| `top_surface_acceleration` | `2000` |
| `initial_layer_acceleration` | `1000` |
| `default_acceleration` | `3000` |
| `travel_acceleration` | `4000` |

**Seam and quality:**

| Field | Value |
|---|---|
| `seam_position` | `nearest` |
| `wall_infill_order` | `inner wall/outer wall/infill` |
| `infill_wall_overlap` | `15%` |

**Travel and retraction:**
- `retract_when_changing_layer`: `true`
- `wipe`: inherited `false` from printer profile
- `z_hop`: inherited `0.2` from printer profile

**Explicitly off:**
- Ironing — can drag/scorch at 250°C.
- Fuzzy skin — not wanted for Voron parts.
- Adaptive layer height — makes early tuning harder.

**Compatibility:** The parent uses `compatible_printers_condition: "printer_notes=~/.*MK4S.*/ and nozzle_diameter[0]==0.4"`, which does not match the user's `PRINTER_MODEL_MK4IS` tag (MK4IS and MK4S are distinct strings). Override with explicit `compatible_printers: ["Stephen 0.4 Klipper CHT"]` so this process appears only for that printer.

## Files affected

| File | Action |
|---|---|
| `~/Library/Application Support/OrcaSlicer/user/default/machine/Stephen 0.4 Klipper CHT.json` | Edit (1 field) |
| `~/Library/Application Support/OrcaSlicer/user/default/filament/Polymaker Polylite ASA @Stephen MK4IS HF0.4.json` | Create |
| `~/Library/Application Support/OrcaSlicer/user/default/filament/Polymaker Polylite ASA @Stephen MK4IS HF0.4.info` | Create (Orca profile metadata sidecar) |
| `~/Library/Application Support/OrcaSlicer/user/default/process/0.20mm Structural ASA @Stephen 0.4 Klipper CHT.json` | Create |
| `~/Library/Application Support/OrcaSlicer/user/default/process/0.20mm Structural ASA @Stephen 0.4 Klipper CHT.info` | Create |

## Validation plan

1. Restart OrcaSlicer and confirm the printer profile still loads without errors.
2. With the printer profile selected, confirm the new filament profile appears in the filament dropdown.
3. With the printer + filament selected, confirm the new process profile appears in the process dropdown.
4. Slice a simple calibration cube and confirm:
   - Start G-code shows `PRINT_START EXTRUDER_TEMP=255 BED_TEMP=100` with no `PRINTER_MODEL=` argument.
   - No `M572` in emitted G-code.
   - No `SET_PRESSURE_ADVANCE` or `M900` emitted by Orca.
   - Brim appears per `auto_brim` logic.
   - Infill is grid pattern.
5. Print a small test part. Confirm first layer sticks, no warping on edges, no underextrusion at 15 mm³/s.
6. Run Orca's Max Volumetric Speed calibration tower with this filament profile. Update `filament_max_volumetric_speed`.

## Follow-up / later work (out of scope)

- Run a PA tower in Orca, add `SET_PRESSURE_ADVANCE ADVANCE=x.xx` to `filament_start_gcode`.
- Clone the 0.20mm process to 0.15mm fine and 0.25mm draft variants once the base is proven.
- Add a thermometer inside the enclosure for a real ambient reading; if consistently > 40°C, raise `fan_max_speed` toward 40 and drop `slow_down_layer_time` toward 10s.
- Consider a PETG/PLA-Klipper-specific filament profile with `enable_pressure_advance: 0` and the same empty `filament_start_gcode` trick, to avoid Orca fighting Klipper on existing materials too.
