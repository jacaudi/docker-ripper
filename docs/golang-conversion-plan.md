# Go conversion plan

Status: **agreed direction, v2** · Baseline: `archive` branch (= `main` @ `21c5796`) ·
Background: [`.claude/CLAUDE.md`](../.claude/CLAUDE.md)

## 1. Goal

Replace the bash ripper (`ripper.sh`), the init script and the Python web UI with one
Go binary, `ripper`, that runs either as a **headless API service** or as **API + web UI**.

- External tools that cannot reasonably be replaced (MakeMKV, abcde, ddrescue) are called
  through a seam with swappable backends, and tested against fake tool binaries.
- Everything else (curl, grep/sed/cut, useradd, python/flask, phusion init, syslog-ng)
  becomes Go or a library.
- The known bugs (CLAUDE.md, "Known bugs and quirks") are fixed deliberately, and each
  fix is recorded as a deviation (§8).

This fork **diverges from upstream permanently**. Breaking changes are allowed when
they're documented.

Out of scope: multi-drive support, new output formats, re-implementing abcde's audio pipeline.

## 2. Decisions log

| # | Topic | Decision |
|---|---|---|
| 1 | Seam / Patchbay | One interface per capability; one or more backends per seam, each in its own package. `internal/patchbay` is the only place that selects backends. Swapping one = add a package plus one case (or a config value). |
| 2 | Entrypoint | `cmd/ripper/main.go` → `internal/cli` (cobra). Follows [go.dev module layout](https://go.dev/doc/modules/layout). |
| 3 | Config | **viper only** (flags > env > defaults), imported only by `internal/cli`. Unmarshalled into a plain `config.Config` with `Validate()` that joins every error. go-service-kit `config` is not used. |
| 4 | Service framework | go-service-kit: `lifecycle`, `obs`, `httpapi`, `outbound`. Not `config`, `storekit`, `mcp`. |
| 5 | Modes | `ripper serve` runs the engine and the API; the web UI is mounted unless `--headless` / `HEADLESS=true`. |
| 6 | Ports | API (+ UI) on `API_ADDR` default `:9090` (today's port). Admin (`/healthz`, `/readyz`, `/metrics`, opt-in pprof) on `ADMIN_ADDR` default `:9091`. |
| 7 | Logging | One JSON stream (kit `obs`) to stdout, teed to `LOG_FILE` (default `/config/Ripper.log`). Tool output is logged line by line as records with a `tool` attribute. The UI renders JSON records and shows non-JSON lines raw. |
| 8 | Shutdown mid-rip | Cancel the tool (SIGTERM to its process group, then kill after `WaitDelay`), delete the partial output dir, don't eject. On restart the disc is still in the drive and is ripped again. |
| 9 | Backend config | Config keys only for seams with ≥ 2 production backends (`DETECTOR_BACKEND`, `EJECT_BACKEND`). Notifications are on when `APPRISE_URLS` is set. Everything else is fixed in the patchbay. |
| 10 | Notifications | [apprise-go](https://pkg.go.dev/github.com/unraid/apprise-go) (`github.com/unraid/apprise-go`, BSD-2), configured with `APPRISE_URLS`. Pushover is `pover://USER_KEY@APP_TOKEN`. |
| 11 | Fixtures | All three sources: synthetic (marked), mined from public reports (marked with source), and captured on your hardware (authoritative). |
| 12 | `/config/ripper.sh` | **Hard break.** No longer executed; documented in migration notes only. Per-disc hook scripts stay as the `script` rip backend. |
| 13 | Registry | `ghcr.io/jacaudi/docker-ripper` using `GITHUB_TOKEN`. |
| 14 | Web env names | **New names only**: `WEB_PATH_PREFIX`, `WEB_USERNAME`, `WEB_PASSWORD`. `PREFIX`/`USER`/`PASS` are ignored. |
| 15 | Upstream | Diverge permanently. Drop upstream-only workflows (sponsor labelling, issue auto-close). |
| 16 | Deployment target | Target-agnostic binary. Docker/compose is the documented target (see §4); a Kubernetes example uses a device plugin, `replicas: 1`, `strategy: Recreate`. |

## 3. Principles applied

| Principle | Concretely |
|---|---|
| KISS | One binary, one process, one engine loop. No plugin system, no config files beyond what viper gives for free. |
| YAGNI | No config key for a seam with one backend. No API endpoint without a consumer (§6.3). No native backend until hardware tests exist for it. |
| DRY | One `rip.Ripper` interface for every disc kind; the patchbay maps kinds to backends. One process runner. One output/permissions routine. One log stream. |
| SOLID | Seams are 1–2 method interfaces (ISP). Backends are substitutable (LSP). Adding a backend needs no change to the engine (OCP). The engine depends on seam interfaces only (DIP). |
| 12-Factor | Config from env (+flags); JSON logs to stdout; port binding via `API_ADDR`/`ADMIN_ADDR`; disposability via kit `lifecycle`; same image in dev and prod. One-off admin processes are cobra subcommands (`detect`, `healthcheck`). |
| Modern / idiomatic Go | Go 1.26 (the kit requires it). `log/slog`, `context` everywhere, `errors.Join`, `exec.Cmd.Cancel` + `WaitDelay`, `embed`, `testing/synctest`, `t.Context()`, table-driven tests. Lint config and Makefile gates copied from go-service-kit. |

## 4. Deployment and runtime

Industry practice for a drive-bound worker is **one instance, pinned to the host that
owns the drive**.
- On Docker this means `--device /dev/srN --device /dev/sgN`.
- On Kubernetes the recommended pattern is a device-plugin DaemonSet rather than a
  privileged workload pod ([generic-device-plugin](https://github.com/squat/generic-device-plugin/issues/62),
  [Talos guide](https://docs.siderolabs.com/kubernetes-guides/advanced-guides/device-plugins.md)).

Docker sends SIGTERM to PID 1 and kills it after a grace period (10 s by default), so:

- run under an init that forwards signals: `ENTRYPOINT ["tini","--","ripper"]`, `CMD ["serve"]`;
- set `lifecycle.Spec.PropagationDelay = NoPropagationDelay` (single replica + Recreate,
  exactly the case the kit documents), `DrainTimeout` ~5 s, and the engine worker
  `FinishCurrentCycle: true` with a short `StopTimeout` (~15 s), enough to cancel the
  tool and delete partial output;
- compose `stop_grace_period` and k8s `terminationGracePeriodSeconds` come from
  `Spec.TerminationGracePeriodSeconds()` (5 s drain + 5 s flush + 15 s worker + 5 s margin = 30 s);
- Docker `HEALTHCHECK CMD ["ripper","healthcheck"]` probes the admin `/healthz`, so the
  image needs no curl.

## 5. Dependencies

### 5.1 Go modules

| Module | Why | Notes |
|---|---|---|
| `github.com/spf13/cobra` | commands, flags | only in `internal/cli` |
| `github.com/spf13/viper` | config resolution | only in `internal/cli` |
| `github.com/leftathome/go-service-kit` | lifecycle, obs, httpapi (huma), outbound | pin a tag (v0.3.0+) |
| `github.com/unraid/apprise-go` | notifications | pre-1.0: pin exactly; `Send` takes no `context` and no custom HTTP client, so the backend wraps it with a timeout. It bypasses kit `outbound` (accepted, documented). |
| `golang.org/x/sys/unix` | ioctl backends | phase 6 only |

### 5.2 External tools

| Tool | Decision | Seam / backend |
|---|---|---|
| `makemkvcon` (detect, rip, reg) | keep, exec | `detect/makemkv`, `rip/makemkv`, `mkvkey` registration |
| `abcde` + cdparanoia/lame/flac/eyeD3/metaflac/glyrc | keep, exec | `rip/abcde` |
| `ddrescue` | keep, exec | `rip/ddrescue` |
| user hook scripts (`BLURAYrip.sh`, `DVDrip.sh`, `CDrip.sh`, `DATArip.sh`) | keep, exec | `rip/script` (same names, `+x` requirement and arguments as today) |
| `cdparanoia -Q` | exec now → ioctl later | inside `detect/makemkv` → `detect/native` |
| `eject`, `sdparm` | exec now → ioctl later | `eject/exec` → `eject/ioctl` |
| `curl` (Pushover) | replace | `notify/apprise` |
| `curl` + `grep -P` (beta key) | replace | `mkvkey/forum` using kit `outbound` |
| `grep/sed/cut/date/timeout`, `useradd/groupadd`, `chmod g+rw` | replace | stdlib; numeric `os.Chown`; small symbolic-mode parser |
| python/flask/waitress/docopt | replace | `internal/api` (huma) + `internal/webui` (embed) |
| phusion `my_init`, syslog-ng | remove | kit `lifecycle` + `obs`, tini |

## 6. Architecture

### 6.1 Layout

```
cmd/ripper/main.go            entrypoint: os.Exit(cli.Execute(ctx))
internal/cli/                 cobra commands + viper binding (only importer of cobra/viper)
  root.go serve.go detect.go healthcheck.go version.go
internal/config/              Config struct, defaults, Validate() — no viper import
internal/patchbay/            selects a backend per seam from Config; builds the lifecycle.Spec
internal/disc/                domain: Kind, Disc, ParseDRV (pure)
internal/engine/              poll → detect → rip → finalize → eject → notify; a lifecycle.Worker
internal/output/              dir naming, label sanitising, finished/ move, chown/chmod, partial cleanup
internal/api/                 huma operations (status, log)
internal/webui/               embedded static UI (mounted unless headless)

seams (interface only)        backends (one package each)
internal/runner/              runner/exec            (fake in tests)
internal/detect/              detect/makemkv          detect/native (phase 6)
internal/rip/                 rip/makemkv  rip/abcde  rip/ddrescue  rip/script
internal/eject/               eject/exec              eject/ioctl (phase 6)
internal/notify/              notify/apprise          notify/nop
internal/mkvkey/              mkvkey/env              mkvkey/forum

internal/testutil/fakebin/    fake external tools for integration + parity tests
test/parity/                  legacy-vs-Go parity harness
```

### 6.2 Seams

```go
// internal/runner
type Cmd struct {
    Name string
    Args []string
    Tool string // log attribute; each output line becomes a slog record
}
type Runner interface {
    Run(ctx context.Context, c Cmd) error
    Output(ctx context.Context, c Cmd) ([]byte, error)
}

// internal/detect
type Detector interface{ Detect(ctx context.Context) (disc.Disc, error) }

// internal/rip  — one interface for every disc kind (DRY)
type Result struct{ Dir string } // what to finalize, or clean up on cancel
type Ripper interface{ Rip(ctx context.Context, d disc.Disc) (Result, error) }

// internal/eject
type Ejector interface{ Eject(ctx context.Context) error }

// internal/notify
type Event struct {
    Kind  Kind // Success | Failure | Stopped
    Title string
    Body  string
}
type Notifier interface{ Notify(ctx context.Context, e Event) error }

// internal/mkvkey
type Source interface{ Key(ctx context.Context) (string, error) }
```

`runner/exec` uses `exec.CommandContext` with `SysProcAttr.Setpgid`, a `Cancel` that sends
SIGTERM to the process group (abcde forks encoders), and `WaitDelay` before SIGKILL.

### 6.3 Patchbay

```go
// internal/patchbay — the only place that knows concrete backends.
func Build(ctx context.Context, cfg config.Config, p *obs.Providers) (lifecycle.Spec, error) {
    run := execrunner.New(p.Logger)

    det, err := newDetector(cfg, run) // switch cfg.DetectorBackend { "makemkv", "native" }
    if err != nil {
        return lifecycle.Spec{}, err
    }
    ej, err := newEjector(cfg, run) // switch cfg.EjectBackend { "exec", "ioctl" }
    if err != nil {
        return lifecycle.Spec{}, err
    }

    rippers := map[disc.Kind]rip.Ripper{
        disc.BluRay:  withHook(cfg, "BLURAYrip.sh", run, makemkv.New(run, cfg.StorageBD, cfg.MinLength)),
        disc.DVD:     withHook(cfg, "DVDrip.sh", run, makemkv.New(run, cfg.StorageDVD, cfg.MinLength)),
        disc.AudioCD: withHook(cfg, "CDrip.sh", run, abcde.New(run, cfg.Drive, cfg.AbcdeConf)),
        disc.Data:    withHook(cfg, "DATArip.sh", run, ddrescue.New(run, cfg.Drive, cfg.StorageData)),
    }

    eng := engine.New(engine.Deps{
        Detect: det, Rippers: rippers, Eject: ej, Notify: newNotifier(cfg), Output: output.New(cfg),
    }, cfg.Engine)

    api := httpapi.New(httpapi.Options{Addr: cfg.APIAddr, Middleware: basicAuth(cfg) /* , ... */})
    apiroutes.Register(api.Huma, eng, cfg.LogFile)
    if !cfg.Headless {
        webui.Mount(api, cfg.WebPathPrefix)
    }
    admin := httpapi.NewAdmin(httpapi.AdminOptions{Addr: cfg.AdminAddr, Registry: p.PromRegistry})

    return lifecycle.Spec{
        Servers:          []*http.Server{api.Server, admin.Server},
        PropagationDelay: lifecycle.NoPropagationDelay, // single replica + Recreate: nothing else serves
        Flush:            p.Shutdown,
        Workers: []lifecycle.Worker{{
            Name: "engine", Run: eng.Run, FinishCurrentCycle: true, StopTimeout: 15 * time.Second,
        }},
    }, nil
}
```

`withHook` returns the `rip/script` backend when `/config/<name>` exists and is executable,
and the default backend otherwise. Each swap is visible in exactly one function.

### 6.4 Commands

| Command | Purpose |
|---|---|
| `ripper serve [--headless]` | Default container command. Startup work (§7 phase 4), then `lifecycle.Run(patchbay.Build(...))`. |
| `ripper detect [--raw]` | One-off: print the detected disc, or with `--raw` the unparsed tool output (fixture capture). |
| `ripper healthcheck` | GET admin `/healthz`, exit 0/1 (Docker `HEALTHCHECK`). |
| `ripper version` | Build info (ldflags), also exported by `obs` as `service_build_info`. |

### 6.5 HTTP surface

The public listener (`API_ADDR`) uses huma, so the OpenAPI spec is generated and its docs
UI is on by default. Basic auth via `httpapi.Options.Middleware` when `WEB_USERNAME` and
`WEB_PASSWORD` are both set (constant-time compare).

| Route | Purpose |
|---|---|
| `GET /api/v1/status` | Engine state (`idle`, `detecting`, `ripping`, `waiting_for_eject`), current disc, started-at, last result. The minimum a headless consumer needs. |
| `GET /api/v1/log?lines=N` | Last N records, newest first, plus file size and a `large` flag. |
| `DELETE /api/v1/log` | Truncate the log file. |
| `GET /` … | Web UI (only when not headless), embedded, served via `RawRoute("GET /…")`. |

Admin listener (`ADMIN_ADDR`): kit-provided `/healthz`, `/readyz` (registered check: drive
device present), `/metrics`, pprof when `PPROF_ENABLED=true`. Candidates **not** in scope
until a consumer needs them: `POST /api/v1/eject`, `POST /api/v1/rescan`.

## 7. Phases

Each phase is one PR: CI green, reviewed, merged before the next starts. Phases 1–4 add Go
code and tests only. The shipped image stays on bash/python until phase 5 cuts it over.

### Phase 0 — Safety net
- ✔ `archive` branch; ✔ `.claude/CLAUDE.md`. Protecting `archive` is a manual setting (§10).
- Fixtures under `internal/disc/testdata/`, with a `source` field on each one:
  1. synthetic, built from MakeMKV's robot format;
  2. mined from public upstream issues and forum posts (source URL recorded);
  3. captured on your hardware with `scripts/capture-fixtures.sh` (runs in the legacy image) — authoritative.
  Cases: empty, open, loading, DVD, BD, UHD, audio CD, data CD, data DVD, blank label, garbage.
- `internal/testutil/fakebin` and the `test/parity` harness, running the **legacy** script only (§9.3).
- Tooling copied from go-service-kit: `Makefile` (`lint`, `test`, `vulncheck`, retried tool
  installs), `.golangci.yml`, `renovate.json`. `ci.yml` runs shellcheck + the parity suite.

Exit: the parity suite passes against the legacy script for every scenario.

### Phase 1 — Skeleton that boots
- `go.mod` (Go 1.26), `cmd/ripper`, `internal/cli` (root, `version`, `healthcheck`),
  `internal/config` (+ viper binding, defaults, `Validate`), `obs.Setup`, `lifecycle.Run`
  with the admin listener only.
- `internal/disc` (`ParseDRV` + fixture tests), `internal/output` (mode parser, naming, sanitising).
- CI: `make lint test vulncheck` (race, shuffle).

Exit: `ripper serve` starts, answers `/healthz`, shuts down cleanly on SIGTERM.

### Phase 2 — HTTP surface
- `internal/api` (status, log), `internal/webui` (existing assets ported to read JSON
  records; petite-vue kept), `--headless`, basic-auth middleware, `WEB_PATH_PREFIX`.
- Risk to verify early: path prefix with huma plus otelhttp route labels. Fallback: register
  operations with the prefix rather than wrapping the handler in `StripPrefix`.

Exit: `httptest` coverage of every route, in both modes, with and without auth and prefix.

### Phase 3 — Engine and backends (exec)
- `runner/exec`, `detect/makemkv`, `rip/{makemkv,abcde,ddrescue,script}`, `eject/exec`,
  `notify/{apprise,nop}`, `output`, `engine`, `patchbay`.
- Engine tests use `testing/synctest` (60 s poll, 5 s manual-eject wait) with
  `lifecycle.Spec.Signals` set to an empty slice, as the kit requires inside a bubble.
- The parity harness now runs legacy **and** Go.

Exit: parity passes (modulo §8); every engine branch unit-tested, including cancel-mid-rip cleanup.

### Phase 4 — Startup work (replaces `/etc/my_init.d/ripper.sh`)
- `mkvkey/env`, `mkvkey/forum` (kit `outbound`, polite rate limit, ToS comment at the
  construction site), `settings.conf` update that touches only `app_Key`, `makemkvcon reg`.
- abcde config: use `/config/abcde.conf` untouched if it exists; otherwise render the
  bundled default with `OUTPUTDIR=$STORAGE_CD` into a temp file.
- Ownership via `output` (numeric chown, no `useradd`).

Exit: tests with an `httptest` forum page and fakebin `makemkvcon`.

### Phase 5 — Image cutover
- Multi-stage Dockerfiles: `golang:1.26` builds `CGO_ENABLED=0`; the runtime is `ubuntu:noble`
  + kept tools + `tini`. The manual build keeps its MakeMKV-from-source stage.
- Delete `root/etc/my_init.d`, `root/etc/syslog-ng`, `root/web`, `root/ripper/ripper.sh`,
  `root/ripper/settings.conf`, python, phusion.
- Workflows: publish to `ghcr.io/jacaudi/docker-ripper` (multi-arch as today); fix the
  base-image watcher (`noble`); delete `IssueModerator.yml` and `LabelSponsors.yml`;
  `ci.yml` becomes a required check.
- README rewrite plus migration notes (§8); compose with `init: true`, `stop_grace_period`,
  `HEALTHCHECK`; k8s example manifest.

Exit: CI smoke test of the image against a fakebin drive; one real rip per disc type on your hardware.

### Phase 6 — Native backends (optional, one PR each, hardware-gated)
- `eject/ioctl` (`CDROM_LOCKDOOR` 0 + `CDROMEJECT`).
- `detect/native`: `CDROM_DRIVE_STATUS` / `CDROM_DISC_STATUS` for empty/open/loading and
  audio vs data, with MakeMKV still the source of truth for DVD vs BD and the label.
- Selected with `EJECT_BACKEND` / `DETECTOR_BACKEND`. The exec backend is deleted once the
  native one has proven itself (YAGNI).

## 8. Behaviour changes (migration notes)

| # | Old | New |
|---|---|---|
| D1 | Pattern map with undefined order; `cd2` matches empty drives | Ordered rules on parsed fields; an empty label never implies CD |
| D2 | Data CDs/DVDs sent to abcde/MakeMKV | Routed to the ISO backend (exact DVD/BD signal confirmed with fixtures) |
| D3 | Unrecognised reply below threshold → "rip" branch → eject | Counted as bad; no rip, no eject |
| D4 | `cut -c5` drive index | Parsed integer |
| D5 | `--profile=/config/default.mmcp.xml` (never exists) | `/config/default.mmcp.xml` if present, else the bundled profile |
| D6 | `ALSOMAKEISO` on a CD after abcde has ejected it | ISO made before abcde for CDs |
| D7 | Raw/empty labels used as directory names | Sanitised; empty → `disc_<timestamp>` |
| D8 | Bad-threshold exit leaves a dead ripper and a live UI | Engine error → kit `lifecycle` shuts the process down → restart policy |
| D9 | MakeMKV key printed; `~/.MakeMKV` mode 777 | Key logged redacted; mode 0700 |
| D10 | Pushover via `POVER_APP_TOKEN`/`POVER_USER_KEY`, fixed text | `APPRISE_URLS` (e.g. `pover://USER_KEY@APP_TOKEN`); success/failure/stopped events |
| D11 | Basic auth realm "FeedCrawler", non-constant-time | Constant-time, realm "Ripper" |
| D12 | `/config/ripper.sh` executed | Ignored (hard break); hook scripts remain |
| D13 | `useradd`/`groupadd` at start | Numeric chown; names resolved if present, else `FILEUSERID`/`FILEGROUPID` |
| D14 | `PREFIX`/`USER`/`PASS` | `WEB_PATH_PREFIX`/`WEB_USERNAME`/`WEB_PASSWORD` only |
| D15 | Plain-text `Ripper.log`, tool output raw | JSON records; tool output one record per line |
| D16 | `/api/log/` | `/api/v1/log`, `/api/v1/status`; OpenAPI docs |
| D17 | One port (9090) | API/UI `:9090`, admin `:9091` |
| D18 | `docker stop` kills a rip, leaving partial output | Rip cancelled cleanly, partial output removed |
| D19 | phusion base, python, syslog-ng in image | `ubuntu:noble` + tools + tini + `ripper` |

Core ripper env names (`DRIVE`, `STORAGE_*`, `EJECTENABLED`, `JUSTMAKEISO`, …) are kept unchanged.

## 9. Testing external tools

### 9.1 Unit
A fake `runner.Runner` records each `Cmd` and returns canned output from fixtures. Each
backend is tested through its seam interface, so a new backend reuses the same table of cases.

### 9.2 Integration — `fakebin`
A small Go program, built once in `TestMain` and symlinked as `makemkvcon`, `abcde`,
`ddrescue`, `cdparanoia`, `eject`, `sdparm` (and `curl` for the legacy run). It is driven by
a scenario directory (`FAKEBIN_DIR`):
- appends `{"tool","args"}` to `calls.jsonl`;
- prints `<tool>.stdout` and exits with `<tool>.exit`;
- simulates side effects (MKV/ISO/FLAC files written into the output directory);
- with `<tool>.block`, blocks until signalled, to test cancel-mid-rip cleanup.

Hook scripts are tested with real tiny shell scripts in `testdata/hooks/`.

### 9.3 Parity — legacy vs Go
Every scenario runs twice in a throwaway container (root, real `/config`, `/out`):
1. the legacy `ripper.sh`, vendored read-only from `archive`, with fakebin on `PATH` and a
   fake `sleep` that ends the loop after N iterations;
2. `ripper serve`, with the same fakebin.

The harness compares the normalised tool-call log and the resulting `/out` tree (paths,
modes, owners). A legacy `curl` to Pushover and a Go apprise call to a local fake server both
normalise to a `notify` event. Expected differences are declared per scenario and keyed to
§8, so every deviation is explicit and reviewed.

### 9.4 Hardware (opt-in)
`go test -tags hardware ./...` with `RIPPER_TEST_DRIVE=/dev/sr0`: detection per inserted
disc, eject, a short ISO read. Gates the phase 6 backends. Never runs in CI.

## 10. Repository admin (manual)

- Protect `archive`: Settings → Rules → Rulesets → New branch ruleset → target `archive`;
  enable *Restrict deletions*, *Block force pushes*, *Restrict updates*.
- Enable GitHub Packages for ghcr publishing (workflow `permissions: packages: write`).

## 11. Remaining open points

- **Hook scripts**: assumed to stay (D12 only drops `ripper.sh`). Confirm.
- **apprise-go maturity**: pre-1.0 and not every target is tested upstream. The seam keeps
  it swappable; re-evaluate at phase 3.
- **go-service-template**: its spec lives on an internal GitLab I can't reach. If the
  layout above should mirror it, share the relevant parts.

## References

- go-service-kit: https://github.com/leftathome/go-service-kit
- apprise-go: https://pkg.go.dev/github.com/unraid/apprise-go · https://unraid.net/blog/apprise-go
- Go module layout: https://go.dev/doc/modules/layout
- Device plugins: https://github.com/squat/generic-device-plugin/issues/62 ·
  https://docs.siderolabs.com/kubernetes-guides/advanced-guides/device-plugins.md
- Docker signals / grace period: https://oneuptime.com/blog/post/2026-01-16-docker-graceful-shutdown-signals/view ·
  https://www.netdata.cloud/guides/docker/docker-exit-code-143/
