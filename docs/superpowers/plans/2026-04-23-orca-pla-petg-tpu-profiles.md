# OrcaSlicer PLA/PETG/TPU Profiles Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create one Mk3-tuned process profile and 8 filament profiles (3 generic + 5 branded) in OrcaSlicer, mirroring the user's calibrated PrusaSlicer setup with Klipper-compatibility fixes baked in.

**Architecture:** All profiles are JSON files in OrcaSlicer's user-profile directory with `.info` sidecars. Process profile inherits from `0.20mm Standard @MyKlipper` (Custom vendor). Generic filament profiles inherit from OrcaFilamentLibrary system parents. Branded filament profiles inherit from the corresponding generic `@Stephen` filament so Klipper fixes are inherited rather than duplicated. Every profile sets `compatible_printers: ["Stephen 0.4 Klipper CHT"]` explicitly to avoid the silent inheritance/regex issues encountered during the ASA migration.

**Tech Stack:** OrcaSlicer 2.3.x JSON profile schema; Klipper 0.13+ on the printer side.

**Reference:** Design spec at `docs/superpowers/specs/2026-04-23-orca-pla-petg-tpu-profiles-design.md`. Existing ASA profiles at `~/Library/Application Support/OrcaSlicer/user/default/{process,filament}/*ASA*` are concrete templates for file shape.

---

## Conventions used throughout this plan

- All file paths to `~/Library/Application Support/OrcaSlicer/user/default/...` use `$ORCA` as a shorthand. Set it once in your shell:
  ```bash
  export ORCA="$HOME/Library/Application Support/OrcaSlicer/user/default"
  ```
- Quit OrcaSlicer before creating any profile files. Orca caches profile state on shutdown and will overwrite changes if it is running. Re-launch only after a task says to.
- After every JSON file write, validate parse with:
  ```bash
  python3 -m json.tool "<path>" > /dev/null && echo OK
  ```
- `.info` sidecar template (used in every filament/process task — `updated_time` uses the current Unix timestamp via `$(date +%s)`):
  ```
  sync_info = create
  user_id =
  setting_id =
  base_id =
  updated_time = <timestamp>
  ```
- Documentation commits go in the repo at `/Users/stephen/git/Klipper-Input-Shaping-MK3S-Upgrade`; profile files themselves are not tracked in git.

---

### Task 1: Quit OrcaSlicer

**Files:** none

- [ ] **Step 1: Confirm OrcaSlicer is not running**

```bash
pgrep -x OrcaSlicer && echo "STILL RUNNING — quit it first" || echo "not running, OK to proceed"
```

Expected: `not running, OK to proceed`

If it prints "STILL RUNNING", quit OrcaSlicer via the application menu (⌘Q) and rerun the check.

- [ ] **Step 2: Set the ORCA shorthand**

```bash
export ORCA="$HOME/Library/Application Support/OrcaSlicer/user/default"
ls "$ORCA/process" "$ORCA/filament"
```

Expected: existing ASA files listed (`0.20mm Structural ASA @Stephen 0.4 Klipper CHT.json` etc.).

---

### Task 2: Create custom process profile `0.20mm Standard @Stephen`

**Files:**
- Create: `$ORCA/process/0.20mm Standard @Stephen.json`
- Create: `$ORCA/process/0.20mm Standard @Stephen.info`

- [ ] **Step 1: Write the process JSON**

```bash
cat > "$ORCA/process/0.20mm Standard @Stephen.json" <<'EOF'
{
    "type": "process",
    "name": "0.20mm Standard @Stephen",
    "inherits": "0.20mm Standard @MyKlipper",
    "from": "User",
    "instantiation": "true",
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "outer_wall_speed": "100",
    "inner_wall_speed": "180",
    "internal_solid_infill_speed": "180",
    "top_surface_speed": "60",
    "sparse_infill_speed": "180",
    "gap_infill_speed": "80",
    "initial_layer_speed": "30",
    "initial_layer_infill_speed": "60",
    "travel_speed": "250",
    "default_acceleration": "3000",
    "outer_wall_acceleration": "2000",
    "inner_wall_acceleration": "3000",
    "internal_solid_infill_acceleration": "3000",
    "top_surface_acceleration": "2000",
    "initial_layer_acceleration": "500",
    "travel_acceleration": "4000",
    "version": "2.3.2.60"
}
EOF
```

- [ ] **Step 2: Validate JSON parse**

```bash
python3 -m json.tool "$ORCA/process/0.20mm Standard @Stephen.json" > /dev/null && echo OK
```

Expected: `OK`

- [ ] **Step 3: Write the .info sidecar**

```bash
cat > "$ORCA/process/0.20mm Standard @Stephen.info" <<EOF
sync_info = create
user_id =
setting_id =
base_id =
updated_time = $(date +%s)
EOF
```

- [ ] **Step 4: Confirm both files exist**

```bash
ls -la "$ORCA/process/0.20mm Standard @Stephen."{json,info}
```

Expected: both files listed with non-zero size.

---

### Task 3: Create `Generic PLA @Stephen` filament

**Files:**
- Create: `$ORCA/filament/Generic PLA @Stephen.json`
- Create: `$ORCA/filament/Generic PLA @Stephen.info`

- [ ] **Step 1: Write the filament JSON**

```bash
cat > "$ORCA/filament/Generic PLA @Stephen.json" <<'EOF'
{
    "type": "filament",
    "name": "Generic PLA @Stephen",
    "inherits": "Generic PLA @System",
    "from": "User",
    "instantiation": "true",
    "filament_vendor": [
        "Generic"
    ],
    "filament_max_volumetric_speed": [
        "21"
    ],
    "enable_pressure_advance": [
        "0"
    ],
    "filament_start_gcode": [
        "SET_PRESSURE_ADVANCE ADVANCE=0.0475"
    ],
    "filament_notes": "Default for any PLA not in the list below.\n\nTowered, matches this generic (use this profile):\n- Overture Matte Black PLA — PA 0.0475\n- Elegoo PLA Pro Purple — PA 0.0455 (within rounding)\n\nTowered, has its own profile (use that instead):\n- 3D-HONGDAK Pink PLA — PA 0.03\n- Eryone Wood PLA — PA 0.035\n- Overture Matte Black ECO-PLA Older — PA 0.0375\n- Pro White PLA — PA 0.055",
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "version": "2.3.2.60"
}
EOF
```

- [ ] **Step 2: Validate JSON parse**

```bash
python3 -m json.tool "$ORCA/filament/Generic PLA @Stephen.json" > /dev/null && echo OK
```

Expected: `OK`

- [ ] **Step 3: Write the .info sidecar**

```bash
cat > "$ORCA/filament/Generic PLA @Stephen.info" <<EOF
sync_info = create
user_id =
setting_id =
base_id =
updated_time = $(date +%s)
EOF
```

- [ ] **Step 4: Confirm both files exist**

```bash
ls -la "$ORCA/filament/Generic PLA @Stephen."{json,info}
```

Expected: both files listed with non-zero size.

---

### Task 4: Create `Generic PETG @Stephen` filament

**Files:**
- Create: `$ORCA/filament/Generic PETG @Stephen.json`
- Create: `$ORCA/filament/Generic PETG @Stephen.info`

- [ ] **Step 1: Write the filament JSON**

```bash
cat > "$ORCA/filament/Generic PETG @Stephen.json" <<'EOF'
{
    "type": "filament",
    "name": "Generic PETG @Stephen",
    "inherits": "Generic PETG HF @System",
    "from": "User",
    "instantiation": "true",
    "filament_vendor": [
        "Generic"
    ],
    "filament_max_volumetric_speed": [
        "21"
    ],
    "enable_pressure_advance": [
        "0"
    ],
    "filament_start_gcode": [
        "SET_PRESSURE_ADVANCE ADVANCE=0.11"
    ],
    "filament_notes": "Default for any PETG not in the list below.\n\nTowered, matches this generic (use this profile):\n- AmazonBasics Orange PETG — PA 0.115 (within rounding)\n- Elegoo Rapid PETG @HF0.4 — PA 0.11\n- Polylite Blue PETG — PA 0.11\n\nTowered, has its own profile (use that instead):\n- Elegoo Black PETG — PA 0.0625 (outlier — re-tower to verify)",
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "version": "2.3.2.60"
}
EOF
```

- [ ] **Step 2: Validate JSON parse**

```bash
python3 -m json.tool "$ORCA/filament/Generic PETG @Stephen.json" > /dev/null && echo OK
```

Expected: `OK`

- [ ] **Step 3: Write the .info sidecar**

```bash
cat > "$ORCA/filament/Generic PETG @Stephen.info" <<EOF
sync_info = create
user_id =
setting_id =
base_id =
updated_time = $(date +%s)
EOF
```

- [ ] **Step 4: Confirm both files exist**

```bash
ls -la "$ORCA/filament/Generic PETG @Stephen."{json,info}
```

Expected: both files listed with non-zero size.

---

### Task 5: Create `Generic TPU @Stephen` filament

**Files:**
- Create: `$ORCA/filament/Generic TPU @Stephen.json`
- Create: `$ORCA/filament/Generic TPU @Stephen.info`

- [ ] **Step 1: Write the filament JSON**

```bash
cat > "$ORCA/filament/Generic TPU @Stephen.json" <<'EOF'
{
    "type": "filament",
    "name": "Generic TPU @Stephen",
    "inherits": "Generic TPU @System",
    "from": "User",
    "instantiation": "true",
    "filament_vendor": [
        "Generic"
    ],
    "enable_pressure_advance": [
        "0"
    ],
    "filament_start_gcode": [
        "; SET_PRESSURE_ADVANCE ADVANCE=0.05  ; uncomment after PA tower"
    ],
    "filament_notes": "Default for any TPU/FLEX not in the list below.\n\nNot yet towered (run a PA tower before trusting):\n- Generic FLEX\n- NinjaTek NinjaFlex TPU\n- Overture High Speed TPU\n- SainSmart TPU",
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "version": "2.3.2.60"
}
EOF
```

- [ ] **Step 2: Validate JSON parse**

```bash
python3 -m json.tool "$ORCA/filament/Generic TPU @Stephen.json" > /dev/null && echo OK
```

Expected: `OK`

- [ ] **Step 3: Write the .info sidecar**

```bash
cat > "$ORCA/filament/Generic TPU @Stephen.info" <<EOF
sync_info = create
user_id =
setting_id =
base_id =
updated_time = $(date +%s)
EOF
```

- [ ] **Step 4: Confirm both files exist**

```bash
ls -la "$ORCA/filament/Generic TPU @Stephen."{json,info}
```

Expected: both files listed with non-zero size.

---

### Task 6: Create `3D-HONGDAK Pink PLA @Stephen` filament

**Files:**
- Create: `$ORCA/filament/3D-HONGDAK Pink PLA @Stephen.json`
- Create: `$ORCA/filament/3D-HONGDAK Pink PLA @Stephen.info`

- [ ] **Step 1: Write the filament JSON**

```bash
cat > "$ORCA/filament/3D-HONGDAK Pink PLA @Stephen.json" <<'EOF'
{
    "type": "filament",
    "name": "3D-HONGDAK Pink PLA @Stephen",
    "inherits": "Generic PLA @Stephen",
    "from": "User",
    "instantiation": "true",
    "filament_vendor": [
        "3D-HONGDAK"
    ],
    "filament_start_gcode": [
        "SET_PRESSURE_ADVANCE ADVANCE=0.03"
    ],
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "version": "2.3.2.60"
}
EOF
```

- [ ] **Step 2: Validate JSON parse**

```bash
python3 -m json.tool "$ORCA/filament/3D-HONGDAK Pink PLA @Stephen.json" > /dev/null && echo OK
```

Expected: `OK`

- [ ] **Step 3: Write the .info sidecar**

```bash
cat > "$ORCA/filament/3D-HONGDAK Pink PLA @Stephen.info" <<EOF
sync_info = create
user_id =
setting_id =
base_id =
updated_time = $(date +%s)
EOF
```

---

### Task 7: Create `Eryone Wood PLA @Stephen` filament

**Files:**
- Create: `$ORCA/filament/Eryone Wood PLA @Stephen.json`
- Create: `$ORCA/filament/Eryone Wood PLA @Stephen.info`

- [ ] **Step 1: Write the filament JSON**

```bash
cat > "$ORCA/filament/Eryone Wood PLA @Stephen.json" <<'EOF'
{
    "type": "filament",
    "name": "Eryone Wood PLA @Stephen",
    "inherits": "Generic PLA @Stephen",
    "from": "User",
    "instantiation": "true",
    "filament_vendor": [
        "Eryone"
    ],
    "filament_start_gcode": [
        "SET_PRESSURE_ADVANCE ADVANCE=0.035"
    ],
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "version": "2.3.2.60"
}
EOF
```

- [ ] **Step 2: Validate JSON parse**

```bash
python3 -m json.tool "$ORCA/filament/Eryone Wood PLA @Stephen.json" > /dev/null && echo OK
```

Expected: `OK`

- [ ] **Step 3: Write the .info sidecar**

```bash
cat > "$ORCA/filament/Eryone Wood PLA @Stephen.info" <<EOF
sync_info = create
user_id =
setting_id =
base_id =
updated_time = $(date +%s)
EOF
```

---

### Task 8: Create `Overture Matte Black ECO-PLA Older @Stephen` filament

**Files:**
- Create: `$ORCA/filament/Overture Matte Black ECO-PLA Older @Stephen.json`
- Create: `$ORCA/filament/Overture Matte Black ECO-PLA Older @Stephen.info`

- [ ] **Step 1: Write the filament JSON**

```bash
cat > "$ORCA/filament/Overture Matte Black ECO-PLA Older @Stephen.json" <<'EOF'
{
    "type": "filament",
    "name": "Overture Matte Black ECO-PLA Older @Stephen",
    "inherits": "Generic PLA @Stephen",
    "from": "User",
    "instantiation": "true",
    "filament_vendor": [
        "Overture"
    ],
    "filament_start_gcode": [
        "SET_PRESSURE_ADVANCE ADVANCE=0.0375"
    ],
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "version": "2.3.2.60"
}
EOF
```

- [ ] **Step 2: Validate JSON parse**

```bash
python3 -m json.tool "$ORCA/filament/Overture Matte Black ECO-PLA Older @Stephen.json" > /dev/null && echo OK
```

Expected: `OK`

- [ ] **Step 3: Write the .info sidecar**

```bash
cat > "$ORCA/filament/Overture Matte Black ECO-PLA Older @Stephen.info" <<EOF
sync_info = create
user_id =
setting_id =
base_id =
updated_time = $(date +%s)
EOF
```

---

### Task 9: Create `Pro White PLA @Stephen` filament

**Files:**
- Create: `$ORCA/filament/Pro White PLA @Stephen.json`
- Create: `$ORCA/filament/Pro White PLA @Stephen.info`

- [ ] **Step 1: Write the filament JSON**

```bash
cat > "$ORCA/filament/Pro White PLA @Stephen.json" <<'EOF'
{
    "type": "filament",
    "name": "Pro White PLA @Stephen",
    "inherits": "Generic PLA @Stephen",
    "from": "User",
    "instantiation": "true",
    "filament_vendor": [
        "Pro"
    ],
    "filament_start_gcode": [
        "SET_PRESSURE_ADVANCE ADVANCE=0.055"
    ],
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "version": "2.3.2.60"
}
EOF
```

- [ ] **Step 2: Validate JSON parse**

```bash
python3 -m json.tool "$ORCA/filament/Pro White PLA @Stephen.json" > /dev/null && echo OK
```

Expected: `OK`

- [ ] **Step 3: Write the .info sidecar**

```bash
cat > "$ORCA/filament/Pro White PLA @Stephen.info" <<EOF
sync_info = create
user_id =
setting_id =
base_id =
updated_time = $(date +%s)
EOF
```

---

### Task 10: Create `Elegoo Black PETG @Stephen` filament

**Files:**
- Create: `$ORCA/filament/Elegoo Black PETG @Stephen.json`
- Create: `$ORCA/filament/Elegoo Black PETG @Stephen.info`

- [ ] **Step 1: Write the filament JSON**

```bash
cat > "$ORCA/filament/Elegoo Black PETG @Stephen.json" <<'EOF'
{
    "type": "filament",
    "name": "Elegoo Black PETG @Stephen",
    "inherits": "Generic PETG @Stephen",
    "from": "User",
    "instantiation": "true",
    "filament_vendor": [
        "Elegoo"
    ],
    "filament_start_gcode": [
        "SET_PRESSURE_ADVANCE ADVANCE=0.0625"
    ],
    "compatible_printers": [
        "Stephen 0.4 Klipper CHT"
    ],
    "version": "2.3.2.60"
}
EOF
```

- [ ] **Step 2: Validate JSON parse**

```bash
python3 -m json.tool "$ORCA/filament/Elegoo Black PETG @Stephen.json" > /dev/null && echo OK
```

Expected: `OK`

- [ ] **Step 3: Write the .info sidecar**

```bash
cat > "$ORCA/filament/Elegoo Black PETG @Stephen.info" <<EOF
sync_info = create
user_id =
setting_id =
base_id =
updated_time = $(date +%s)
EOF
```

---

### Task 11: Verify all 9 profiles loaded

**Files:** none

- [ ] **Step 1: Inventory the user profile directories**

```bash
ls "$ORCA/process/" | grep "@Stephen"
ls "$ORCA/filament/" | grep "@Stephen"
```

Expected output (8 .json + 8 .info filament entries; 2 .json + 2 .info process entries):

Process:
```
0.20mm Standard @Stephen.info
0.20mm Standard @Stephen.json
0.20mm Structural ASA @Stephen 0.4 Klipper CHT.info
0.20mm Structural ASA @Stephen 0.4 Klipper CHT.json
```

Filament:
```
3D-HONGDAK Pink PLA @Stephen.info
3D-HONGDAK Pink PLA @Stephen.json
Elegoo Black PETG @Stephen.info
Elegoo Black PETG @Stephen.json
Eryone Wood PLA @Stephen.info
Eryone Wood PLA @Stephen.json
Generic PETG @Stephen.info
Generic PETG @Stephen.json
Generic PLA @Stephen.info
Generic PLA @Stephen.json
Generic TPU @Stephen.info
Generic TPU @Stephen.json
Overture Matte Black ECO-PLA Older @Stephen.info
Overture Matte Black ECO-PLA Older @Stephen.json
Polymaker Polylite ASA @Stephen MK4IS HF0.4.info
Polymaker Polylite ASA @Stephen MK4IS HF0.4.json
Pro White PLA @Stephen.info
Pro White PLA @Stephen.json
```

If any expected file is missing, return to the corresponding task and re-run.

- [ ] **Step 2: Launch OrcaSlicer**

```bash
open -a OrcaSlicer
```

- [ ] **Step 3: Confirm printer profile is selected**

In Orca's printer dropdown, select `Stephen 0.4 Klipper CHT`. If it is not visible, the printer profile has been broken — stop and investigate before proceeding.

- [ ] **Step 4: Confirm filament profiles appear in dropdown**

Open the filament profile dropdown. Confirm all 8 new profiles appear:

- `Generic PLA @Stephen`
- `Generic PETG @Stephen`
- `Generic TPU @Stephen`
- `3D-HONGDAK Pink PLA @Stephen`
- `Eryone Wood PLA @Stephen`
- `Overture Matte Black ECO-PLA Older @Stephen`
- `Pro White PLA @Stephen`
- `Elegoo Black PETG @Stephen`

If any profile is missing, the most likely causes are: parent does not load (check `inherits` field), `compatible_printers` not set as explicit array, or Orca was running when the file was created (it overwrote your changes on shutdown — quit Orca, recreate, relaunch).

- [ ] **Step 5: Confirm process profile appears in dropdown**

Open the process profile dropdown. Confirm `0.20mm Standard @Stephen` appears alongside `0.20mm Structural ASA @Stephen 0.4 Klipper CHT`.

- [ ] **Step 6: Inspect filament_notes for one generic**

Select `Generic PLA @Stephen`. In the filament settings panel, navigate to the Notes section (typically under the "Notes" tab or in an advanced settings panel). Confirm the towered/branded list appears as written in Task 3.

---

### Task 12: Slice verification — Generic PLA + custom process

**Files:**
- Output: `/tmp/Voron_Design_Cube_v7_PLA_verify.gcode`

- [ ] **Step 1: Slice a calibration cube**

In OrcaSlicer:
1. Load any small test object (e.g. `~/Downloads/Voron_Design_Cube_v7.stl` if available, otherwise any small box). The existing cube at `~/Downloads/Voron_Design_Cube_v7_0.2mm_ASA_MK4IS_1h8m.gcode` was sliced from this STL — the source file should be in your STL collection or downloadable from the Voron repo.
2. Select printer: `Stephen 0.4 Klipper CHT`.
3. Select filament: `Generic PLA @Stephen`.
4. Select process: `0.20mm Standard @Stephen`.
5. Click Slice.
6. Export G-code to `/tmp/Voron_Design_Cube_v7_PLA_verify.gcode`.

- [ ] **Step 2: Confirm SET_PRESSURE_ADVANCE is present and correct**

```bash
grep -n "SET_PRESSURE_ADVANCE" /tmp/Voron_Design_Cube_v7_PLA_verify.gcode
```

Expected: at least one line containing `SET_PRESSURE_ADVANCE ADVANCE=0.0475`. There should be no other `SET_PRESSURE_ADVANCE` lines (which would indicate Orca's own PA system is active).

- [ ] **Step 3: Confirm Marlin/Prusa linear-advance commands are absent**

```bash
grep -nE "^(M572|M900)" /tmp/Voron_Design_Cube_v7_PLA_verify.gcode || echo "OK — no M572/M900"
```

Expected: `OK — no M572/M900`. Any output from grep is a failure — investigate the parent profile chain.

- [ ] **Step 4: Confirm PRINT_START emits with the expected first-layer temps**

```bash
grep -n "^PRINT_START" /tmp/Voron_Design_Cube_v7_PLA_verify.gcode
```

Expected: `PRINT_START EXTRUDER_TEMP=215 BED_TEMP=60` (or similar — exact temps come from `Generic PLA @System` parent's `nozzle_temperature_initial_layer` and `hot_plate_temp_initial_layer`).

- [ ] **Step 5: Confirm acceleration commands match design**

```bash
grep -nE "^SET_VELOCITY_LIMIT" /tmp/Voron_Design_Cube_v7_PLA_verify.gcode | head -20
```

Expected: per-feature `SET_VELOCITY_LIMIT ACCEL=...` commands with values matching the design (2000 for outer wall, 3000 for inner wall and infill, 4000 for travel, 500 for initial layer). Exact emission depends on Orca's per-feature accel mode — at minimum confirm no value above 4500 (would indicate a setting was missed).

- [ ] **Step 6: Confirm the inheritance chain delivered HF max volumetric speed**

```bash
grep -n "^; max_volumetric_speed" /tmp/Voron_Design_Cube_v7_PLA_verify.gcode
```

Expected: `; max_volumetric_speed = 21` or similar (the value Orca recorded from the filament profile's `filament_max_volumetric_speed`).

---

### Task 13: Slice verification — branded filament inheritance

**Files:**
- Output: `/tmp/Voron_Design_Cube_v7_ProWhite_verify.gcode`

- [ ] **Step 1: Slice the same cube with a branded filament**

In OrcaSlicer:
1. Same printer + process as Task 12.
2. Filament: `Pro White PLA @Stephen`.
3. Slice and export to `/tmp/Voron_Design_Cube_v7_ProWhite_verify.gcode`.

- [ ] **Step 2: Confirm branded PA value applies**

```bash
grep -n "SET_PRESSURE_ADVANCE" /tmp/Voron_Design_Cube_v7_ProWhite_verify.gcode
```

Expected: a line containing `SET_PRESSURE_ADVANCE ADVANCE=0.055` (Pro White's value, not the generic 0.0475). This confirms the branded → generic inheritance chain delivers the brand-specific PA value correctly.

---

### Task 14: Print test — Generic PLA on the calibration cube

**Files:** none (physical print)

- [ ] **Step 1: Send the PLA gcode to the printer**

Upload `/tmp/Voron_Design_Cube_v7_PLA_verify.gcode` to Mainsail (mini1:7125) and start the print. Use a known-good PLA spool that is generally well-behaved; the print is for quality verification, not material qualification.

- [ ] **Step 2: Observe the first layer**

Watch the first layer go down. Confirm:
- Adhesion across the entire first layer.
- No visible underextrusion or overextrusion at the slower 30 mm/s + 60 mm/s initial-layer speeds.

If first-layer adhesion is bad, abort the print, check that PRINT_START completed bed mesh successfully, and confirm bed temp reached 60°C before the first layer started.

- [ ] **Step 3: Observe upper-layer outer walls**

Once past layer ~5, observe outer walls visually. Confirm:
- No obvious ringing on corners (if there is, IS may have drifted — re-run input shaping).
- No layer shift or skipped steps.

- [ ] **Step 4: Inspect the finished cube**

After completion:
- Outer walls should show no visible ringing/echoing.
- Layer adhesion: try to split at a layer line (you should not be able to split it cleanly).
- Top surface should be smoother than what stock MyKlipper defaults produced previously, due to lower top_surface_speed and acceleration.

If quality is materially worse than your prior PrusaSlicer prints, drop `outer_wall_speed` to 80 and `outer_wall_acceleration` to 1500 in `0.20mm Standard @Stephen.json` and retest. Note the change in the spec follow-ups section.

---

### Task 15: Update the design doc with results

**Files:**
- Modify: `/Users/stephen/git/Klipper-Input-Shaping-MK3S-Upgrade/docs/superpowers/specs/2026-04-23-orca-pla-petg-tpu-profiles-design.md`

- [ ] **Step 1: Mark spec status as implemented**

Edit the spec file's status line from:
```
Status: Design awaiting user review
```
to:
```
Status: Implemented 2026-04-23
```

- [ ] **Step 2: Commit the status change**

```bash
cd /Users/stephen/git/Klipper-Input-Shaping-MK3S-Upgrade
git add docs/superpowers/specs/2026-04-23-orca-pla-petg-tpu-profiles-design.md
git commit -m "$(cat <<'COMMIT_EOF'
Mark PLA/PETG/TPU profile spec as implemented

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
COMMIT_EOF
)"
```

Expected: clean commit, no pre-commit hook errors.
