# Repo Migration: `3dprinter-settings`

**Date:** 2026-04-23
**Status:** Design approved, ready for planning

## Problem

The current working directory is a fork of `charminULTRA/Klipper-Input-Shaping-MK3S-Upgrade` — a Mk3 conversion project. Work done here has outgrown that scope:
- Custom OrcaSlicer printer, process, and filament profiles (3 printer, 6 process, 18 filament files) live outside any repo, in `~/Library/Application Support/OrcaSlicer/user/default/` on the Mac.
- Docker Compose setup on mini1 (`~/docker/klipper/`) is not version-controlled.
- Klipper configs in `~/docker/klipper/printer_data/config/` are not a git working tree; the repo checkout drifts from what's actually running. Current practice: edit via Mainsail (writes to `printer_data/config/`), occasionally hand-copy into the repo. No sync guarantees.
- A second printer (likely a Voron Trident) is planned, and the current structure has no clear place for a second Klipper instance.
- `moonraker.conf` on mini1 contains a live JWT token that must not end up in git.

## Goals

1. Single repo containing everything needed to rebuild the 3D-printing stack on either machine (mini1 or Mac) from bare install.
2. Zero drift between committed configs and what Klipper/Mainsail actually run.
3. Extensible to a second printer without restructuring.
4. No secrets in git.
5. Preserves existing design docs and Claude project memory relevant to ongoing OrcaSlicer work.

## Non-goals

- Version-controlling runtime state (gcodes, logs, Moonraker database). Runtime state stays where it is and is deliberately gitignored.
- Version-controlling the Klipper firmware source. It's an upstream clone at `~/docker/klipper/klipper/`; the rebuild runbook instructs re-cloning it.
- Migrating git history from the old repo. The old repo will be archived read-only on GitHub after a successful transition.

## Target repo

- **Name:** `stephenwilley/3dprinter-settings`
- **Visibility:** Private
- **Local path:** `~/git/3dprinter-settings/` on both Mac and mini1

## Repo layout

```
3dprinter-settings/
├── README.md                        # Rebuild runbook, daily workflow
├── .gitignore
├── klipper/
│   └── mk3/
│       ├── printer.cfg
│       ├── macros.cfg
│       ├── mainsail.cfg
│       ├── moonraker.conf           # includes secrets.conf; no tokens inline
│       ├── steppers.cfg
│       ├── tmc2130.cfg
│       ├── einsy-rambo.cfg
│       ├── adxlmcu-BTT.cfg
│       ├── timelapse.cfg
│       └── secrets.conf.example     # template; real secrets.conf gitignored
├── orcaslicer/
│   └── user/default/
│       ├── machine/                 # custom printer profiles
│       ├── process/                 # custom process profiles
│       └── filament/                # custom filament profiles
├── docker/
│   ├── docker-compose.yml           # all services, single file
│   ├── config.json                  # Mainsail printer list (used by mainsail service)
│   └── timelapse.py                 # moonraker component (mounted read-only)
└── docs/
    └── superpowers/
        ├── specs/                   # brainstorming/design docs
        └── plans/                   # implementation plans
```

When a second printer is added: `klipper/trident/` + new services `klipper-trident` / `moonraker-trident` in the same `docker-compose.yml`, new entry in `docker/config.json`, new printer profile in `orcaslicer/user/default/machine/`.

## Sync model

Hybrid, chosen because each machine's environment has a cleaner option than the other.

### Mac: symlink

Replace `~/Library/Application Support/OrcaSlicer/user/default/` with a symlink to `~/git/3dprinter-settings/orcaslicer/user/default/`. OrcaSlicer is not sandboxed (DMG install), so it follows the symlink transparently. Profile edits made in Orca's UI land directly in the repo working tree.

### mini1: Docker bind mount

Docker Compose bind-mounts `~/git/3dprinter-settings/klipper/mk3/` as `/opt/printer_data/config` in both `klipper-mk3` and `moonraker-mk3` services. Similarly bind-mount `~/git/3dprinter-settings/docker/config.json` and `~/git/3dprinter-settings/docker/timelapse.py` at their current container paths.

The existing compose file's full set of mount declarations is preserved. The `mkuf/klipper` and `mkuf/moonraker` images declare internal mount points (notably for `database/`, `gcodes/`, `logs/`, `run/`, `/opt/klipper`, `/opt/printer_data`); removing any of them causes the container to present empty directories at those paths. **Only the host-side path of config-related mounts changes** — mount list itself stays identical.

Runtime state (`printer_data/database/`, `printer_data/gcodes/`, `printer_data/logs/`, `run/`) stays at `~/docker/klipper/` on mini1, outside the repo. Klipper firmware source stays at `~/docker/klipper/klipper/`, outside the repo.

### Daily workflow

1. Edit happens wherever (Mainsail in browser, Orca UI, direct file edit).
2. Writes land in the repo working tree on one machine.
3. `git status` shows pending changes. Commit when a logical unit of change is complete.
4. Push from the machine where the edit was made, pull on the other.

The Mac and mini1 are two working trees synchronized through GitHub as the central hub.

## Container naming

Current compose uses `klipper` / `moonraker` / `mainsail`. Rename to `klipper-mk3` / `moonraker-mk3` / `mainsail` (mainsail unchanged — it's shared across printers). `container_name:` fields match the service names. `depends_on:` references updated accordingly.

This is done now (single printer) so no further renaming is needed when Trident arrives — we just add `klipper-trident` / `moonraker-trident` services in the same file.

## Secrets handling

`moonraker.conf` currently contains a JWT token inline. After migration:

- `klipper/mk3/moonraker.conf` (tracked): all non-secret configuration, ending with `[include secrets.conf]`.
- `klipper/mk3/secrets.conf` (gitignored): only the sections that contain tokens.
- `klipper/mk3/secrets.conf.example` (tracked): the same structure with dummy/placeholder values. On rebuild, copy to `secrets.conf` and paste real tokens.

## .gitignore

- `klipper/*/secrets.conf`
- `klipper/*/saved_variables.cfg` (Klipper auto-writes on Z calibration / SAVE_CONFIG)
- `orcaslicer/user/default/**/*.bak*` (OrcaSlicer backup files)
- `.DS_Store`
- Defensive (not mounted into repo anyway): any `gcodes/`, `logs/`, `database/` at repo root.

## Rebuild runbook (README.md)

### Losing mini1

1. Install Docker + Docker Compose on new Linux host.
2. `git clone git@github.com:stephenwilley/3dprinter-settings.git ~/git/3dprinter-settings`.
3. Create runtime directories: `mkdir -p ~/docker/klipper/printer_data/{database,gcodes,logs} ~/docker/klipper/run`.
4. Clone Klipper firmware: `git clone https://github.com/Klipper3d/klipper.git ~/docker/klipper/klipper`.
5. `cp ~/git/3dprinter-settings/klipper/mk3/secrets.conf.example ~/git/3dprinter-settings/klipper/mk3/secrets.conf`, paste tokens.
6. Flash MCU firmware if board changed (see `klipper/mk3/` for MCU config references).
7. `cd ~/git/3dprinter-settings/docker && docker compose up -d`.
8. Confirm Mainsail at `http://mini1/` shows Klipper connected.

### Losing Mac

1. Install OrcaSlicer (DMG).
2. `git clone git@github.com:stephenwilley/3dprinter-settings.git ~/git/3dprinter-settings`.
3. Open OrcaSlicer once to let it create `~/Library/Application Support/OrcaSlicer/user/default/`, then quit.
4. `rm -rf "~/Library/Application Support/OrcaSlicer/user/default"` and replace with symlink: `ln -s ~/git/3dprinter-settings/orcaslicer/user/default "~/Library/Application Support/OrcaSlicer/user/default"`.
5. Reopen OrcaSlicer — profiles appear.

### Daily workflow

- Edit via Mainsail (mini1) or Orca UI (Mac). Changes land in the repo working tree.
- On the machine that edited: `git status`, `git commit`, `git push`.
- On the other machine: `git pull` before editing to avoid conflicts.

## Migration path

Each step leaves the printer in a known-good state. `.backup` directories provide rollback at every cutover.

1. Create empty private repo `stephenwilley/3dprinter-settings` on GitHub.
2. `git clone` onto Mac at `~/git/3dprinter-settings/`; scaffold directory tree, `.gitignore`, skeleton README.
3. Copy Klipper configs from mini1's live `printer_data/config/` (authoritative) into `klipper/mk3/`. Split JWT from `moonraker.conf` into `secrets.conf`; write `secrets.conf.example` template.
4. Copy OrcaSlicer profiles from `~/Library/Application Support/OrcaSlicer/user/default/` into `orcaslicer/user/default/`.
5. Copy `docker-compose.yml`, `config.json`, `timelapse.py` from mini1 into `docker/`. Rename services to `klipper-mk3`/`moonraker-mk3`. Update config-related mount host paths to point at the repo dir. Preserve all other mount declarations.
6. Copy `docs/superpowers/specs/*` and `docs/superpowers/plans/*` from the current repo (including this design doc) into the new repo.
7. First commit + push to GitHub.
8. Clone on mini1: `git clone git@github.com:stephenwilley/3dprinter-settings.git ~/git/3dprinter-settings`.
9. mini1 cutover: `docker compose down` (old compose), `mv ~/docker/klipper/printer_data/config ~/docker/klipper/printer_data/config.backup`, `cd ~/git/3dprinter-settings/docker && docker compose up -d` (new compose). Verify Klipper comes up clean; test G28.
10. Mac cutover: quit OrcaSlicer, `mv "~/Library/Application Support/OrcaSlicer/user/default" "~/Library/Application Support/OrcaSlicer/user/default.backup"`, symlink to repo, reopen OrcaSlicer, verify profiles present.
11. After ~1 week of running successfully: archive `stephenwilley/Klipper-Input-Shaping-MK3S-Upgrade` on GitHub (read-only).

## Claude project memory

Claude Code keys project memory by cwd slug. Current path: `~/.claude/projects/-Users-stephen-git-Klipper-Input-Shaping-MK3S-Upgrade-1---Primary-Configuration-Files/`. New working path will create `~/.claude/projects/-Users-stephen-git-3dprinter-settings/`.

- Create `~/.claude/projects/-Users-stephen-git-3dprinter-settings/memory/`.
- **Move** `MEMORY.md` and `orca_layer_height_processes_followup.md` from old project's `memory/` dir to the new one. (Old repo being archived — no reason to keep memory in two places.)
- Transcript `.jsonl` files stay at the old path. They're immutable history; moving them would confuse Claude Code.
- The repo itself contains no Claude state — all memory lives under `~/.claude/` and is machine-local by design.

## Validation criteria

Migration is complete when:

- `docker compose up -d` on mini1 using the new compose brings all three services up clean.
- Mainsail at `http://mini1/` shows Klipper connected, prints start normally.
- `git status` on the repo working tree on mini1 shows only expected live edits (not bulk drift).
- OrcaSlicer on Mac shows all custom printer/process/filament profiles and can slice a test file.
- A Mainsail edit (e.g., changing a macro) appears as a pending change in `git status` on mini1 within seconds.
- An OrcaSlicer edit (e.g., tweaking a filament PA value) appears as a pending change in `git status` on Mac within seconds.
- Claude Code opened in `~/git/3dprinter-settings/` on Mac has access to the migrated memory entries.

## Open questions

None at design time. Any additional integrations (Spoolman, Home Assistant bridges, OctoEverywhere, etc.) with their own tokens are assumed to either (a) not be present, or (b) be handled by extending the same `secrets.conf` pattern — to be discovered during step 3 of migration.
