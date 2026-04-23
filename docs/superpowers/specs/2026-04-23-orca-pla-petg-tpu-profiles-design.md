# OrcaSlicer PLA/PETG/TPU profile setup for Mk3 + Revo HF + Klipper

Date: 2026-04-23
Status: Implemented 2026-04-23, with three follow-up additions made during verification — see "Post-implementation additions" below.

## Context

Following the successful ASA migration (see `2026-04-22-orca-asa-profiles-design.md`), the user wants to migrate their PLA, PETG, and TPU setups from PrusaSlicer to OrcaSlicer. The same machine constraints apply: Mk3 frame (Cartesian), Klipper with input shaping, pressure advance owned by `printer.cfg`, Revo 6 + Obxidian high-flow nozzle, concrete paver + closed cell foam vibration mount.

The user's PrusaSlicer custom filament profiles are functionally thin: each is a Prusa generic parent plus a single `SET_PRESSURE_ADVANCE` line in `start_filament_gcode`. PA values were determined per-filament using PA towers. The PrusaSlicer custom process profiles are similarly thin: Prusa MK4IS defaults with the user's accelerations.

In their last Orca trial the user observed faster prints but lower visual quality than PrusaSlicer. Inspection of the `MyKlipper` system process profiles confirms why: defaults are tuned for CoreXY-class machines (5000 default accel, 7000 travel accel, 120 mm/s outer walls, 350 mm/s travel) and exceed both the printer's hardware ceiling (4500 accel, 300 mm/s in `printer.cfg`) and what a Cartesian Mk3 frame can deliver cleanly.

## Goals

1. Migrate the user's calibrated PLA and PETG filament profiles to OrcaSlicer, preserving each filament's tower-derived pressure advance value.
2. Provide one generic TPU profile as a starting point (PA to be calibrated later).
3. Create a single custom process profile tuned for the Mk3 frame at 0.20mm to recover the print quality lost when using stock MyKlipper defaults.
4. Use the `@Stephen` suffix on every custom profile, matching the existing convention.

## Non-goals

- Migrating uncalibrated branded TPU/FLEX profiles. The 4 PrusaSlicer entries (Generic FLEX, NinjaTek, Overture HS TPU, SainSmart) had no PA set and no other distinguishing values. One Generic TPU profile covers all of them until the user actually loads one and runs a tower.
- Migrating branded PLA/PETG profiles whose PA matches the generic value within rounding. The point is to preserve genuine per-filament calibration, not to mirror inventory.
- Creating per-quality process variants (0.15mm, 0.25mm, etc.) up front. One 0.20mm process now; clone later if needed.
- Per-filament process profiles. The process is material-agnostic; the filament profile carries material-specific behavior.
- Changing `printer.cfg`. The hardware ceiling stays where it is; the slicer operates conservatively below it for quality.

## Compatibility approach (lesson from the ASA setup)

Every new profile in this spec sets `compatible_printers: ["Stephen 0.4 Klipper CHT"]` explicitly as an array, never relying on `compatible_printers_condition` regexes.

The ASA migration repeatedly produced "profile doesn't appear in dropdown" symptoms because:

1. Some Prusa parents use a `compatible_printers_condition` regex (e.g. `printer_notes=~/.*MK4S.*/`) that the user's `MK4IS`-tagged printer does not match.
2. An empty `compatible_printers_condition` field is treated by Orca as "always fails" rather than "no restriction".
3. When the inheritance chain crosses vendors that aren't enabled in the user's Orca install (Custom + OrcaFilamentLibrary only), parent profiles can fail to load silently.

The fix in every case is the same: explicit `compatible_printers` list naming `Stephen 0.4 Klipper CHT`, and inherit only from parents that live in vendors actually enabled in the user's Orca install (Custom or OrcaFilamentLibrary). This spec respects both rules. The branded filaments inherit from `Generic <material> @Stephen` (a User profile, always available), and re-state `compatible_printers` explicitly rather than relying on inheritance, to avoid surprises.

## Design

### 1. Custom process profile — `0.20mm Standard @Stephen`

File: `~/Library/Application Support/OrcaSlicer/user/default/process/0.20mm Standard @Stephen.json`

**Inherits:** `0.20mm Standard @MyKlipper` → `fdm_process_klipper_common` → `fdm_process_common`. The MyKlipper parent already sets up Klipper-native gcode and `exclude_object: 1`. We override only the speed and acceleration block.

**Why the override:** The Mk3 + Revo HF + concrete-foam-mount is well-tuned but is still a Cartesian belt-and-frame system. The MyKlipper outer-wall defaults (120 mm/s @ 3000 mm/s²) cause visible ringing on this class of machine even with good input shaping. Dropping outer wall speed and acceleration is the single biggest quality lever; everything else follows from keeping ratios sensible. Travel and inner walls stay closer to the hardware ceiling because they are hidden or insensitive to ringing.

**Speeds (mm/s):**

| Field | Value | Rationale |
|---|---|---|
| `outer_wall_speed` | 100 | Visible walls; conservative within Mk3 frame envelope. |
| `inner_wall_speed` | 180 | Hidden, ringing irrelevant. |
| `internal_solid_infill_speed` | 180 | Same. |
| `top_surface_speed` | 60 | Visible flat surfaces — slowest critical zone. |
| `sparse_infill_speed` | 180 | At 0.45 × 0.20 line, this is ~16 mm³/s — under typical HF max flow (~21 mm³/s for PETG HF, ~25 for PLA HF). |
| `gap_infill_speed` | 80 | Small precise moves. |
| `initial_layer_speed` | 30 | First-layer adhesion. |
| `initial_layer_infill_speed` | 60 | Slightly faster on first-layer infill, still safe. |
| `travel_speed` | 250 | Below 300 mm/s `printer.cfg` ceiling so Klipper isn't clamping. |

Other speeds (`bridge_speed`, `support_speed`, overhang speeds) inherit unchanged.

**Accelerations (mm/s²):**

| Field | Value | Rationale |
|---|---|---|
| `default_acceleration` | 3000 | Conservative under 4500 hardware ceiling. |
| `outer_wall_acceleration` | 2000 | Single biggest quality lever. |
| `inner_wall_acceleration` | 3000 | Hidden, can run higher. |
| `internal_solid_infill_acceleration` | 3000 | Same. |
| `top_surface_acceleration` | 2000 | Visible. |
| `initial_layer_acceleration` | 500 | Adhesion-critical. |
| `travel_acceleration` | 4000 | Just under hardware ceiling. |

**Other settings:** All other process settings (wall_loops, infill density, top/bottom shells, brim, support, etc.) inherit unchanged from MyKlipper Standard. ASA-specific overrides like grid infill and brim live on the existing ASA process profile, not here.

**Compatibility:** `compatible_printers: ["Stephen 0.4 Klipper CHT"]` (explicit list, same pattern as the ASA process — sidesteps the MyKlipper parent's compatibility regex which targets MyKlipper machines).

### 2. Filament profiles — generics

Three generic profiles, one per material. Each is the canonical "use this filament unless I have a calibrated branded variant" entry. Each carries the Klipper compatibility fixes once so branded variants can inherit them.

#### `Generic PLA @Stephen`

File: `~/Library/Application Support/OrcaSlicer/user/default/filament/Generic PLA @Stephen.json`

**Inherits:** `Generic PLA @System` (OrcaFilamentLibrary).

**Overrides:**

| Field | Value | Rationale |
|---|---|---|
| `enable_pressure_advance` | `0` | Klipper owns PA. |
| `filament_start_gcode` | `["SET_PRESSURE_ADVANCE ADVANCE=0.0475"]` | Default PA for generic PLA from user's tower history. |
| `filament_max_volumetric_speed` | `21` | Matches PrusaSlicer setting; Revo HF can push this. |
| `compatible_printers` | `["Stephen 0.4 Klipper CHT"]` | Restrict to user's printer. |
| `filament_notes` | (see below) | Tracks which branded filaments have been towered against this profile. |

**`filament_notes` content:**

```
Default for any PLA not in the list below.

Towered, matches this generic (use this profile):
- Overture Matte Black PLA — PA 0.0475
- Elegoo PLA Pro Purple — PA 0.0455 (within rounding)

Towered, has its own profile (use that instead):
- 3D-HONGDAK Pink PLA — PA 0.03
- Eryone Wood PLA — PA 0.035
- Overture Matte Black ECO-PLA Older — PA 0.0375
- Pro White PLA — PA 0.055
```

**Deliberately not set:** filament cost, filament colour. User does not want to maintain those.

#### `Generic PETG @Stephen`

File: `~/Library/Application Support/OrcaSlicer/user/default/filament/Generic PETG @Stephen.json`

**Inherits:** `Generic PETG HF @System` (OrcaFilamentLibrary HF variant for the Revo HF).

**Overrides:**

| Field | Value | Rationale |
|---|---|---|
| `enable_pressure_advance` | `0` | Klipper owns PA. |
| `filament_start_gcode` | `["SET_PRESSURE_ADVANCE ADVANCE=0.11"]` | Default PETG PA from user's tower history. |
| `filament_max_volumetric_speed` | `21` | Matches PrusaSlicer setting. |
| `compatible_printers` | `["Stephen 0.4 Klipper CHT"]` | Restrict to user's printer. |
| `filament_notes` | (see below) | Tracks which branded filaments have been towered against this profile. |

**`filament_notes` content:**

```
Default for any PETG not in the list below.

Towered, matches this generic (use this profile):
- AmazonBasics Orange PETG — PA 0.115 (within rounding)
- Elegoo Rapid PETG @HF0.4 — PA 0.11
- Polylite Blue PETG — PA 0.11

Towered, has its own profile (use that instead):
- Elegoo Black PETG — PA 0.0625 (outlier — re-tower to verify)
```

#### `Generic TPU @Stephen`

File: `~/Library/Application Support/OrcaSlicer/user/default/filament/Generic TPU @Stephen.json`

**Inherits:** `Generic TPU @System` (OrcaFilamentLibrary).

**Overrides:**

| Field | Value | Rationale |
|---|---|---|
| `enable_pressure_advance` | `0` | Klipper owns PA. |
| `filament_start_gcode` | `["; SET_PRESSURE_ADVANCE ADVANCE=0.05  ; uncomment after PA tower"]` | TPU PA not yet calibrated; placeholder line. |
| `compatible_printers` | `["Stephen 0.4 Klipper CHT"]` | Restrict to user's printer. |
| `filament_notes` | (see below) | Tracks which branded filaments have been towered against this profile. |

**`filament_notes` content:**

```
Default for any TPU/FLEX not in the list below.

Not yet towered (run a PA tower before trusting):
- Generic FLEX
- NinjaTek NinjaFlex TPU
- Overture High Speed TPU
- SainSmart TPU
```

Other TPU settings (slow speeds, conservative retraction, fan profile) inherit from the Orca system parent without changes.

### 3. Filament profiles — branded variants

Each branded profile is a thin override of the corresponding `@Stephen` generic, changing only the PA value. Branded profiles only exist where the calibrated PA differs meaningfully (>0.005) from the generic — see "Why these and not others" below.

All branded profiles share this shape:

```json
{
  "type": "filament",
  "name": "<brand> <colour> <material> @Stephen",
  "inherits": "Generic <material> @Stephen",
  "from": "User",
  "instantiation": "true",
  "filament_vendor": "<brand>",
  "filament_start_gcode": ["SET_PRESSURE_ADVANCE ADVANCE=<value>"],
  "compatible_printers": ["Stephen 0.4 Klipper CHT"]
}
```

Profiles to create:

| Profile name | Inherits | PA value |
|---|---|---|
| `3D-HONGDAK Pink PLA @Stephen` | `Generic PLA @Stephen` | `0.03` |
| `Eryone Wood PLA @Stephen` | `Generic PLA @Stephen` | `0.035` |
| `Overture Matte Black ECO-PLA Older @Stephen` | `Generic PLA @Stephen` | `0.0375` |
| `Pro White PLA @Stephen` | `Generic PLA @Stephen` | `0.055` |
| `Elegoo Black PETG @Stephen` | `Generic PETG @Stephen` | `0.0625` |

**Why these and not others:** From the user's PrusaSlicer catalog of 16 custom profiles, these 5 have a calibrated PA value that differs meaningfully from the generic. The remaining 11 either match the generic PA within rounding (Overture Matte Black PLA at 0.0475, Elegoo PLA Pro Purple at 0.0455, AmazonBasics PETG at 0.115, Elegoo Rapid PETG at 0.11, Polylite Blue PETG at 0.11) or have no calibrated PA at all (Generic FLEX, NinjaTek, Overture HS TPU, SainSmart TPU). The user can clone-and-tweak as new spools are loaded.

**Note on Elegoo Black PETG (0.0625):** This value is far below the cluster of other PETGs (~0.11) and may be a bad tower run rather than a real material difference. Migrate as-is, but worth re-towering before trusting.

### 4. `.info` sidecar files

Every `.json` profile in `~/Library/Application Support/OrcaSlicer/user/default/{filament,process}/` requires a matching `.info` file with the same basename and `.info` extension, or Orca will not load the profile. Use this template for all new profiles:

```
sync_info = create
user_id =
setting_id =
base_id =
updated_time = <unix timestamp>
```

Leave `base_id` empty for all new files. The `inherits` field in the `.json` profile does the actual parent linking; `base_id` in `.info` only drives Orca's update-sync tracking and is safe to leave blank for user-authored profiles. This matches the pattern of the existing ASA process `.info`.

### 5. Klipper `printer.cfg`

**No changes.** The hardware ceiling (4500 accel, 300 mm/s) remains. The slicer's per-feature operating points all fall under that ceiling. Lowering the ceiling would lose headroom for travel moves and partially waste the input-shaping calibration done at 4500.

## Files affected

| File | Action |
|---|---|
| `~/Library/Application Support/OrcaSlicer/user/default/process/0.20mm Standard @Stephen.json` + `.info` | Create |
| `~/Library/Application Support/OrcaSlicer/user/default/filament/Generic PLA @Stephen.json` + `.info` | Create |
| `~/Library/Application Support/OrcaSlicer/user/default/filament/Generic PETG @Stephen.json` + `.info` | Create |
| `~/Library/Application Support/OrcaSlicer/user/default/filament/Generic TPU @Stephen.json` + `.info` | Create |
| `~/Library/Application Support/OrcaSlicer/user/default/filament/3D-HONGDAK Pink PLA @Stephen.json` + `.info` | Create |
| `~/Library/Application Support/OrcaSlicer/user/default/filament/Eryone Wood PLA @Stephen.json` + `.info` | Create |
| `~/Library/Application Support/OrcaSlicer/user/default/filament/Overture Matte Black ECO-PLA Older @Stephen.json` + `.info` | Create |
| `~/Library/Application Support/OrcaSlicer/user/default/filament/Pro White PLA @Stephen.json` + `.info` | Create |
| `~/Library/Application Support/OrcaSlicer/user/default/filament/Elegoo Black PETG @Stephen.json` + `.info` | Create |

Total: 9 new profiles (3 generic filaments + 5 branded filaments + 1 process), 18 files including `.info` sidecars.

## Validation plan

1. Restart Orca; confirm **all 9 new profiles** appear in their respective dropdowns when `Stephen 0.4 Klipper CHT` is the selected printer. If any profile is missing, the most likely cause is a parent that doesn't load — check that every `inherits` value resolves to either an OrcaFilamentLibrary system profile or a User profile we created in this spec, and that `compatible_printers` is set as an explicit list.
2. With `Generic PLA @Stephen` selected, slice a small calibration cube. Confirm:
   - Emitted G-code contains `SET_PRESSURE_ADVANCE ADVANCE=0.0475` from `filament_start_gcode`.
   - No `M572` (PrusaSlicer linear advance) anywhere in the file.
   - No `M900` (Marlin linear advance) anywhere in the file.
   - No second `SET_PRESSURE_ADVANCE` from Orca's own PA system (because `enable_pressure_advance: 0`).
3. With `0.20mm Standard @Stephen` selected, confirm `M204` (or Orca's `SET_VELOCITY_LIMIT`) values match the design accelerations.
4. Print a small test part in PLA at the new speeds. Compare visual quality against the same part sliced in PrusaSlicer. Outer-wall ringing should be absent or significantly reduced versus stock MyKlipper.
5. Print a small test part in PETG, confirm bed adhesion at the inherited HF temperatures and that PA value tracks.
6. Repeat with one branded variant (e.g. `Pro White PLA @Stephen`) to confirm inheritance chain delivers the brand-specific PA value.

## Post-implementation additions

After verification of the PLA slice output but before the physical print test, three small changes were made to clean up naming inconsistencies and add TPU process support that was missing from the original spec:

**1. ASA profile renames.** The two ASA profiles created in the prior spec (`2026-04-22-orca-asa-profiles-design.md`) used noisier suffixes than the new convention:

- `Polymaker Polylite ASA @Stephen MK4IS HF0.4` → `Polymaker Polylite ASA @Stephen`
- `0.20mm Structural ASA @Stephen 0.4 Klipper CHT` → `0.20mm Structural ASA @Stephen`

Each rename was: `mv` for the `.json` and `.info` sidecar, then update the `name` field inside the JSON. No inheritance chain referenced the old names, so nothing else needed updating.

**2. `filename_format` override on every custom process.** The MyKlipper parent inherits `fdm_process_common`'s default of `{input_filename_base}_{layer_height}mm_{filament_type[initial_tool]}_{printer_model}_{print_time}.gcode`, which puts `MK4IS` in every output filename. Since the user's printer is a Mk3 chassis pretending to be an MK4IS only for parent-profile compatibility, the embedded `MK4IS` is misleading. Applied to all three custom processes:

```
"filename_format": "{input_filename_base}_{layer_height}mm_{filament_type[initial_tool]}_{print_time}.gcode"
```

**3. `0.20mm Flex @Stephen` process for TPU.** TPU on a direct-drive setup needs much lower acceleration than PLA/PETG — the soft filament physically can't follow quick direction changes and produces blobs/pulling at every corner. The filament profile alone does not address this because acceleration lives in the process profile. Created a TPU-specific process inheriting from `0.20mm Standard @MyKlipper`:

| Field | Value |
|---|---|
| `outer_wall_speed` | 30 |
| `inner_wall_speed` | 50 |
| `internal_solid_infill_speed` | 50 |
| `top_surface_speed` | 25 |
| `sparse_infill_speed` | 50 |
| `gap_infill_speed` | 25 |
| `initial_layer_speed` | 20 |
| `initial_layer_infill_speed` | 30 |
| `travel_speed` | 150 |
| `default_acceleration` | 1000 |
| `outer_wall_acceleration` | 800 |
| `inner_wall_acceleration` | 1000 |
| `internal_solid_infill_acceleration` | 1000 |
| `top_surface_acceleration` | 800 |
| `initial_layer_acceleration` | 500 |
| `travel_acceleration` | 1500 |
| `z_hop` | `["0"]` (overrides printer profile's 0.2 — TPU strings on hops) |

Lower travel speed (150 vs 250) reduces stringing on flexible materials. `compatible_printers` and `filename_format` follow the same convention as the other custom processes.

## Follow-up / later work (out of scope)

- Run a TPU PA tower and update `Generic TPU @Stephen` start gcode.
- Re-tower Elegoo Black PETG to validate the 0.0625 outlier.
- Clone `0.20mm Standard @Stephen` to 0.15mm and 0.25mm variants if those qualities become routinely needed.
- If the user wants per-filament colour swatches for visual inventory, add `filament_colour` to each branded profile (skipped here per user preference).
- If the user adopts new branded filaments and runs a PA tower, decide whether the calibrated value differs meaningfully from the generic before creating a branded profile.
