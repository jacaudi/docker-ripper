# docker-ripper — project memory

Fork of `rix1337/docker-ripper` (this repo: `jacaudi/docker-ripper`). A single Docker
image that polls an optical drive, detects the disc type, rips it with the right
tool, fixes ownership/permissions, ejects, and optionally sends a Pushover
notification. A small web UI shows the tail of the rip log.

Everything below "Repository map" describes the **current bash/python implementation**
(the behaviour the Go port must match or deliberately change).

The pre-conversion state is frozen on the `archive` branch. Do not commit to it.

## Engineering principles (apply to every change)

- **KISS / YAGNI** — smallest thing that works; no speculative options, layers or flags.
- **DRY** — one implementation per behaviour (e.g. the DVD and BluRay handlers in
  `ripper.sh` are copy-paste twins; the Go port must have one video ripper).
- **SOLID** — small single-purpose types; depend on narrow interfaces, not concretions.
- **12-Factor** — config from env (flags may override); logs to stdout as a JSON event
  stream; port binding via env; fast startup and graceful shutdown (cancel context,
  stop child processes, clean up partial output).
- **Idiomatic / modern Go** — follow go.dev guidance (`cmd/<name>/main.go`, everything
  else in `internal/`), stdlib first (`log/slog`, `net/http` method+path patterns,
  `embed`, `context`, `errors.Is/As/Join`, `testing/synctest`, `t.Context()`),
  "accept interfaces, return structs", errors wrapped with `%w`, no package-level
  state, table-driven tests, golangci-lint (gofumpt, goimports, errorlint, gosec,
  revive, bodyclose, noctx) clean.
- **Seam** — one interface per capability (detect, rip, eject, notify, run a process,
  MakeMKV key source). A seam has one or more **backends**, each in its own package,
  each a drop-in implementation of that interface.
- **Patchbay** — the single place that picks a backend for each seam and plugs it in
  (`internal/patchbay`, called from the `serve` command). Swapping or adding a backend
  is "add a package + one case in the patchbay (or a config value)", never a
  refactor of the callers. Backends never construct other backends.

## Target stack (decided — see `docs/golang-conversion-plan.md` and `docs/plan/`)

- **Go 1.27**; one binary `ripper`; entrypoint `cmd/ripper/main.go` → `internal/cli`.
- **cobra** (`serve [--headless]`, `detect [--raw]`, `healthcheck`, `version`). **viper** is the only
  config loader and lives only in `internal/cli`. Config is a clean `RIPPER_*` set of 21 keys; the
  legacy variable names are not supported. Drives are **auto-discovered** (`RIPPER_DRIVES` is only a filter).
- **go-service-kit** v0.3.0: `lifecycle`, `obs`, `httpapi` (API/UI/docs `:9090`, admin `:9091`), `outbound`.
- **Seams** (5): runner, detect, rip, eject, notify. `internal/patchbay` selects the backends.
- **Engine**: one scan per tick → one watcher per drive → a FIFO **job queue** → a worker pool
  (`RIPPER_MAX_PARALLEL_JOBS`, 0 = no cap; at most one job per drive).
- **apprise-go** for notifications, and as the hook for automation after a rip.
- **Web UI**: React 19 + Ant Design 6, embedded. **API docs**: Scalar, embedded, at `/docs`.
- Observability: Slot 0 = Prometheus `/metrics` + JSON logs on stdout (fixed schema and message
  catalogue). Probes: `/livez`, `/startupz`, `/healthz` (dependency health) and `/readyz` (ready for work).
  OTLP traces only, as go-service-kit does today; `internal/telemetry` is the one place to add OTLP
  metrics/logs later. An in-memory log ring feeds the UI.
- **Image**: distroless `gcr.io/distroless/cc-debian13` (no shell), with MakeMKV built from source at
  a pinned version, ELF tools copied with their library closure, and `tini-static` as PID 1.
- **Audio CDs**: cyanrip (AccurateRip, MusicBrainz, cover art, ReplayGain), built from source;
  abcde is gone. MakeMKV and cyanrip share one static ffmpeg.
- Every rip is staged (`<kind>/.staging/`) and renamed into place. After any rip the engine waits
  for the disc to be removed.
- User scripts (`/config/ripper.sh`, per-disc hooks) are removed. The conversion is **not 1-to-1**;
  it is checked against a behaviour spec and Go tests, not against the legacy script.
- Tooling: `taskfile.yml` (Task), golangci-lint v2, govulncheck, OSV-Scanner, CodeQL, Trivy,
  Scorecard, release-please with Conventional Commits; images on `ghcr.io/jacaudi/docker-ripper`.
- Executable plan: `docs/plan/phases.md` (tasks), `contracts.md` (normative), `testing.md`, `tooling.md`.

## Repository map

| Path | Purpose |
|---|---|
| `root/` | Copied verbatim to `/` in the image (`COPY root/ /`). |
| `root/etc/my_init.d/ripper.sh` | Container init (phusion `my_init`). MakeMKV key/registration, abcde.conf, users/groups, perms, then `bash /config/ripper.sh &`. |
| `root/etc/my_init.d/web.sh` | Starts `python3 /web/web.py --port=9090 --prefix=$PREFIX --log=/config/Ripper.log --user=$USER --pass=$PASS &`. |
| `root/ripper/ripper.sh` | **The ripper.** Infinite poll loop: detect → rip → chown/chmod → eject → notify. |
| `root/ripper/abcde.conf` | Default abcde config (MP3 + FLAC, gnudb CDDB, cover-art `post_encode`). |
| `root/ripper/default.mmcp.xml` | MakeMKV output profile (copy tracks, LPCM → raw/WAV). |
| `root/ripper/settings.conf` | `app_Key = ""` — **dead file**, never copied anywhere. |
| `root/web/web.py` | Flask + waitress log viewer (docopt CLI). |
| `root/web/web/` | Static UI: `index.html` (petite-vue, wrapped in `{% raw %}`), bootstrap css/icons, favicon. |
| `root/etc/syslog-ng/syslog-ng.conf` | Overrides phusion's syslog-ng config (adds stdout destination). |
| `latest/Dockerfile` | "PPA" image: `phusion/baseimage:noble-1.0.0` + `ppa:heyarje/makemkv-beta`. amd64 + arm64. |
| `manual-build/Dockerfile` + `install/install.sh` | Builds MakeMKV from source (GPG-verified tarballs), adds OpenJDK 11 for BD-J. amd64 only. Remaps `nobody` to uid 99/gid 100 (unRAID). |
| `docker-compose.yml` | Reference deployment with every env var documented. |
| `.github/workflows/BuildImages.yml` | Builds/pushes `manual-latest`, `<makemkv-version>`, `latest`, `ppa-latest` (+ `-amd64`/`-arm64`) to Docker Hub `rix1337/docker-ripper`. |
| `.github/workflows/ManualBuildOnBetaRelease.yml` | Every 6h: triggers manual build when a new MakeMKV version appears on the forum. |
| `.github/workflows/UpdateOnBaseImageChange.yml` | Every 6h: triggers build when base image changes (checks `focal`, image is `noble` — stale). |
| `.github/workflows/{IssueModerator,LabelSponsors}.yml` | Upstream issue hygiene; need upstream secrets. |

There are no tests, no linters and no CI checks other than image builds.

## Runtime flow

### Container start (`/etc/my_init.d/ripper.sh`)

1. `mkdir -p /config`; copy `/ripper/ripper.sh` → `/config/ripper.sh` **only if absent**
   (so users keep stale scripts forever after image upgrades; they may edit it freely).
2. Fetch the free beta key by scraping `forum.makemkv.com/...t=1053` (regex `T-[\w\d@]{66}`).
   `KEY` env overrides it. The key is **echoed to the log** (secret leak).
3. Write `app_Key = "<key>"` to `$HOME/.MakeMKV/settings.conf` if it differs; `chmod -R 777`;
   run `makemkvcon reg $KEY` (deliberately unquoted); log registration status.
4. If `STORAGE_CD` is set, `sed` `OUTPUTDIR=` in `/ripper/abcde.conf`. Then if
   `/config/abcde.conf` exists it **overwrites** `/ripper/abcde.conf` (user file wins).
5. Create group `FILEGROUP` (gid `FILEGROUPID`, default 4321) and user `FILEUSER`
   (uid `FILEUSERID`, default 321) if missing. `chown -R`/`chmod -R` `/config`.
6. `bash /config/ripper.sh &` — unsupervised; if it exits nothing restarts it.

### Poll loop (`/config/ripper.sh`, a copy of `root/ripper/ripper.sh`)

Every 60 s (`sleep 1m`):

1. `rm -rf /tmp/*.tmp`.
2. **Detect**: `timeout 30s makemkvcon -r --cache=1 info disc:9999 | grep DRV:.*$DRIVE`,
   then match the line against `DRIVE_TYPE_PATTERNS` (see below). If the result is
   `empty`, double-check with `cdparanoia -d $DRIVE -Q` ("audio tracks" ⇒ `cd1`).
   No match ⇒ `BAD_RESPONSE++`; any match resets it to 0.
3. `empty` / `open` / `loading` ⇒ log and wait.
4. Anything else (including **no match**):
   - `BAD_RESPONSE >= BAD_THRESHOLD` ⇒ eject and `exit 1`.
   - `JUSTMAKEISO=true` ⇒ `handle_data_disc` (ddrescue ISO), eject.
   - else `process_disc_type` (BD/DVD/CD handler), then if `ALSOMAKEISO=true` also
     `handle_data_disc`, then eject.
5. `ejectdisc`: `eject -v $DRIVE`, fallback `sdparm --command=unlock` + `--command=eject`.
   With `EJECTENABLED=false` it busy-waits (5 s) until detection reports `open`/`empty`.
   Then Pushover via `curl` if `POVER_APP_TOKEN` and `POVER_USER_KEY` are set.

### makemkvcon `DRV:` line → disc type

Robot-mode line: `DRV:<index>,<state>,999,<flags>,"<drive name>","<disc label>","<device>"`.

| Key | Pattern | Meaning | Handler |
|---|---|---|---|
| `empty` | `DRV:N,0,999,0,"` | no disc | wait |
| `open` | `DRV:N,1,999,0,"` | tray open | wait |
| `loading` | `DRV:N,3,999,0,"` | loading | wait |
| `bd1` / `bd2` | `DRV:N,2,999,12,"` / `…,28,"` | BluRay / UHD | MakeMKV → MKV |
| `dvd` | `DRV:N,2,999,1,"` | DVD | MakeMKV → MKV |
| `cd1` | `DRV:N,2,999,0,"` | CD (audio *or data*) | abcde → MP3+FLAC |
| `cd2` | `","","<DRIVE>"` | inserted, empty label | abcde |

Label = text between the first `","` and the last `","`. Disc index = `cut -c5` of the line.

### Rip handlers

| Type | Default command | Output dir | Override hook (`/config/…`, must be `+x`) | Hook args |
|---|---|---|---|---|
| BluRay | `makemkvcon --profile=/config/default.mmcp.xml -r --decrypt --minlength=$MINIMUMLENGTH mkv disc:<idx> all <dir>` | `$STORAGE_BD/[ts_]<label>` | `BLURAYrip.sh` | `<idx> <dir> <logfile>` |
| DVD | same as BluRay | `$STORAGE_DVD/[ts_]<label>` | `DVDrip.sh` | `<idx> <dir> <logfile>` |
| Audio CD | `abcde -d $DRIVE -c /ripper/abcde.conf -N -x -l` | abcde `OUTPUTDIR` | `CDrip.sh` | `<drive> <STORAGE_CD> <logfile>` |
| Data / ISO | `ddrescue $DRIVE <dir>/<label>.iso` | `$STORAGE_DATA/[ts_]<label>` | `DATArip.sh` | `<drive> <isopath> <logfile>` |

`[ts_]` = `YYYYMMDD_HHMMSS_` when `TIMESTAMPPREFIX=true`. Tool stdout/stderr is appended
to `/config/Ripper.log`. After a video rip, `move_to_finished`: with
`SEPARATERAWFINISH=true` move `<dir>` → `<root>/finished/<dir>` and chown/chmod the whole
root; otherwise chown/chmod `<dir>`. CD/data chown/chmod the whole storage root.

### Web UI (`web.py`, port 9090)

- `GET {prefix}/` → `index.html`; `GET {prefix}/<path>` → static files; `GET /` → redirect to prefix (when set).
- `GET {prefix}/api/log/` → `{"log": [last 100 lines, newest first], "filesize": "1.2MiB", "large_file": size > 1e6}`.
- `DELETE {prefix}/api/log/` → truncate the log.
- HTTP Basic auth on every route when both `USER` and `PASS` are set (realm says "FeedCrawler", non-constant-time compare).
- The UI polls the log every 10 s.

## Configuration (env)

| Var | Default | Used by |
|---|---|---|
| `DRIVE` | `/dev/sr0` | ripper |
| `STORAGE_CD` / `STORAGE_DATA` / `STORAGE_DVD` / `STORAGE_BD` | `/out/Ripper/{CD,DATA,DVD,BluRay}` | ripper (+ init for CD) |
| `EJECTENABLED` | `true` | ripper |
| `JUSTMAKEISO` / `ALSOMAKEISO` | `false` | ripper |
| `SEPARATERAWFINISH` | `false` | ripper |
| `TIMESTAMPPREFIX` | `false` | ripper |
| `MINIMUMLENGTH` | `600` (s) | ripper (MakeMKV) |
| `BAD_THRESHOLD` | `5` | ripper |
| `DEBUG` / `DEBUGTOWEB` | `false` | ripper (stdout / log file) |
| `FILEUSER` / `FILEUSERID` | `nobody` / `321` | init + ripper |
| `FILEGROUP` / `FILEGROUPID` | `users` / `4321` | init + ripper |
| `FILEMODE` | `g+rw` (symbolic chmod) | init + ripper |
| `KEY` | scraped beta key | init |
| `PREFIX` / `USER` / `PASS` | unset | web (README calls them `OPTIONAL_WEB_UI_*`; compose uses the short names) |
| `POVER_APP_TOKEN` / `POVER_USER_KEY` | unset | ripper |

Fixed paths: `/config` (state, log, overrides), `/out` (rips), `/ripper` (defaults),
`/web`, `$HOME=/root` (`~/.MakeMKV`).

## External tools

| Tool | Why | Replaceable in Go? |
|---|---|---|
| `makemkvcon` | detection + DVD/BD rip + registration | **No** (proprietary). Detection alone could go native via ioctl/SG_IO. |
| `abcde` (+ `cdparanoia`, `cd-discid`, `lame`, `flac`, `eyeD3`, `metaflac`, `glyrc`) | audio CD pipeline; `abcde.conf` is a user-facing customisation surface | Keep. |
| `cdparanoia -Q` | audio-CD fallback detection | Yes — `CDROM_DISC_STATUS` ioctl. |
| `ddrescue` | ISO with bad-sector recovery | Keep (recovery logic is the point). |
| `eject`, `sdparm` | eject / unlock tray | Yes — `CDROMEJECT`, `CDROM_LOCKDOOR` ioctls. |
| `curl` | Pushover, key scrape | Yes — apprise-go (notifications), kit `outbound` (key scrape). |
| `grep/sed/cut/date/timeout` | text munging | Yes — stdlib. |
| `useradd/groupadd` | named owners | Yes — numeric `os.Chown`, `os/user` lookup. |
| python3, flask, waitress, docopt | web UI | Yes — kit `httpapi` (huma) + `embed`. |
| phusion `my_init`, syslog-ng | init / logging | Yes — kit `lifecycle` + `obs`, single Go process under `tini`. |
| `ccextractor`, OpenJDK | used *by* MakeMKV | Keep in image. |

## Known bugs and quirks (do not silently "preserve" these in a port)

1. `declare -A` iteration order is undefined, and patterns overlap: `cd2` matches *any*
   line with an empty label — empty drive, open tray, BD, DVD ⇒ nondeterministic
   detection (an empty drive can be "ripped" as a CD).
2. Data discs have no type: data CDs match `cd1` and go to abcde; data DVDs match `dvd`.
   `handle_data_disc` only runs via `JUSTMAKEISO`/`ALSOMAKEISO`.
3. An unrecognised `makemkvcon` reply (below threshold) falls into the "rip" branch:
   prints "not recognized", may run `ALSOMAKEISO` on nothing, then **ejects**.
4. `cut -c5` breaks for drive index ≥ 10.
5. `--profile=/config/default.mmcp.xml` — init never copies that file to `/config`.
6. `abcde.conf` has `EJECTCD=y`, so with `ALSOMAKEISO=true` the CD is already ejected
   before `ddrescue` runs; and `ejectdisc` then ejects again.
7. Empty label ⇒ rip lands directly in the storage root; labels are not sanitised.
8. `get_disc_directory` ignores its third argument; `cleanup_tmp_files` does `cd -` and `exit`s on failure.
9. Pushover sends a fixed message even after "too many bad responses"; set-but-empty
   tokens count as set.
10. `/config/ripper.sh` is never updated after first run; ripper and web are unsupervised
    background jobs under `my_init`.
11. Init echoes the MakeMKV key; `chmod -R 777 ~/.MakeMKV`.
12. `USER` env var collides with the conventional shell variable.
13. `web.py` `tail()` seeks from end in text mode, which raises and falls back to reading
    the whole file every request.
14. `install.sh` tests `$version` instead of `$MAKEMKV_VERSION`, so it always falls back
    to forum scraping.
15. `UpdateOnBaseImageChange.yml` watches `focal`; images use `noble`. Workflows push to
    upstream's Docker Hub namespace with upstream secrets.

## Branches

- `main` — upstream-tracking default.
- `archive` — frozen snapshot of `main` before the Go conversion. Never commit to it.
- `chore/claude-md` — this file.
- `claude/golang-conversion-plan-*` — the Go conversion plan (`docs/golang-conversion-plan.md`).

Images are published to `ghcr.io/jacaudi/docker-ripper` (decided; workflows still
point at upstream Docker Hub until phase 5).

## Working in this repo

- Shell: `bash -n` and `shellcheck` any script you touch.
- Images: `docker build -f latest/Dockerfile -t ripper:dev .` (fast) or
  `-f manual-build/Dockerfile` (compiles MakeMKV, slow).
- Real ripping needs a passed-through drive (`--device=/dev/sr0 --device=/dev/sg0`,
  sometimes `--privileged`); never assume one exists in CI.
