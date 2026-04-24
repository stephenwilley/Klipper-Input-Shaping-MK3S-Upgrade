# 3dprinter-settings Repo Migration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate Klipper configs, OrcaSlicer profiles, and Docker Compose stack from drifting hand-copied snapshots into a single private repo with zero-drift sync via symlink (Mac) and Docker bind mount (mini1).

**Architecture:** Repo at `~/git/3dprinter-settings/` on both machines. Mac OrcaSlicer user dir symlinks into the repo. mini1 Docker Compose bind-mounts the repo's `klipper/mk3/` dir as `/opt/printer_data/config`. Mainsail/Orca writes land directly in the repo working tree, and `git status` reveals them immediately. Services renamed to `klipper-mk3` / `moonraker-mk3` to prepare for a second printer.

**Tech Stack:** Git, SSH, Docker Compose, Klipper, Moonraker, Mainsail, OrcaSlicer.

**Spec:** `docs/superpowers/specs/2026-04-23-repo-migration-design.md`

**Starting state (already done before this plan executes):**
- Private GitHub repo `stephenwilley/3dprinter-settings` exists.
- Clone at `~/git/3dprinter-settings/` on Mac with one commit containing only a stub `README.md`.
- Nothing done on mini1 yet. Old docker stack (`~/docker/klipper/`) is running normally. OrcaSlicer on Mac is closed and printer is idle (clean cutover window).

**Phases:**
- **Phase A (Tasks 1–6):** All work on Mac. Populate the new repo, push to GitHub. Running printer is untouched. Safe to pause at any point.
- **Phase B (Tasks 7–10):** Cutover. Brief window where the docker stack is down and Orca profiles are in flux. `.backup` dirs at every step give rollback.
- **Phase C (Tasks 11–12):** Post-cutover cleanup.

---

## File structure (end state)

```
~/git/3dprinter-settings/
├── README.md                        # rebuild runbook (Task 11)
├── .gitignore                       # Task 1
├── klipper/
│   └── mk3/
│       ├── printer.cfg              # from mini1 live (Task 2)
│       ├── macros.cfg               # from mini1 live
│       ├── mainsail.cfg             # from mini1 live
│       ├── moonraker.conf           # sanitized, includes secrets.conf (Task 2)
│       ├── steppers.cfg             # from mini1 live
│       ├── tmc2130.cfg              # from mini1 live
│       ├── einsy-rambo.cfg          # from mini1 live
│       ├── adxlmcu-BTT.cfg          # from mini1 live
│       ├── timelapse.cfg            # from mini1 live
│       ├── secrets.conf             # gitignored (created on mini1 at Task 7)
│       └── secrets.conf.example     # tracked template (Task 2)
├── orcaslicer/
│   └── user/default/
│       ├── machine/                 # from Mac Library (Task 3)
│       ├── process/                 # from Mac Library
│       └── filament/                # from Mac Library
├── docker/
│   ├── docker-compose.yml           # from mini1, renamed+repointed (Task 4)
│   ├── config.json                  # Mainsail printer list
│   └── timelapse.py                 # Moonraker component
└── docs/
    └── superpowers/
        ├── specs/                   # copied from old repo (Task 5)
        └── plans/                   # copied from old repo (Task 5)
```

---

## Task 1: Scaffold directories, .gitignore, skeleton README

**Files:**
- Create: `~/git/3dprinter-settings/.gitignore`
- Modify: `~/git/3dprinter-settings/README.md` (overwrite the stub)
- Create (empty): `klipper/mk3/`, `orcaslicer/user/default/{machine,process,filament}/`, `docker/`, `docs/superpowers/{specs,plans}/`

- [ ] **Step 1.1: Create directory tree**

```bash
cd ~/git/3dprinter-settings
mkdir -p klipper/mk3 \
         orcaslicer/user/default/machine \
         orcaslicer/user/default/process \
         orcaslicer/user/default/filament \
         docker \
         docs/superpowers/specs \
         docs/superpowers/plans
```

Verify:

```bash
find . -type d -not -path './.git*'
```

Expected output: 10 directories listed (repo root + 9 subdirectories).

- [ ] **Step 1.2: Write .gitignore**

Create `~/git/3dprinter-settings/.gitignore` with exactly:

```gitignore
# Secrets (Moonraker tokens etc. — split via [include])
klipper/*/secrets.conf

# Klipper auto-writes on SAVE_CONFIG / Z calibration
klipper/*/saved_variables.cfg

# OrcaSlicer backup files (profile edits write .bak-YYYYMMDD siblings)
orcaslicer/user/default/**/*.bak*
orcaslicer/user/default/**/*~

# macOS
.DS_Store

# Runtime state — not mounted into repo but defensive
gcodes/
logs/
database/
printer_data/
```

- [ ] **Step 1.3: Write skeleton README**

Overwrite `~/git/3dprinter-settings/README.md` with this placeholder (full content written at Task 11):

```markdown
# 3dprinter-settings

Private repo holding Klipper configuration, OrcaSlicer profiles, and Docker Compose setup for my 3D printing stack.

Rebuild runbook and daily workflow documentation comes in Task 11.
```

- [ ] **Step 1.4: Commit**

```bash
cd ~/git/3dprinter-settings
git add .gitignore README.md
git commit -m "Scaffold repo structure, .gitignore, README stub

Set up directory tree for klipper/, orcaslicer/, docker/, docs/. Empty
subdirectories are committed once the next tasks populate them."
```

Note: empty directories are not tracked by git — they become visible once populated in later tasks. That's fine.

---

## Task 2: Copy Klipper configs from mini1, split secrets

**Files:**
- Create: `klipper/mk3/*.cfg` and `klipper/mk3/*.conf` (9 files copied from mini1)
- Create: `klipper/mk3/secrets.conf.example` (template)
- Modify: `klipper/mk3/moonraker.conf` (remove `[power 3d_printer]` section, add `[include secrets.conf]`)

Authoritative source = mini1's **live** `~/docker/klipper/printer_data/config/`, not the old repo checkout. Live is what's actually running.

- [ ] **Step 2.1: Copy all .cfg and .conf files from mini1**

```bash
scp mini1:/home/stephen/docker/klipper/printer_data/config/{printer,macros,mainsail,steppers,tmc2130,einsy-rambo,adxlmcu-BTT,timelapse}.cfg \
    ~/git/3dprinter-settings/klipper/mk3/
scp mini1:/home/stephen/docker/klipper/printer_data/config/moonraker.conf \
    ~/git/3dprinter-settings/klipper/mk3/
```

Verify:

```bash
ls ~/git/3dprinter-settings/klipper/mk3/
```

Expected: 9 files — `adxlmcu-BTT.cfg einsy-rambo.cfg macros.cfg mainsail.cfg moonraker.conf printer.cfg steppers.cfg timelapse.cfg tmc2130.cfg`

- [ ] **Step 2.2: Confirm JWT is present in the copied moonraker.conf**

```bash
grep -n "^\[power 3d_printer\]\|^token:" ~/git/3dprinter-settings/klipper/mk3/moonraker.conf
```

Expected output: two matching lines (the section header and the token line).

- [ ] **Step 2.3: Remove `[power 3d_printer]` section from moonraker.conf and add [include]**

The section is 7 lines: the `[power 3d_printer]` header plus 6 config lines (`type`, `address`, `port`, `protocol`, `device`, `token`). Replace it with `[include secrets.conf]`.

Read the file to confirm exact structure, then use the Edit tool to replace the full block. The expected block to remove:

```
[power 3d_printer]
type: homeassistant
address: homeassistant.esstec.ca
port: 443
protocol: https
device: switch.3d_printer_power_switch
token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiI3Njk2Y2Y0YjJkYzU0OTRhOTE5MGU5YmYwNzgxNjRlOCIsImlhdCI6MTc3NTI1ODUyOSwiZXhwIjoyMDkwNjE4NTI5fQ.-wy8sw5-VO-PEPrgdIKF4lIrloMO1MIf8tz6OygQK0o
```

Replace with:

```
[include secrets.conf]
```

Verify JWT is gone:

```bash
grep -i "token:\|eyJh" ~/git/3dprinter-settings/klipper/mk3/moonraker.conf
```

Expected: no output.

Verify include line is present:

```bash
grep "^\[include secrets.conf\]" ~/git/3dprinter-settings/klipper/mk3/moonraker.conf
```

Expected: one matching line.

- [ ] **Step 2.4: Write secrets.conf.example template**

Create `~/git/3dprinter-settings/klipper/mk3/secrets.conf.example` with:

```ini
# Template for secrets.conf.
# On rebuild: cp secrets.conf.example secrets.conf, fill in real values.
# secrets.conf is gitignored.

[power 3d_printer]
type: homeassistant
address: homeassistant.esstec.ca
port: 443
protocol: https
device: switch.3d_printer_power_switch
token: YOUR_HOMEASSISTANT_LONG_LIVED_ACCESS_TOKEN
```

- [ ] **Step 2.5: Commit**

```bash
cd ~/git/3dprinter-settings
git add klipper/mk3/
git status
```

Before committing, verify `secrets.conf` is NOT staged (gitignored). The staged list should be 9 `.cfg`/`.conf` files + `secrets.conf.example`.

```bash
git commit -m "Import Klipper configs from mini1, split HA token into secrets.conf

Configs copied from mini1's live printer_data/config/ (authoritative).
The [power 3d_printer] section containing a Home Assistant JWT was
extracted into a gitignored secrets.conf; moonraker.conf now ends with
[include secrets.conf]. secrets.conf.example ships as a template."
```

---

## Task 3: Copy OrcaSlicer profiles

**Files:**
- Create: `orcaslicer/user/default/machine/*` (copies of 3 printer profiles + .info + .bak)
- Create: `orcaslicer/user/default/process/*` (copies of 6 process profiles + .info files)
- Create: `orcaslicer/user/default/filament/*` (copies of 18 filament profiles + .info files)

- [ ] **Step 3.1: Copy all three subdirectories**

```bash
cp -R "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/." \
      ~/git/3dprinter-settings/orcaslicer/user/default/machine/
cp -R "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/process/." \
      ~/git/3dprinter-settings/orcaslicer/user/default/process/
cp -R "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/filament/." \
      ~/git/3dprinter-settings/orcaslicer/user/default/filament/
```

Verify counts:

```bash
ls ~/git/3dprinter-settings/orcaslicer/user/default/machine/ | wc -l
ls ~/git/3dprinter-settings/orcaslicer/user/default/process/ | wc -l
ls ~/git/3dprinter-settings/orcaslicer/user/default/filament/ | wc -l
```

Expected: `3`, `6`, `18` respectively (matching what's in the Library).

- [ ] **Step 3.2: Verify .bak* files will be correctly excluded by .gitignore**

```bash
cd ~/git/3dprinter-settings
git status --porcelain | grep -i "\.bak" || echo "No .bak files showing as untracked — good."
```

Expected: "No .bak files showing as untracked — good." (Because `.gitignore` pattern `orcaslicer/user/default/**/*.bak*` excludes them. The `.bak` file stays on disk but git ignores it.)

- [ ] **Step 3.3: Commit**

```bash
cd ~/git/3dprinter-settings
git add orcaslicer/
git commit -m "Import OrcaSlicer user profiles (3 machine, 6 process, 18 filament)

Copied from ~/Library/Application Support/OrcaSlicer/user/default/.
.bak files are gitignored."
```

---

## Task 4: Copy and adapt docker files

**Files:**
- Create: `docker/docker-compose.yml` (from mini1, with edits)
- Create: `docker/config.json` (from mini1, unchanged)
- Create: `docker/timelapse.py` (from mini1, unchanged)

**Edits to docker-compose.yml:**
1. Rename service `klipper` → `klipper-mk3` (including `container_name`).
2. Rename service `moonraker` → `moonraker-mk3` (including `container_name`).
3. Update `depends_on` references for moonraker (was `klipper`) and mainsail (was `moonraker`).
4. Change config-related mount host paths:
   - `./printer_data/config` → `../klipper/mk3` (both services)
   - `./config.json` → `./config.json` (already correct — config.json now sits next to compose file)
   - `./timelapse.py` → `./timelapse.py` (same — sits next to compose file)
5. Change runtime-state mount host paths to absolute paths on mini1:
   - `./printer_data` → `/home/stephen/docker/klipper/printer_data` (both services)
   - `./printer_data/database`, `./printer_data/gcodes`, `./printer_data/logs` → absolute equivalents
   - `./run` → `/home/stephen/docker/klipper/run`
   - `./klipper` → `/home/stephen/docker/klipper/klipper` (firmware source for moonraker)
6. Preserve ALL other mounts exactly (including the seemingly-redundant overlay mounts — the container images declare them and removing any causes empty-directory bugs).

- [ ] **Step 4.1: Copy the three files from mini1**

```bash
scp mini1:/home/stephen/docker/klipper/docker-compose.yml ~/git/3dprinter-settings/docker/
scp mini1:/home/stephen/docker/klipper/config.json       ~/git/3dprinter-settings/docker/
scp mini1:/home/stephen/docker/klipper/timelapse.py      ~/git/3dprinter-settings/docker/
```

Verify:

```bash
ls ~/git/3dprinter-settings/docker/
```

Expected: `config.json docker-compose.yml timelapse.py`

- [ ] **Step 4.2: Rewrite docker-compose.yml with renames + path updates**

Use Write to replace `~/git/3dprinter-settings/docker/docker-compose.yml` with:

```yaml
services:
  klipper-mk3:
    image: mkuf/klipper:latest
    container_name: klipper-mk3
    restart: always
    privileged: true
    volumes:
      - /dev:/dev
      - /home/stephen/docker/klipper/printer_data:/opt/printer_data
      - ../klipper/mk3:/opt/printer_data/config
      - /home/stephen/docker/klipper/printer_data/gcodes:/opt/printer_data/gcodes
      - /home/stephen/docker/klipper/printer_data/logs:/opt/printer_data/logs
      - /home/stephen/docker/klipper/run:/opt/printer_data/run
    labels:
      - "com.centurylinklabs.watchtower.enable=false"
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

  moonraker-mk3:
    image: mkuf/moonraker:latest
    container_name: moonraker-mk3
    restart: always
    ports:
      - "7125:7125" # Port for the Moonraker API
    volumes:
      - /home/stephen/docker/klipper/printer_data:/opt/printer_data
      - ../klipper/mk3:/opt/printer_data/config
      - /home/stephen/docker/klipper/printer_data/database:/opt/printer_data/database
      - /home/stephen/docker/klipper/printer_data/gcodes:/opt/printer_data/gcodes
      - /home/stephen/docker/klipper/printer_data/logs:/opt/printer_data/logs
      - /home/stephen/docker/klipper/klipper:/opt/klipper
      - /home/stephen/docker/klipper/run:/opt/printer_data/run
      - /bin/ffmpeg:/usr/bin/ffmpeg:ro
      - ./timelapse.py:/opt/moonraker/moonraker/components/timelapse.py:ro
    depends_on:
      - klipper-mk3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

  mainsail:
    image: ghcr.io/mainsail-crew/mainsail:latest
    container_name: mainsail
    restart: always
    ports:
      - "80:80"
    volumes:
      - /home/stephen/docker/klipper/printer_data:/home/mainsail
      - ./config.json:/usr/share/nginx/html/config.json
    depends_on:
      - moonraker-mk3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
```

Verify syntactic validity from the Mac:

```bash
cd ~/git/3dprinter-settings/docker
docker compose config --quiet 2>&1 || echo "Compose validation failed"
```

Expected: no error output (may warn about the `version` field being obsolete if the original had one — that's fine). If `docker` isn't installed on the Mac, skip this check — we'll validate on mini1 at Task 9.

- [ ] **Step 4.3: Commit**

```bash
cd ~/git/3dprinter-settings
git add docker/
git commit -m "Import Docker Compose stack, rename services to *-mk3

Services klipper and moonraker renamed to klipper-mk3 / moonraker-mk3
to prepare for a second printer instance. Mainsail stays singular
(shared across printers via config.json instances array).

Config-related mounts now bind the repo's klipper/mk3/ dir and the
docker/ dir's config.json and timelapse.py. Runtime-state mounts use
absolute paths pointing at /home/stephen/docker/klipper/ on mini1
where database/, gcodes/, logs/, and the Klipper firmware source
continue to live outside the repo."
```

---

## Task 5: Copy design docs from old repo

**Files:**
- Create: `docs/superpowers/specs/*.md` (4 specs from old repo)
- Create: `docs/superpowers/plans/*.md` (3 plans from old repo)

- [ ] **Step 5.1: Copy specs**

```bash
cp "/Users/stephen/git/Klipper-Input-Shaping-MK3S-Upgrade/docs/superpowers/specs/"*.md \
   ~/git/3dprinter-settings/docs/superpowers/specs/
cp "/Users/stephen/git/Klipper-Input-Shaping-MK3S-Upgrade/docs/superpowers/plans/"*.md \
   ~/git/3dprinter-settings/docs/superpowers/plans/
```

Verify:

```bash
ls ~/git/3dprinter-settings/docs/superpowers/specs/
ls ~/git/3dprinter-settings/docs/superpowers/plans/
```

Expected: specs has 4 `.md` files (two ASA, one PLA/PETG/TPU, one repo-migration design). Plans has 3 `.md` files (two corresponding implementation plans plus this plan).

- [ ] **Step 5.2: Commit**

```bash
cd ~/git/3dprinter-settings
git add docs/
git commit -m "Import design specs and plans from old repo

Includes the repo-migration design and plan that produced this very
state, plus the earlier OrcaSlicer ASA and PLA/PETG/TPU profile work."
```

---

## Task 6: Push to GitHub

- [ ] **Step 6.1: Push**

```bash
cd ~/git/3dprinter-settings
git push origin main
```

Verify remote state:

```bash
git log --oneline origin/main
```

Expected: 5 or 6 commits pushed (one per earlier task, plus the pre-existing stub).

**End of Phase A. Phase B (the cutover) starts next — confirm with user before proceeding.**

---

## Task 7: Clone on mini1 and populate secrets.conf

- [ ] **Step 7.1: Clone repo on mini1**

```bash
ssh mini1 'cd ~ && mkdir -p git && cd git && git clone git@github.com:stephenwilley/3dprinter-settings.git'
```

Verify:

```bash
ssh mini1 'ls -la ~/git/3dprinter-settings/'
```

Expected: the full tree, including `.git/`, `README.md`, `klipper/mk3/`, etc.

If mini1 doesn't yet have the SSH key authorized for GitHub, this will fail. In that case: ensure mini1's SSH public key is added to GitHub (Settings → SSH Keys), then retry.

- [ ] **Step 7.2: Create secrets.conf on mini1 from template**

```bash
ssh mini1 'cp ~/git/3dprinter-settings/klipper/mk3/secrets.conf.example \
              ~/git/3dprinter-settings/klipper/mk3/secrets.conf'
```

- [ ] **Step 7.3: Paste the real HA token into secrets.conf on mini1**

The real token (currently in mini1's live `~/docker/klipper/printer_data/config/moonraker.conf` at line 42) must go into the `token:` field of the new secrets.conf.

Inspect current token:

```bash
ssh mini1 'grep "^token:" ~/docker/klipper/printer_data/config/moonraker.conf'
```

Copy the value and use SSH to write it into secrets.conf (replace the whole token line):

```bash
ssh mini1 'TOKEN=$(grep "^token:" ~/docker/klipper/printer_data/config/moonraker.conf) && \
           sed -i "s|^token:.*|${TOKEN}|" ~/git/3dprinter-settings/klipper/mk3/secrets.conf'
```

Verify:

```bash
ssh mini1 'grep "^token:" ~/git/3dprinter-settings/klipper/mk3/secrets.conf | head -c 30 && echo "..."'
```

Expected: `token: eyJhbGciOiJIUzI1NiIsInR5...` (first 30 chars of a JWT, confirming it's not still the placeholder).

Confirm secrets.conf is gitignored (no commit):

```bash
ssh mini1 'cd ~/git/3dprinter-settings && git status --porcelain klipper/mk3/secrets.conf'
```

Expected: no output (file is ignored).

---

## Task 8: Stop old docker stack, back up current config

- [ ] **Step 8.1: Stop running containers**

```bash
ssh mini1 'cd ~/docker/klipper && docker compose down'
```

Verify all three containers are stopped:

```bash
ssh mini1 'docker ps -a --filter name=klipper --filter name=moonraker --filter name=mainsail --format "{{.Names}} {{.Status}}"'
```

Expected: either no output (containers removed by `down`) OR all three showing `Exited`.

- [ ] **Step 8.2: Back up current live config directory**

```bash
ssh mini1 'mv ~/docker/klipper/printer_data/config ~/docker/klipper/printer_data/config.backup.$(date +%Y%m%d)'
```

Verify:

```bash
ssh mini1 'ls -la ~/docker/klipper/printer_data/ | grep config'
```

Expected: one `config.backup.YYYYMMDD` directory. No `config` directory (it's been renamed).

The bind mount in the new compose will expose `~/git/3dprinter-settings/klipper/mk3/` as `/opt/printer_data/config` inside the container — no `config` dir needed at `printer_data/` on the host.

- [ ] **Step 8.3: Back up the old docker-compose.yml as a rollback reference**

```bash
ssh mini1 'cp ~/docker/klipper/docker-compose.yml ~/docker/klipper/docker-compose.yml.backup.$(date +%Y%m%d)'
```

---

## Task 9: Bring up new docker stack

- [ ] **Step 9.1: Start new stack from the repo's docker-compose.yml**

```bash
ssh mini1 'cd ~/git/3dprinter-settings/docker && docker compose up -d'
```

Expected: three containers created — `klipper-mk3`, `moonraker-mk3`, `mainsail` — with status `Started`.

- [ ] **Step 9.2: Verify containers are healthy**

```bash
ssh mini1 'docker ps --format "{{.Names}} {{.Status}}"'
```

Expected: three lines, each showing `Up N seconds` (or longer). No `Restarting` or `Exited`.

- [ ] **Step 9.3: Verify Klipper connected to MCU**

```bash
ssh mini1 'docker logs klipper-mk3 2>&1 | tail -20'
```

Expected: no `Error` lines; final lines show `Build file` or `Loaded MCU` or similar indicating Klipper started normally. If you see `Unable to open serial port` or `mcu 'mcu': Unable to connect`, the MCU path may differ — check `/dev` mounts and the board's serial path in `printer.cfg`.

- [ ] **Step 9.4: Verify Moonraker API responds**

```bash
ssh mini1 'curl -sf http://localhost:7125/server/info | head -c 100 && echo'
```

Expected: a JSON snippet starting with `{"result":{"klippy_connected":true,"klippy_state":"ready"…`.

- [ ] **Step 9.5: Verify Mainsail serves**

```bash
curl -sf http://mini1/ -o /dev/null && echo "Mainsail OK" || echo "Mainsail FAILED"
```

Expected: `Mainsail OK`.

- [ ] **Step 9.6: Live-sync smoke test**

Edit any comment line in `~/git/3dprinter-settings/klipper/mk3/printer.cfg` on mini1 (or Mac then push/pull — but mini1 is fastest right now):

```bash
ssh mini1 "cd ~/git/3dprinter-settings/klipper/mk3 && \
           sed -i 's|^# Printer configuration|# Printer configuration (sync-test)|' printer.cfg && \
           git diff --stat printer.cfg"
```

Expected: git shows the `printer.cfg` as changed. Inside Mainsail, the Configuration view shows the modified comment — proving the bind mount is exposing the repo dir as the container's config.

Revert the test edit:

```bash
ssh mini1 'cd ~/git/3dprinter-settings/klipper/mk3 && git checkout -- printer.cfg'
```

**Stop here if any of 9.1–9.6 fails.** Rollback: `ssh mini1 'cd ~/git/3dprinter-settings/docker && docker compose down && cd ~/docker/klipper && mv printer_data/config.backup.YYYYMMDD printer_data/config && docker compose up -d'`.

---

## Task 10: Symlink OrcaSlicer on Mac

**Precondition:** OrcaSlicer is closed. (User confirmed at design time; recheck before executing.)

- [ ] **Step 10.1: Confirm OrcaSlicer is closed**

```bash
pgrep -lf OrcaSlicer || echo "OrcaSlicer not running — OK"
```

Expected: `OrcaSlicer not running — OK`. If a process is listed, close OrcaSlicer fully before proceeding.

- [ ] **Step 10.2: Back up current user/default/**

```bash
mv "/Users/stephen/Library/Application Support/OrcaSlicer/user/default" \
   "/Users/stephen/Library/Application Support/OrcaSlicer/user/default.backup.$(date +%Y%m%d)"
```

Verify:

```bash
ls "/Users/stephen/Library/Application Support/OrcaSlicer/user/" | grep -E "^default"
```

Expected: a `default.backup.YYYYMMDD` entry and no plain `default` entry yet.

- [ ] **Step 10.3: Create symlink to repo**

```bash
ln -s "/Users/stephen/git/3dprinter-settings/orcaslicer/user/default" \
      "/Users/stephen/Library/Application Support/OrcaSlicer/user/default"
```

Verify:

```bash
ls -la "/Users/stephen/Library/Application Support/OrcaSlicer/user/default"
```

Expected: output shows `default -> /Users/stephen/git/3dprinter-settings/orcaslicer/user/default`.

Verify the symlink resolves to a directory with expected contents:

```bash
ls "/Users/stephen/Library/Application Support/OrcaSlicer/user/default/machine/"
```

Expected: the 3 machine profile files (`Stephen 0.4 Klipper CHT.json` etc.).

- [ ] **Step 10.4: Open OrcaSlicer, verify profiles present**

This step is manual — the user opens OrcaSlicer and confirms:
- The `Stephen 0.4 Klipper CHT` printer appears in the printer dropdown.
- Process dropdown contains `0.20mm Standard @Stephen`, `0.20mm Structural ASA @Stephen`, `0.20mm Flex @Stephen`.
- Filament dropdown contains the custom filaments (Generic PLA @Stephen, etc.).
- Slicing a small test STL produces output with no errors.

If any profile is missing or Orca errors on load, the rollback is: quit Orca, `rm "/Users/stephen/Library/Application Support/OrcaSlicer/user/default"` (removes the symlink, not the target), `mv ".../default.backup.YYYYMMDD" ".../default"`, reopen Orca.

---

## Task 11: Write comprehensive README (rebuild runbook)

**Files:**
- Modify: `~/git/3dprinter-settings/README.md` (full rewrite from the skeleton)

- [ ] **Step 11.1: Write full README**

Overwrite `~/git/3dprinter-settings/README.md` with:

```markdown
# 3dprinter-settings

Private repo holding Klipper configuration, OrcaSlicer profiles, and Docker Compose setup for my 3D printing stack.

## Layout

- `klipper/mk3/` — Klipper, Moonraker, Mainsail configuration for the Prusa Mk3 (Bear upgrade, Revo 6 HF, Obxidian HF nozzle).
- `orcaslicer/user/default/` — OrcaSlicer custom printer, process, and filament profiles.
- `docker/` — Docker Compose stack (`klipper-mk3`, `moonraker-mk3`, `mainsail` services) with supporting files (`config.json` for Mainsail's printer list; `timelapse.py` Moonraker component).
- `docs/superpowers/` — Design specs and implementation plans.

## Sync model

This repo is the source of truth for configs, with live locations pointing at it:

- **Mac:** `~/Library/Application Support/OrcaSlicer/user/default/` is a symlink → `~/git/3dprinter-settings/orcaslicer/user/default/`.
- **mini1:** Docker Compose bind-mounts `~/git/3dprinter-settings/klipper/mk3/` as `/opt/printer_data/config/` inside the `klipper-mk3` and `moonraker-mk3` containers.

Edits made in Mainsail or OrcaSlicer's UI land directly in the repo working tree. `git status` on either machine shows pending changes.

### Daily workflow

1. Edit via Mainsail (browser) or OrcaSlicer's UI.
2. `git status` → `git diff` to review.
3. `git commit -am "..."` → `git push`.
4. Before editing on the other machine: `git pull` first.

## Rebuild runbook

### Rebuilding on a new mini1 (Linux host)

1. Install Docker + Docker Compose.
2. Clone:

   ```bash
   git clone git@github.com:stephenwilley/3dprinter-settings.git ~/git/3dprinter-settings
   ```

3. Create runtime directories:

   ```bash
   mkdir -p ~/docker/klipper/printer_data/{database,gcodes,logs}
   mkdir -p ~/docker/klipper/run
   ```

4. Clone Klipper firmware source (not in this repo):

   ```bash
   git clone https://github.com/Klipper3d/klipper.git ~/docker/klipper/klipper
   ```

5. Create `secrets.conf` from template and paste tokens:

   ```bash
   cp ~/git/3dprinter-settings/klipper/mk3/secrets.conf.example \
      ~/git/3dprinter-settings/klipper/mk3/secrets.conf
   # edit secrets.conf, replace YOUR_HOMEASSISTANT_LONG_LIVED_ACCESS_TOKEN with a real token
   ```

6. Flash MCU firmware if the board changed (see `klipper/mk3/einsy-rambo.cfg` and `adxlmcu-BTT.cfg` for board IDs).

7. Bring up the stack:

   ```bash
   cd ~/git/3dprinter-settings/docker && docker compose up -d
   ```

8. Verify Mainsail at `http://<hostname>/`.

### Rebuilding on a new Mac

1. Install OrcaSlicer (DMG).
2. Clone repo:

   ```bash
   git clone git@github.com:stephenwilley/3dprinter-settings.git ~/git/3dprinter-settings
   ```

3. Open OrcaSlicer once to let it create `~/Library/Application Support/OrcaSlicer/user/default/`, then quit.
4. Replace the directory with a symlink:

   ```bash
   rm -rf "/Users/$(whoami)/Library/Application Support/OrcaSlicer/user/default"
   ln -s ~/git/3dprinter-settings/orcaslicer/user/default \
         "/Users/$(whoami)/Library/Application Support/OrcaSlicer/user/default"
   ```

5. Reopen OrcaSlicer — profiles present.

## Adding a second printer

1. Create `klipper/<printer-name>/` with the printer's configs (`printer.cfg`, `moonraker.conf`, `mainsail.cfg`, `secrets.conf.example`, etc.).
2. Add `klipper-<printer>` and `moonraker-<printer>` services to `docker/docker-compose.yml`. Moonraker port stepped (7125 → 7126).
3. Add new runtime-state dirs: `~/docker/klipper/printer_data-<printer>/{database,gcodes,logs}/`. Update the compose to bind-mount them.
4. Add instance to `docker/config.json`:

   ```json
   { "name": "Trident", "hostname": "mini1", "port": 7126 }
   ```

5. Create an OrcaSlicer printer profile in `orcaslicer/user/default/machine/` and extend `compatible_printers` in process/filament profiles that should apply to both printers.
6. `docker compose up -d` — the new services come up without disrupting the existing one.

## What is NOT tracked

- `klipper/*/secrets.conf` — the real token file. Template is `secrets.conf.example`.
- `klipper/*/saved_variables.cfg` — Klipper auto-writes this on Z offset calibration / `SAVE_CONFIG`.
- `orcaslicer/user/default/**/*.bak*` — OrcaSlicer's profile edit backups.
- Runtime state on mini1 (`~/docker/klipper/printer_data/database/`, `gcodes/`, `logs/`).
- Klipper firmware source (`~/docker/klipper/klipper/`).
- `.DS_Store`.
```

- [ ] **Step 11.2: Commit and push**

```bash
cd ~/git/3dprinter-settings
git add README.md
git commit -m "Write full rebuild runbook and daily workflow README

Replaces stub README with complete rebuild instructions for both
machines, the sync model description, and the process for adding
a second printer."
git push origin main
```

---

## Task 12: Move Claude project memory to new path

- [ ] **Step 12.1: Create new Claude project memory directory**

```bash
mkdir -p ~/.claude/projects/-Users-stephen-git-3dprinter-settings/memory
```

- [ ] **Step 12.2: Move memory files from old project**

```bash
mv ~/.claude/projects/-Users-stephen-git-Klipper-Input-Shaping-MK3S-Upgrade-1---Primary-Configuration-Files/memory/*.md \
   ~/.claude/projects/-Users-stephen-git-3dprinter-settings/memory/
```

Verify:

```bash
ls ~/.claude/projects/-Users-stephen-git-3dprinter-settings/memory/
```

Expected: `MEMORY.md` and `orca_layer_height_processes_followup.md` (and any other memory files that were in the old project).

- [ ] **Step 12.3: Verify the old project's memory dir is now empty of .md files**

```bash
ls ~/.claude/projects/-Users-stephen-git-Klipper-Input-Shaping-MK3S-Upgrade-1---Primary-Configuration-Files/memory/
```

Expected: either empty or contains only transcript artifacts (no `.md` files). The `.jsonl` transcript files stay in the old project dir — they're immutable history and Claude Code keys them by cwd.

---

## Post-migration: archive old repo (do this ~1 week after Task 12 completes cleanly)

This is intentionally NOT part of the immediate plan. After roughly a week of the new setup running without surprises, on GitHub:

1. Navigate to `stephenwilley/Klipper-Input-Shaping-MK3S-Upgrade` → Settings → General.
2. Scroll to "Danger Zone" → "Archive this repository" → confirm.

No git operations needed — archiving just flags the repo read-only. All commits and issues are preserved.

---

## Verification summary

After Task 12, the following should all be true:

- `docker ps` on mini1 shows `klipper-mk3`, `moonraker-mk3`, `mainsail` all `Up`.
- Mainsail at `http://mini1/` shows Klipper in `ready` state.
- Editing `printer.cfg` via Mainsail shows up as a `git status` diff in `~/git/3dprinter-settings/` on mini1 within seconds.
- OrcaSlicer on Mac shows the `Stephen 0.4 Klipper CHT` printer and all `@Stephen` profiles.
- Editing a filament PA value in Orca UI shows up as a `git status` diff on Mac.
- Running Claude Code in `~/git/3dprinter-settings/` on Mac surfaces the migrated memory (e.g., the 0.10mm/0.30mm process followup note).
- `~/docker/klipper/printer_data/config.backup.YYYYMMDD/` still exists on mini1 (rollback reference).
- `~/Library/Application Support/OrcaSlicer/user/default.backup.YYYYMMDD/` still exists on Mac (rollback reference).

If any of those fail, roll back using the `.backup.YYYYMMDD` directories and investigate before retrying.
