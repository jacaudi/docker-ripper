# Go conversion plan

Status: **agreed, v4 (executable, simplified)** · Baseline: `archive` branch (= `main` @ `21c5796`) ·
Background: [`.claude/CLAUDE.md`](../.claude/CLAUDE.md)

## How to use this plan

The plan is written so that an executor (a human or a smaller model) can carry it out **without
making design decisions**.

| File | Contains | Use it to |
|---|---|---|
| this file | goal, decisions, glossary, rules, deployment, migration notes | understand *why* and what's forbidden |
| [`plan/contracts.md`](plan/contracts.md) | config table, exact Go types and interfaces, behaviour tables, HTTP routes, CSP, primitives | copy names, signatures and values |
| [`plan/testing.md`](plan/testing.md) | test layers, fixtures, fakebin protocol, end-to-end smoke scenarios | write tests |
| [`plan/tooling.md`](plan/tooling.md) | taskfile, lint/static analysis, security scanning, Conventional Commits, release-please, Renovate, workflows | set up quality gates and releases |
| [`plan/phases.md`](plan/phases.md) | numbered tasks with files, steps and "done when" commands | do the work, in order |

If something you need is not specified, **stop and ask**. Do not invent behaviour.

## 1. Goal

Replace the bash ripper, the init script and the Python web UI with one Go binary, `ripper`.
It runs either as a **headless API service** (`ripper serve --headless`) or as **API + web UI**
(`ripper serve`), and rips discs reliably without anyone watching, on **one or more drives at once**
(e.g. a BluRay on `sr0` and an audio CD on `sr1` concurrently).

The conversion is **not 1-to-1**. What carries over unchanged:
- the tool command lines;
- the MakeMKV `DRV:` truth table and the cdparanoia audio-CD fallback;
- the output folder names;
- the eject → sdparm fallback.

Configuration, logging, the API, the UI and the error handling are redesigned. This fork
**diverges from upstream permanently**.

Out of scope: new output formats (audio is FLAC and/or MP3; video is MakeMKV's MKV; data is ISO).

## 2. Decisions log

| # | Topic | Decision |
|---|---|---|
| 1 | Seam / Patchbay | One interface per capability (5 seams: runner, detect, rip, eject, notify). Backends are separate packages, single shared instances. `internal/patchbay` is the only place that selects them. |
| 2 | Entrypoint | `cmd/ripper/main.go` → `internal/cli` (cobra). Follows [go.dev module layout](https://go.dev/doc/modules/layout). |
| 3 | Config | **viper only**, confined to `internal/cli`. A clean `RIPPER_*` set of 22 keys (C§1); no legacy names. |
| 4 | Service framework | go-service-kit v0.3.0: `lifecycle`, `obs`, `httpapi`, `outbound`. |
| 5 | Modes | `ripper serve` = engine + API + docs, plus the web UI unless `--headless`. |
| 6 | Ports | API, UI and docs on `:9090`; admin (`/healthz`, `/readyz`, `/metrics`) on `:9091`. |
| 7 | Logging | JSON to stdout (the durable log), plus an in-memory ring of the last 2000 records for `GET /api/v1/log`. No log file. |
| 8 | Shutdown mid-rip | Cancel the tool's process group, delete the staging dir, no eject, notify "stopped". |
| 9 | Backend config | None. One production backend per seam; swapping one is a patchbay change. |
| 10 | Notifications | apprise-go v0.3.3; also the hook for automation after a rip (`json://`, `form://`). |
| 11 | Fixtures | Plain `.txt` files plus an expectation table. `ripper detect --raw` hardware captures replace the synthetic ones. |
| 12 | User scripts | Removed (`/config/ripper.sh` and the per-disc hooks). |
| 13 | Registry | `ghcr.io/jacaudi/docker-ripper`. |
| 14 | Web settings | `RIPPER_WEB_PATH_PREFIX`, `RIPPER_WEB_USERNAME`, `RIPPER_WEB_PASSWORD`. |
| 15 | Upstream | Diverge permanently; drop upstream-only workflows. |
| 16 | Deployment | Target-agnostic binary. Docker/compose is documented; there is also a k8s example with a device plugin. |
| 17 | API docs | Scalar 1.73.1, vendored and embedded, at `/docs` (always on). OpenAPI at `/openapi.json`. |
| 18 | Web UI | React 19 + Ant Design 6 (Vite 8, TypeScript 5.9): a status card and a log panel, embedded. Typed client generated from huma's OpenAPI. |
| 19 | Go version | Go 1.27 (`go 1.27.0`, `toolchain go1.27.2`). |
| 20 | Task runner | Task v3.54, lowercase `taskfile.yml`. No Makefile. CI calls the same tasks. |
| 21 | Quality gates | golangci-lint v2.14, govulncheck, OSV-Scanner, CodeQL, Trivy, OpenSSF Scorecard; SHA-pinned actions; signed images with SBOM + provenance (L§2–3). |
| 22 | Releases | release-please (manifest, release type `go`, Conventional Commits, GitHub App token). Images publish from the release workflow. |
| 23 | Verification | A behaviour spec plus Go tests (unit, fakebin integration, end-to-end smoke). **No legacy parity harness.** Hardware acceptance at cutover. |
| 24 | Bad drive replies | After 5 in a row a drive becomes `unusable` and one Failure notification is sent; `/readyz` fails only when **no** usable drive is left. No eject, no exit; polling continues and the drive recovers by itself. |
| 25 | Re-rip protection | After any rip attempt the engine waits for the disc to be removed. It never re-rips the same disc. |
| 26 | Staging | Every rip goes into `<kind>/.staging/…` and is renamed into place on success; staging is cleaned at start. |
| 27 | Native ioctl backends | Future note only (phases.md "Future"). |
| 28 | Drives and jobs | **No drive configuration needed.** One MakeMKV scan per tick discovers every drive; each drive gets a watcher (state machine). A newly inserted disc becomes a **job** on a FIFO queue; a worker pool runs the jobs (at most one per drive; `RIPPER_MAX_PARALLEL_JOBS` caps it, 0 = no cap). Drives appear and disappear at runtime. `RIPPER_DRIVES` is only an optional filter. API: `/api/v1/status` (drives) and `/api/v1/jobs`. |
| 29 | Observability | Slot 0 = Prometheus pull (`:9091/metrics`) + JSON logs on stdout (schema + catalogue). Probes: `/livez` (process), `/startupz` (startup done), `/healthz` (**dependency health**: tools, dirs, MakeMKV registration, scanner, loop), `/readyz` (ready for work: started, ≥1 usable drive, not draining), all with one JSON schema (C§6.1). OTel: exactly go-service-kit's current behaviour (OTLP traces), behind `internal/telemetry` so OTLP metrics and logs can be added later in one place (C§6.5). |
| 30 | Runtime image | **Distroless** `gcr.io/distroless/cc-debian13` (scratch is impossible: `makemkvcon` is a proprietary glibc binary). One image (amd64 + arm64); no shell, no interpreter, no `RUN` in the final stage. MakeMKV is built from source in a matching `debian:trixie` stage at a pinned version, with a static audio-only ffmpeg. Tools are copied with their library closure. PID 1 is `tini-static`, copied in. `HOME=/config`. |
| 31 | Audio CDs | **abcde is replaced by cyanrip** (C, LGPL-2.1; better than abcde): AccurateRip v1/v2 + EAC CRC verification, offset correction, pregaps, MusicBrainz tags + cover art, ReplayGain, parallel multi-format encoding (`RIPPER_AUDIO_FORMATS`: flac, mp3, opus, aac, alac). Built from source against the same static ffmpeg as MakeMKV. There is no Go port of abcde ([research](#references)). |

## 3. Glossary

- **Seam**: the interface for one capability: `runner.Runner`, `detect.Detector`, `rip.Ripper`,
  `eject.Ejector`, `notify.Notifier` (C§2.2).
- **Backend**: one package implementing a seam, e.g. `rip/cyanrip` (C§2.3).
- **Patchbay**: `internal/patchbay`. `Backends(...)` picks the backends; `Spec(...)` assembles the
  HTTP servers, readiness and the `lifecycle.Spec`.
- **Engine**: `internal/engine`. It scans every tick, keeps one **watcher** per discovered drive,
  queues **jobs** and runs them on a worker pool (C§3.3). It is a single `lifecycle.Worker`.
- **Watcher**: the per-drive state machine (idle → queued → ripping → ejecting → awaiting_removal; unusable).
- **Job**: one rip of one inserted disc, bound to its drive: queued → running → succeeded/failed/cancelled/skipped.
- **Scan**: one iteration of the engine loop: discover drives, update watchers, dispatch jobs (C§3.3).
- **Rip plan**: the ordered rip steps for an (ISO mode, kind) pair (C§3.4).
- **Drive ID**: the device basename, e.g. `sr0` (C§1.2).
- **Kind dir**: `BluRay`, `DVD`, `CD` or `DATA` under `RIPPER_OUTPUT_DIR`.
- **Staging dir**: `<kind dir>/.staging/<ts>-<drive>/<name>`, where a rip writes before it is finalized (C§3.5).
- **Slot 0**: the observability defaults that need no config: Prometheus pull metrics and JSON logs on stdout.
- **Awaiting removal**: the engine state after a rip attempt, until the drive reports empty or open.
- **fakebin**: the fake external-tools binary used in tests (T§3).
- **C§n / T§n / L§n**: section n of `contracts.md` / `testing.md` / `tooling.md`.

## 4. Rules

**Do**
- Use Go 1.27 primitives where they fit:
  - `os.Root` for every output-dir file operation;
  - `errors.AsType`, `strings.Lines`, `sync.WaitGroup.Go`, `http.CrossOriginProtection`;
  - `testing/synctest` with `synctest.Sleep`;
  - `t.Context()`, and `slog` `…Context` calls so logs carry `trace_id`;
  - `context.WithoutCancel` for cleanup;
  - `debug.ReadBuildInfo` for the version.
- Run `task lint test vuln` before every commit (`task check` once `web/` exists).
- Use Conventional Commit messages and PR titles (L§4).
- Keep seams at 1–2 methods, with a compile-time assertion in every backend.
- Wrap errors with `%w` and context, e.g. `fmt.Errorf("audio rip: %w", err)`.

**Do not**
- Do not import viper or cobra outside `internal/cli`.
- Do not use the package-level viper instance, and do not call `v.AutomaticEnv()`.
- Do not keep package-level mutable state.
- Do not use `http.StripPrefix`, or anything else that clones the request, as listener-wide middleware
  (per-route `StripPrefix` after the mux has matched is fine).
- Do not use `cmd.StdoutPipe` with `cmd.Run`/`Wait`.
- Do not log argv, the MakeMKV key, apprise URLs or the web password. Do not use a `msg` that isn't in the C§6.3 catalogue.
- Do not use disc labels, paths or errors as metric labels.
- Do not use `-ldflags -X` for the version. Do not use `encoding/json/v2`.
- Do not use `synctest` around real processes or listeners.
- Do not execute user-supplied scripts. Do not call `useradd`/`groupadd`.
- Do not load anything from a CDN at runtime.
- Do not add config keys, endpoints, backends or seams that aren't in `contracts.md`.
- Do not add a `Makefile`, or run a check in CI that isn't a `task`.

## 5. Principles applied

| Principle | Concretely |
|---|---|
| KISS | One binary, one engine loop, one output root, one log stream, a UI with no router. |
| YAGNI | 22 config keys; no backend switches; no parity harness; no log file; native ioctl deferred. |
| DRY | One `rip.Ripper` interface and one staging/finalize path for every kind; TS types generated from the Go types; CI calls the tasks. |
| SOLID | 5 small seams; substitutable backends; new backend = new package + one patchbay line; the engine depends only on seams; patchbay split into `Backends` and `Spec`. |
| 12-Factor | Env config (+ one flag); logs to stdout; port binding via env; kit `lifecycle` for disposability; admin one-offs (`detect`, `healthcheck`). |

## 6. Deployment and runtime

- **One instance, pinned to the host that owns the drive.**
  - Docker: `--device /dev/srN --device /dev/sgN` for **each** drive.
  - Kubernetes: a device-plugin DaemonSet ([generic-device-plugin](https://github.com/squat/generic-device-plugin/issues/62),
    [Talos guide](https://docs.siderolabs.com/kubernetes-guides/advanced-guides/device-plugins.md)),
    `replicas: 1`, `strategy: Recreate`, and a node selector.
- **Image**: distroless, with no shell. Debug with `docker debug` or `kubectl debug` (an ephemeral
  container), not `docker exec … sh`.
- **PID 1 is `tini-static`**, copied into the image: `ENTRYPOINT ["/usr/bin/tini-static","--","/usr/local/bin/ripper"]`,
  `CMD ["serve"]`. Compose doesn't need `init: true`, and k8s (which has no init) is covered too.
- **User**: root by default, because device nodes are `root:cdrom 660` on hosts and chowning output
  needs `CAP_CHOWN`. Rootless recipe: `user: "1000:1000"` + `group_add: ["<host cdrom gid>"]`,
  with `RIPPER_UID`/`RIPPER_GID` unset.
- **Shutdown budget**:
  - `PropagationDelay: NoPropagationDelay`;
  - `DrainTimeout: 5s`;
  - `FlushTimeout`: the default, 5s;
  - engine `StopTimeout: 15s`. The kit cancels the context immediately; 15 s is the cleanup budget per
    running job: process-group kill ≤ 5 s + staging cleanup ≤ 5 s + "stopped" notification ≤ 3 s.
  
  `Spec.GracePeriod()` = 5 + 5 + 15 + 5 margin = **30 s**; the container's grace must exceed it, so use
  compose `stop_grace_period: 35s` and k8s `terminationGracePeriodSeconds: 35`.
- **Probes** (all on 9091, schemas in C§6.1): Docker `HEALTHCHECK CMD ["ripper","healthcheck"]` checks `/healthz` (dependencies).
  - k8s `startupProbe`: `/startupz`; `livenessProbe`: `/livez` (process only, so a broken dependency
    never causes a restart loop);
  - k8s `readinessProbe`: `/readyz` (started, ≥1 usable drive, not draining).
- **Metrics**: Prometheus scrapes `:9091/metrics` (ServiceMonitor targets the admin port).
- **OTLP**: set `OTEL_EXPORTER_OTLP_ENDPOINT` (+ the standard `OTEL_*` variables) to export traces
  (go-service-kit's current behaviour). OTLP metrics and logs are a later, one-package change (C§6.5).
- Go ≥ 1.25 sets `GOMAXPROCS` from the container's CPU limit automatically.

## 7. Dependencies

### 7.1 Go modules

| Module | Version | Notes |
|---|---|---|
| cobra | v1.10.2 | |
| viper | v1.21.0 | |
| go-service-kit | v0.3.0 | |
| apprise-go | v0.3.3 (exact) | pre-1.0 |
| huma | `github.com/danielgtaylor/huma/v2` v2.39.0 | the kit's version; used directly in `internal/api` |
| Prometheus client | `github.com/prometheus/client_golang` v1.24.1 | the kit's version; `*prometheus.Registry` in `internal/health` |
| OTel API | `go.opentelemetry.io/otel`, `/trace`, `/metric`, `/attribute`, `/codes` v1.45.0 | the kit's version; API only outside `internal/telemetry` |
| OTel SDK (tests and `internal/telemetry` only) | `go.opentelemetry.io/otel/sdk`, `/sdk/metric` v1.45.0 | in-memory exporter/reader in tests |

Go modules other than these need an entry here first.

Two trade-offs are accepted:
- apprise-go's `Send` has no context and uses its own HTTP client, so it bypasses kit `outbound`.
  It is wrapped with a timeout (C§5.2).
- Vendoring Scalar means ripper owns a CSP relaxation (`style-src 'unsafe-inline'`, on `/docs` only)
  and the Renovate/sha256 upkeep.

### 7.2 External tools

| Tool | Decision | Backend |
|---|---|---|
| `makemkvcon` | keep (proprietary) | `detect/makemkv`, `rip/makemkv`, `makemkvkey.Register` |
| `cyanrip` | **new**, built from source (ELF) | `rip/cyanrip` (rip + verify + tag + encode) |
| `cdparanoia` | keep (ELF) | `detect/makemkv` (`-Q`, audio vs data) |
| `ddrescue` | keep (with a map file) | `rip/ddrescue` |
| `eject`, `sdparm` | keep (ELF) | `eject/execeject` |
| `tini-static` | keep (static) | PID 1 |
| `abcde`, eyeD3 (Python), glyrc, cd-discid, flac/metaflac, lame, wget | **remove** | `rip/cyanrip` |
| `ccextractor` (closed captions), OpenJDK (BD-J menus) | **remove** | — (MakeMKV rips without them) |
| curl, grep/sed/cut/date/timeout, useradd, bash, python/flask, phusion, syslog-ng | **remove** | apprise-go, kit, stdlib |

## 8. Migration notes (for the README)

**Configuration:** all variables are renamed to `RIPPER_*` (C§1.2).

| Old | New |
|---|---|
| `DRIVE` | none needed: drives are discovered. Optional filter `RIPPER_DRIVES` |
| `STORAGE_CD/DATA/DVD/BD` | `RIPPER_OUTPUT_DIR` (fixed `CD/ DATA/ DVD/ BluRay/` subdirs; `/out/Ripper` keeps the old layout) |
| `EJECTENABLED` | `RIPPER_EJECT` |
| `JUSTMAKEISO` / `ALSOMAKEISO` | `RIPPER_ISO_MODE=only` / `also` |
| `MINIMUMLENGTH` | `RIPPER_MIN_TITLE_LENGTH` |
| `FILEUSER`/`FILEUSERID`, `FILEGROUP`/`FILEGROUPID` | `RIPPER_UID`, `RIPPER_GID` (numeric; unset = no chown) |
| `FILEMODE` (symbolic) | `RIPPER_UMASK` (octal, default `002`) |
| `KEY` | `RIPPER_MAKEMKV_KEY` |
| `POVER_APP_TOKEN` + `POVER_USER_KEY` | `RIPPER_APPRISE_URLS=pover://USER_KEY@APP_TOKEN` |
| `PREFIX` / `USER` / `PASS` | `RIPPER_WEB_PATH_PREFIX` / `RIPPER_WEB_USERNAME` / `RIPPER_WEB_PASSWORD` |
| `DEBUG`, `DEBUGTOWEB` | `RIPPER_LOG_LEVEL` |
| `TIMESTAMPPREFIX`, `SEPARATERAWFINISH`, `BAD_THRESHOLD` | removed |
| `/config/abcde.conf` | removed; use `RIPPER_AUDIO_FORMATS` and `RIPPER_AUDIO_DRIVE_OFFSETS` |

**Removed features**
- `/config/ripper.sh` and the hook scripts (`BLURAYrip.sh`, `DVDrip.sh`, `CDrip.sh`, `DATArip.sh`).
  Use an apprise webhook target for automation after a rip.
- The `finished/` folder (staging makes every final folder complete by construction).
- The timestamp prefix.
- `/config/Ripper.log`, and clearing the log from the UI.
- Exit on repeated bad drive replies.
- User and group names (numeric IDs only).

**Behaviour changes**
1. Detection uses ordered rules on parsed fields. An empty drive can no longer be "ripped" as a CD,
   and drive indexes ≥ 10 work.
2. Data CDs are imaged with ddrescue instead of being sent to the audio ripper. DVD/BD data discs are still
   treated as video.
3. An unrecognised drive reply never triggers a rip or an eject.
4. After any rip attempt the engine waits for the disc to be removed: no re-rip loops.
5. Rips are staged and renamed into place. Partial output is removed on failure, on shutdown, and
   at the next start after a crash.
6. `ddrescue` keeps its `.map` file next to the finished `.iso` (a record of any unreadable areas).
   An interrupted ISO rip is discarded with its staging dir and restarts from scratch.
7. The ripper is the only thing that ejects (abcde used to eject CDs itself). `RIPPER_EJECT=false` is
   now honoured for CDs.
8. **Audio CDs are ripped with cyanrip instead of abcde**: AccurateRip-verified, MusicBrainz tags and
   cover art, ReplayGain, with formats from `RIPPER_AUDIO_FORMATS` (`flac`, `mp3`, `opus`, `aac`, `alac`).
   It uses cyanrip's own folder and file naming; a disc missing from MusicBrainz gets placeholder names.
   `abcde.conf` is no longer read.
9. `ISO_MODE` skips audio CDs, which can't be imaged.
10. `default.mmcp.xml` is now actually used: from `/config` if present, otherwise the built-in one.
11. The MakeMKV key is never logged; `~/.MakeMKV` has mode 0700.
12. JSON logs on stdout with a fixed schema. The UI shows the last 2000 records since the process started.
    Prometheus metrics on `:9091/metrics`; optional OTLP export via the standard `OTEL_*` variables.
13. The API is `/api/v1/status`, `/api/v1/jobs` and `/api/v1/log`, with OpenAPI at `/openapi.json` and docs at `/docs`.
    Admin endpoints are on port 9091.
14. The web UI is rewritten in React + Ant Design. Basic auth is constant-time, with CSRF protection.
15. The image is distroless (`cc-debian13`), with no shell: one multi-arch image at
    `ghcr.io/jacaudi/docker-ripper` replaces the `latest`/`manual-latest` pair. MakeMKV is pinned
    and updated through releases. Closed-caption extraction (ccextractor) and BD-J menu support
    (Java) are not included.
16. Drives are discovered automatically and rip at the same time through a job queue
    (`RIPPER_MAX_PARALLEL_JOBS` caps it). Health probes: `/livez`, `/startupz`, `/healthz`, `/readyz`.

## 9. Repository admin (manual)

- Protect `archive`: Settings → Rules → Rulesets → New branch ruleset → target `archive`;
  enable *Restrict deletions*, *Block force pushes* and *Restrict updates*.
- Create a GitHub App for release-please (contents + pull-requests: write) and install it.
  Add the secrets `RELEASE_APP_ID` and `RELEASE_APP_PRIVATE_KEY` (L§5).
- Settings → General: allow squash merging only, with the PR title as the default commit message.
- Settings → Actions → General: allow GitHub Actions to create and approve pull requests.
- Branch protection on `main`: require the `ci` jobs and CodeQL; require a linear history.

## 10. Remaining risks

- **apprise-go** is pre-1.0. It sits behind the `Notifier` seam; re-evaluate in phase 2.
- **OTLP logs/metrics** are not exported yet (traces only, as in go-service-kit); adding them is a
  one-package change (C§6.5).
- **cyanrip**: the retry rule is verified in cyanrip's source (a MusicBrainz miss or an ambiguous match exits 1 before ripping; ripper passes `-R 1` to pick the first release);
  P2.3b confirms it on hardware. `-s` (drive offset) is mandatory; without your drive's real offset
  (`RIPPER_AUDIO_DRIVE_OFFSETS`), AccurateRip can report mismatches.
- **Unverified in this environment:** that Debian's `tini` ships `/usr/bin/tini-static` (fallback in P4.1);
  that `makemkv-bin` has an arm64 binary (P4.1 stops and asks); that `makemkv-oss` builds against
  FFmpeg 8.1; the AntD 6 `ConfigProvider csp` prop (P3.5's test catches it); the action major versions.
- **Scanning during a rip**: `makemkvcon info` runs every tick while other drives rip; checked at P4.5.
- **The ffmpeg component list** was derived from cyanrip's source; it is checked with `cyanrip -o help` (P4.1)
  and real rips including AAC, ReplayGain and cover art (P4.5).
- **Concurrent MakeMKV instances** (multi-drive) are assumed to work; this is checked at P4.5 with two drives if available.
- **`makemkvcon reg`** is assumed to write `app_Key` itself; this is checked at P4.5.
- **DVD/BD data discs** are treated as video until a hardware capture shows a distinguishing signal.
- **The beta key** is fetched once per start; auto-released MakeMKV bumps and the restart policy cover rotation.
- **MakeMKV source build in CI** depends on makemkv.com and a GPG keyserver. If it fails, the previous
  image stays published. The audio-only ffmpeg codec set is checked at P4.5 with DTS-HD/TrueHD and LPCM discs.
- **MusicBrainz coverage** differs from gnudb. Misses fall back to cyanrip's placeholder names.
- **Scanner visibility**: copied libraries are visible to Trivy only through the `status.d` fragments
  that `collect-rootfs.sh` writes.

## References

- go-service-kit: https://github.com/leftathome/go-service-kit
- cyanrip: https://github.com/cyanreg/cyanrip · audio-ripper research: no Go port of abcde exists; Go
  libraries considered: go.uploadedlobster.com/musicbrainzws2 (MIT), go.uploadedlobster.com/discid
  (cgo/LGPL), github.com/b0bbywan/go-disc-cuer (cgo)
- apprise-go: https://pkg.go.dev/github.com/unraid/apprise-go · https://unraid.net/blog/apprise-go
- Scalar configuration: https://github.com/scalar/scalar/blob/main/documentation/configuration.md
- Task: https://taskfile.dev · golangci-lint: https://golangci-lint.run · govulncheck: https://go.dev/doc/security/vuln/
- release-please action: https://github.com/googleapis/release-please-action · Conventional Commits: https://www.conventionalcommits.org
- OSV-Scanner: https://google.github.io/osv-scanner/ · OpenSSF Scorecard: https://github.com/ossf/scorecard · Trivy: https://trivy.dev
- Go module layout: https://go.dev/doc/modules/layout
- Device plugins: https://github.com/squat/generic-device-plugin/issues/62 ·
  https://docs.siderolabs.com/kubernetes-guides/advanced-guides/device-plugins.md
- Docker signals / grace period: https://oneuptime.com/blog/post/2026-01-16-docker-graceful-shutdown-signals/view ·
  https://www.netdata.cloud/guides/docker/docker-exit-code-143/
