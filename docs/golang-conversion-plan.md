# Go conversion plan

Status: **proposal** · Baseline: `archive` branch (= `main` @ `21c5796`) ·
Background: [`.claude/CLAUDE.md`](../.claude/CLAUDE.md)

## 1. Goal

Replace the bash ripper (`ripper.sh` + init script) and the Python web UI with one
statically linked Go binary that:

- keeps every documented env var, output path, hook script name and hook argument list working;
- calls the external tools that cannot reasonably be replaced (MakeMKV, abcde, ddrescue)
  through one seam that tests can plug fakes into;
- replaces everything else (curl, grep/sed/cut, eject/sdparm, useradd, python/flask) with Go;
- fixes the known detection and lifecycle bugs listed in CLAUDE.md, each one deliberately
  and recorded as a deviation.

Out of scope: multi-drive support, new output formats, a new web UI design, replacing
abcde's audio pipeline.

## 2. Principles applied

| Principle | What it means here |
|---|---|
| KISS | One binary, one process, one loop. Stdlib first. Two third-party modules at most (`golang.org/x/sync`, later `golang.org/x/sys`). |
| YAGNI | No plugin system, no config files, no feature flags for old behaviour, no interface without a second implementation (a fake counts). |
| DRY | One video ripper for DVD and BluRay; one "run tool, tee output to log" helper; one ownership/permission routine. |
| SOLID | Each package does one job. Interfaces are small (one or two methods) and declared by the package that *uses* them. The hook override is a decorator, so new behaviour doesn't mean editing the rippers. |
| 12-Factor | Config from env only (`config.Load(getenv)`). Logs go to stdout via `slog`, teed to `/config/Ripper.log` for the UI. Port from env. `SIGTERM` cancels the context, which stops child processes (`exec.Cmd.Cancel` + `WaitDelay`). A fatal error exits non-zero and the restart policy takes over. Same image for dev and prod. |
| Idiomatic / modern Go | `log/slog`, `context`, `errors.Join`/`%w`, `net/http` method+path patterns, `embed`, `signal.NotifyContext`, `os.Root` for static files, `testing/synctest` for the poll loop, `t.Context()`, table-driven tests, no package-level state. Toolchain: Go ≥ 1.25 (`synctest`), pinned with `go`/`toolchain` in `go.mod`. |
| Seam | Every side effect sits behind a narrow seam: process execution, drive state, eject, notification, ownership, HTTP. Tests swap the implementation without editing the caller. |
| Patchbay | `cmd/ripper/main.go` is the only place that builds concrete implementations and connects them. Packages receive collaborators and never construct their own. Moving to a native implementation (e.g. ioctl eject) is a one-line change in the patchbay. |

Seams we deliberately **don't** add (YAGNI): a filesystem abstraction (tests use
`t.TempDir()`), a clock interface (`testing/synctest` virtualises time), a config interface.

## 3. Dependency decisions

| Today | Decision | Seam | Rationale |
|---|---|---|---|
| `makemkvcon` (detect, rip, reg) | **Keep, exec** | `proc.Runner` | Proprietary; it is the decryption engine. |
| `abcde` + cdparanoia/lame/flac/eyeD3/metaflac/glyrc | **Keep, exec** | `proc.Runner` | Users customise `abcde.conf`; re-implementing CDDB, ripping, encoding and tagging is a project of its own. |
| `ddrescue` | **Keep, exec** | `proc.Runner` | Its bad-sector retry map is why it's used. |
| `cdparanoia -Q` (audio fallback) | Exec in phase 3 → **native** in phase 6 | `Detector` | `CDROM_DISC_STATUS` ioctl tells audio from data directly. |
| `eject`, `sdparm` | Exec in phase 3 → **native** in phase 6 | `Ejector` | `CDROMEJECT` / `CDROM_LOCKDOOR` ioctls (`x/sys/unix`). |
| `curl` → Pushover | **Replace** | `Notifier` | `net/http` POST form. |
| `curl` + `grep -P` → beta key scrape | **Replace** | none (plain func taking `*http.Client`) | `net/http` + `regexp`. |
| `grep/sed/cut/date/timeout` | **Replace** | — | stdlib; `context.WithTimeout` replaces `timeout 30s`. |
| `sed -i OUTPUTDIR=` in abcde.conf | **Replace** | — | Write a generated conf into a temp dir (see §6.4); stop editing files in the image. |
| `useradd` / `groupadd` | **Remove** | `Owner` (func) | Resolve names with `os/user`; fall back to `FILEUSERID`/`FILEGROUPID`; `os.Chown` takes numbers. |
| `chmod -R g+rw` | **Replace** | — | Small symbolic-mode parser (`[ugoa]*[+-=][rwxX]*`, comma-separated) + `filepath.WalkDir`. |
| python3, flask, waitress, docopt | **Replace** | — | `net/http` + `embed`; same routes and JSON. |
| phusion `my_init`, syslog-ng | **Remove** | — | Single Go process under `tini` (`ENTRYPOINT ["tini","--","ripper"]`). |
| ccextractor, OpenJDK | **Keep in image** | — | MakeMKV uses them. |

## 4. Target layout

```
go.mod                         module github.com/jacaudi/docker-ripper
cmd/ripper/main.go             patchbay: load config, build implementations, connect them, run
internal/config/               Config struct, Load(getenv) (Config, error), validation
internal/disc/                 pure domain: Kind, Disc, ParseDRV(out, device) — no I/O
internal/proc/                 Runner seam + Exec impl (os/exec, context cancel, tee to log)
internal/makemkv/              Detector (info disc:9999 → disc.Disc), video Ripper, key + registration
internal/rip/                  Audio (abcde), ISO (ddrescue), Hook decorator, Placer (dir naming/finish/perm)
internal/drive/                Ejector: exec impl (phase 3), ioctl impl (phase 6)
internal/notify/               Pushover Notifier, Nop
internal/perm/                 symbolic mode parser, recursive chown/chmod
internal/ripper/               Loop: the poll/rip/eject state machine; consumer-side interfaces
internal/web/                  http.Handler for UI + /api/log/, embedded static assets
internal/testutil/fakebin/     fake external tools for integration and parity tests (see §7)
test/parity/                   legacy-vs-Go parity harness (see §7.3)
latest/Dockerfile, manual-build/Dockerfile   multi-stage: build Go, then the runtime image
```

Interfaces live where they are consumed (`internal/ripper` declares `Detector`,
`Ripper`, `Ejector`, `Notifier`); implementations return concrete structs.

### 4.1 Core types and seams (sketch)

```go
// internal/disc
type Kind int
const (Unknown Kind = iota; Empty; Open; Loading; BluRay; DVD; AudioCD; Data)
type Disc struct {
    Kind   Kind
    Index  int    // makemkv drive index (any number of digits)
    Label  string // sanitised for use as a directory name
    Device string
}
func ParseDRV(out []byte, device string) (Disc, error) // pure, deterministic, ordered rules

// internal/proc
type Cmd struct {
    Name   string
    Args   []string
    Stdout io.Writer // nil = log sink
}
type Runner interface{ Run(ctx context.Context, c Cmd) error }

// internal/ripper (consumer-side seams)
type Detector interface{ Detect(ctx context.Context) (disc.Disc, error) }
type Ripper   interface{ Rip(ctx context.Context, d disc.Disc) error }
type Ejector  interface{ Eject(ctx context.Context) error }
type Notifier interface{ Notify(ctx context.Context, msg string) error }

type Loop struct {
    Detect   Detector
    Rippers  map[disc.Kind]Ripper // BluRay, DVD → makemkv; AudioCD → abcde; Data → ISO
    ISO      Ripper               // JUSTMAKEISO / ALSOMAKEISO
    Eject    Ejector
    Notify   Notifier
    Interval time.Duration
    BadLimit int
    Mode     Mode // Normal | JustISO | AlsoISO
    EjectOn  bool
    Log      *slog.Logger
}
func (l *Loop) Run(ctx context.Context) error  // ticks every Interval until ctx done or BadLimit hit
func (l *Loop) Step(ctx context.Context) error // one detect→rip→eject→notify pass (unit-tested)
```

### 4.2 Patchbay (sketch)

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    if err := run(ctx, os.Getenv, os.Stdout); err != nil {
        slog.Error("ripper stopped", "err", err)
        os.Exit(1)
    }
}

// run is the patchbay: the only place where concrete implementations are created and wired.
func run(ctx context.Context, getenv func(string) string, stdout io.Writer) error {
    cfg, err := config.Load(getenv)
    if err != nil {
        return err
    }
    logFile, err := os.OpenFile(cfg.LogFile, os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0o644)
    // ... sink := io.MultiWriter(stdout, logFile); logger := slog.New(slog.NewTextHandler(sink, ...))
    sh := proc.Exec{Log: sink}
    place := rip.Placer{Timestamp: cfg.TimestampPrefix, SeparateFinish: cfg.SeparateRawFinish, Perm: perm.New(cfg.Owner, cfg.Mode)}
    video := rip.Hook{Script: cfg.Hook("BLURAYrip.sh"), Run: sh, Next: makemkv.Ripper{Run: sh, Root: cfg.StorageBD, ...}}
    // ... audio, iso, dvd built the same way
    loop := &ripper.Loop{Detect: makemkv.Detector{Run: sh, Device: cfg.Drive}, ...}

    if err := makemkv.Register(ctx, sh, http.DefaultClient, cfg.Key, cfg.Home); err != nil {
        return err
    }
    g, ctx := errgroup.WithContext(ctx)
    g.Go(func() error { return loop.Run(ctx) })
    g.Go(func() error { return web.Serve(ctx, cfg.Web, cfg.LogFile) })
    return g.Wait()
}
```

`run` takes `getenv` and `stdout`, so the whole binary can be exercised in-process by
tests with a fake environment and fakebin on `PATH`.

## 5. Behaviour: kept vs. intentionally changed

Kept (contract): every env var and default in CLAUDE.md; output directory layout;
`TIMESTAMPPREFIX` format `YYYYMMDD_HHMMSS_`; `finished/` layout; hook names, executable
requirement, argument order; `/config/Ripper.log` and the `/api/log/` JSON shape;
`PREFIX`/`USER`/`PASS` web settings; MakeMKV command lines; abcde command line; 60 s poll.

Intentional deviations (each gets a parity-test override and a CHANGELOG line):

| # | Old | New |
|---|---|---|
| D1 | Pattern map with undefined order; `cd2` matches empty drives | Ordered rules on parsed fields: state first, then media flags. Empty label ≠ CD. |
| D2 | Data CDs/DVDs go to abcde/MakeMKV | Data discs routed to the ISO ripper. CD: no audio tracks per `cdparanoia -Q` (later the `CDROM_DISC_STATUS` ioctl). DVD/BD: the exact signal is confirmed against captured fixtures (Q1). |
| D3 | Unrecognised reply below threshold → "rip" branch → eject | Counts as bad, logs, waits. No rip, no eject. |
| D4 | `cut -c5` drive index | Parsed integer. |
| D5 | `--profile=/config/default.mmcp.xml` (never exists) | Use `/config/default.mmcp.xml` if present, else the bundled `/ripper/default.mmcp.xml`. |
| D6 | `ALSOMAKEISO` on a CD after abcde already ejected | Make the ISO **before** abcde runs for CDs. |
| D7 | Empty/unsafe label used raw | Sanitise; empty label → `disc_<timestamp>`. |
| D8 | Bad-threshold exit leaves web UI running, ripper dead | Process exits non-zero; the container restart policy restarts it (`restart: unless-stopped` added to compose). |
| D9 | MakeMKV key printed in logs | Logged as redacted (`T-…<last 4>`). `~/.MakeMKV` mode `0700`, not `777`. |
| D10 | Pushover fires on fatal path with "finished" text | Notifies on success ("Ripped <label>") and on fatal stop ("Ripper stopped: …"); empty tokens = disabled. |
| D11 | Basic auth non-constant-time, realm "FeedCrawler" | `subtle.ConstantTimeCompare`, realm "Ripper". |
| D12 | `/config/ripper.sh` is user-editable and runs | No longer executed. Hooks are the extension point. If `/config/ripper.sh` exists, log a one-time warning naming the hooks. (See open question Q2.) |
| D13 | Users/groups created with `useradd` | Numeric `chown`; names resolved if they exist, otherwise `FILEUSERID`/`FILEGROUPID`. |

## 6. Phases

Each phase is one PR, CI-green, reviewed, merged before the next starts. The image
stays shippable after every phase. Bash/Python are deleted only in phase 5.

### Phase 0 — Safety net (no Go in the image yet)

- `archive` branch, protected. ✔ (protection is a manual GitHub setting, see §9)
- `.claude/CLAUDE.md`. ✔
- **Fixtures**: commit real `makemkvcon -r --cache=1 info disc:9999` and `cdparanoia -Q`
  output for: empty, open, loading, DVD, BD, UHD, audio CD, data CD, data DVD, blank
  label, unexpected/garbage. Store as `internal/disc/testdata/<case>.txt`.
  Needs a real drive (see Q1); `scripts/capture-fixtures.sh` makes it one command.
- `internal/testutil/fakebin` + `test/parity` harness running the **legacy** script
  against fixtures (§7). This pins down today's behaviour before any Go replaces it.
- CI workflow `ci.yml`: `shellcheck`, then parity suite (legacy only).

Exit: parity harness green against the legacy script for every fixture scenario.

### Phase 1 — Skeleton and pure core

- `go.mod`, `cmd/ripper` (patchbay that only loads config and logs it), `internal/config`,
  `internal/disc` (`ParseDRV` + tests from fixtures), `internal/perm` (mode parser + tests),
  `internal/proc` (`Exec` + tests with fakebin).
- CI: `gofmt -l`, `go vet`, `staticcheck`, `go test -race ./...`, `govulncheck`.

Exit: ≥90 % coverage on `disc`, `config`, `perm`; CI green.

### Phase 2 — Web UI port

- `internal/web`: same routes, `embed` the existing static assets unchanged (drop the
  `{% raw %}` wrapper), `os.Root` for static file serving, efficient tail (seek from end
  in byte mode), `PORT` env with default `9090`.
- Ship it first: the Dockerfile runs `ripper web` instead of `web.py` (the only
  subcommand we add, and it gets removed again in phase 5). Python stays for nothing else.

Exit: `httptest` suite covers GET/DELETE/auth/prefix/redirect; manual check in the image.

### Phase 3 — Ripper loop (exec everywhere)

- `makemkv.Detector`, `makemkv.Ripper`, `rip.Audio`, `rip.ISO`, `rip.Hook`, `rip.Placer`,
  `drive.ExecEjector` (eject → sdparm fallback), `notify.Pushover`, `ripper.Loop`.
- Loop tests use `testing/synctest` for the 60 s interval and 5 s manual-eject poll, so
  they run instantly.
- Parity harness now runs **both** legacy and Go for every scenario (§7.3).

Exit: parity green (modulo D1–D13), all loop branches unit-tested.

### Phase 4 — Startup work (replaces `/etc/my_init.d/ripper.sh`)

- `makemkv.FetchBetaKey`, `makemkv.EnsureSettings` (only touches `app_Key`, keeps other
  lines such as `app_ccextractor`), `makemkv.Register` (exec `makemkvcon reg <key>`).
- abcde config: if `/config/abcde.conf` exists use it untouched (today's precedence);
  otherwise render the bundled default with `OUTPUTDIR=$STORAGE_CD` into a temp file.
- Ownership via `perm` (no `useradd`).

Exit: tests with `httptest` forum page + fakebin `makemkvcon`.

### Phase 5 — Cut over the image

- Multi-stage Dockerfiles: `golang:<pinned>` builds `CGO_ENABLED=0` → runtime stage
  `ubuntu:noble` + tools + `tini`; `ENTRYPOINT ["tini","--","ripper"]`. Manual build keeps
  its MakeMKV-from-source stage.
- Remove `root/etc/my_init.d`, `root/etc/syslog-ng`, `root/web/web.py`, `root/ripper/ripper.sh`,
  `root/ripper/settings.conf`, python packages, `curl`/`wget`/`git` if unused, phusion base.
- Fix workflows: base-image watcher, registry (Q3), Go CI as required check.
- README: migration notes (D1–D13), hooks as the extension point.

Exit: image smoke test in CI (`docker run … ripper` against fakebin drive scenario);
manual rip of one disc per type on real hardware (Q1).

### Phase 6 — Native replacements (optional, one PR each, hardware-verified)

- `drive.IoctlEjector` (`CDROM_LOCKDOOR` 0 + `CDROMEJECT`) → swap in patchbay, delete exec ejector, drop `eject`/`sdparm` packages.
- Native state/media check (`CDROM_DRIVE_STATUS`, `CDROM_DISC_STATUS`) → replaces
  `cdparanoia -Q` and handles empty/open/loading without calling MakeMKV. MakeMKV stays the
  source of truth for DVD vs BD and for the disc label.
- Each lands only after a `//go:build hardware` test passes on a real drive.

## 7. Testing external tools

Four layers, cheapest first. All but layer 4 run in CI without a drive.

### 7.1 Unit — in-memory fakes

A `fakeRunner` records every `proc.Cmd` and returns canned stdout/exit codes from
fixtures. Covers argument construction, branching, error handling. `ParseDRV` is pure
and table-tested directly against `testdata/*.txt`.

### 7.2 Integration — `fakebin`

`internal/testutil/fakebin` is a small Go program that `TestMain` builds once into a
temp dir and symlinks as `makemkvcon`, `abcde`, `ddrescue`, `cdparanoia`, `eject`,
`sdparm`, `curl`. Behaviour is driven by a scenario dir (`FAKEBIN_DIR`):

- appends `{"tool":…, "args":[…]}` to `calls.jsonl`;
- prints `<tool>.stdout`, exits with `<tool>.exit` (default 0);
- simulates side effects, e.g. `makemkvcon mkv … <dir>` writes `title_t00.mkv`,
  `ddrescue <dev> <iso>` writes the ISO, `abcde` writes `Artist-Album/01.Track.flac`.

`PATH` is prepended with the fakebin dir, so `proc.Exec` runs real processes, with
real exit codes, context cancellation and output teeing. User hooks are tested
with real tiny shell scripts in `testdata/hooks/`.

### 7.3 Parity — legacy script vs. Go binary

`test/parity` runs every scenario twice inside a throwaway container (root, writable
`/config`, `/ripper`, `/out`, so neither side is patched):

1. the **legacy** `ripper.sh` from the `archive` branch (vendored read-only into
   `test/parity/legacy/`), with fakebin on `PATH` and a fake `sleep` that stops the loop
   after N iterations;
2. the **Go** binary, with the same fakebin and the same `FAKEBIN_DIR`.

It then compares, after normalising timestamps:

- the external-tool call log (`calls.jsonl`; a legacy `curl` to Pushover and a Go HTTP
  call to a local fake Pushover server both map to a `notify` event);
- the resulting `/out` tree (paths, ownership, mode bits).

Expected differences are declared per scenario in `test/parity/scenarios.go`, keyed to
D1–D13, so every deviation is explicit and reviewed. Scenarios: each fixture ×
{default, `JUSTMAKEISO`, `ALSOMAKEISO`, `SEPARATERAWFINISH`, `TIMESTAMPPREFIX`,
`EJECTENABLED=false`, each hook present, eject-failure fallback, Pushover on/off,
bad-response threshold}.

This answers "a way to call the scripts we can test against": the legacy scripts become
an executable spec, and the Go code must match it or document why not.

### 7.4 Hardware — opt-in

`//go:build hardware` tests, run with `RIPPER_TEST_DRIVE=/dev/sr0 go test -tags hardware ./...`
on a machine with a drive: detection per inserted disc, eject, a short ISO read. Also used
to (re)capture fixtures. Never in CI.

## 8. CI

New `.github/workflows/ci.yml` on PRs and pushes:

1. `gofmt -l` empty, `go vet`, `staticcheck`, `govulncheck`
2. `go test -race -shuffle=on ./...`
3. parity suite (Docker)
4. `docker build` of both Dockerfiles with no push, plus a smoke test

`BuildImages.yml` publishes only from `main` after CI passes.

## 9. Open questions (need your decision)

- **Q1 — Hardware fixtures.** I can't capture real `makemkvcon` output here. Can you run
  `scripts/capture-fixtures.sh` (phase 0) with each disc type? Until then, fixtures are
  synthesised from the robot-mode format and marked `synthetic`.
- **Q2 — `/config/ripper.sh` customisation (D12).** Recommended: hooks only, plus a
  startup warning. Alternative: keep a `LEGACY_SCRIPT=true` escape hatch that execs the
  old script (costs keeping bash + python in the image, so I'd rather not).
- **Q3 — Registry.** Workflows push to `rix1337/docker-ripper` on Docker Hub with upstream
  secrets. Publish this fork to `ghcr.io/jacaudi/docker-ripper` instead?
- **Q4 — Web env names.** Keep `PREFIX`/`USER`/`PASS` (compose) only, or also accept the
  README's `OPTIONAL_WEB_UI_*` names? `USER` clashes with the standard shell variable.
  Recommended: accept both and warn on the short names.
- **Q5 — Upstream.** Is this fork meant to diverge permanently, or should changes stay
  upstreamable to `rix1337/docker-ripper`? That affects how big each PR should be.

## 10. Repository admin (manual)

Protecting `archive` (Settings → Rules → Rulesets → New branch ruleset):
target `archive`; enable *Restrict deletions*, *Block force pushes*, *Restrict updates*
(no bypass, or bypass for admins only). Rulesets and classic branch protection are
available on public repos on any plan; private repos need GitHub Pro/Team or higher.
