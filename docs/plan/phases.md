# Phases — executable task list

## How to execute

- Do the tasks **in order**. Each task lists **Files**, **Do**, **Test** and **Done when**.
- A task is done only when its "Done when" command exits 0.
- Commit after every task with a Conventional Commit message:
  `<type>(<package>): <summary> (P<phase>.<n>)` (L§4).
- Where to find details:
  - [`contracts.md`](contracts.md) is cited as C§n;
  - [`testing.md`](testing.md) as T§n;
  - [`tooling.md`](tooling.md) as L§n.
  
  Copy names and values from there exactly.
- If a task needs a decision that isn't written down, **stop and ask**. Do not guess.
- One PR per phase. CI must be green before the next phase starts.
- Tasks marked **(owner)** need the repository owner (hardware or GitHub settings).

Pinned versions (Renovate keeps them current afterwards):

| Thing | Version |
|---|---|
| Go | `go 1.27.0` + `toolchain go1.27.2` in go.mod; `golang:1.27` image |
| Node | 24 LTS (`node:24-alpine`); `engines.node: ">=24"` |
| Task | v3.54.0 |
| golangci-lint | v2.14.0 (built from source by `task tools`) |
| govulncheck | v1.8.0 |
| cobra / viper | v1.10.2 / v1.21.0 |
| go-service-kit | v0.3.0 |
| apprise-go | v0.3.3 (exact) |
| react / react-dom | 19.3.0 |
| antd | 6.6.5 |
| vite / @vitejs/plugin-react | 8.3.4 / 6.1.2 |
| vitest / @testing-library/react / @testing-library/dom / @testing-library/jest-dom / jsdom | 5.0.3 / 16.3.3 / ^10 / latest 6.x / 30.1.2 |
| @types/react / @types/react-dom | 19.x |
| typescript | **5.9.3** (not 6.x/7.x: openapi-typescript needs `^5`, typescript-eslint 8.71 needs `<6.1`) |
| eslint / typescript-eslint / eslint-plugin-react-hooks / prettier | 10.12.0 / 8.71.1 / 7.1.1 / 3.9.9 |
| FFmpeg (static, in the image) | n8.1.3 (the 9.x major is a separate Renovate PR: check MakeMKV builds against it) |
| cyanrip | v0.9.3.1 |
| openapi-typescript / openapi-fetch | 7.13.0 / 0.17.0 |
| @scalar/api-reference | 1.73.1 |
| GitHub Actions | current major, pinned by commit SHA (L§3) |

---

## Phase 0 — Foundations

**P0.1 Go module and tooling**
- Files: `go.mod`, `taskfile.yml`, `.golangci.yml`, `renovate.json`, `.gitignore` (append), `.dockerignore`.
- Do:
  - `go mod init github.com/jacaudi/docker-ripper`; set `go 1.27.0` and `toolchain go1.27.2`.
  - Copy `taskfile.yml`, `.golangci.yml` and `renovate.json` **exactly** from L§1, L§2 and L§6.
    Delete any `Makefile`.
  - `.gitignore`: `internal/webui/dist/*`, `!internal/webui/dist/.gitkeep`, `web/node_modules/`, `bin/`.
  - `.dockerignore`: `web/node_modules`, `bin`. Do **not** ignore `.git` (C§5.7).
- Done when: `task tools` exits 0 and `bin/golangci-lint version` prints `built with go1.27`.
  (`task lint test vuln` need at least one Go package; they become the done-criterion from P1.4.)
- Dependencies: add each module with `go get <module>@<version>` (versions in the table above) in the
  task that first imports it, never earlier.

**P0.2 Releases and commit conventions**
- Files: `release-please-config.json`, `.release-please-manifest.json`, `.github/workflows/release.yml`
  (the `release-please` job only), `.github/pull_request_template.md` (checklist: conventional title,
  `task lint test vuln` run (`task check` once `web/` exists), migration note if user-visible).
- Do: L§4, L§5. The `release-please` job needs `permissions: contents: write, pull-requests: write`
  at job level (top level stays `contents: read`).
- **(owner):** create the GitHub App and its secrets; set squash-only merging; allow Actions to
  create PRs (main plan §9).
- Done when: after merge, release-please opens a release PR.

**P0.3 fakebin**
- Files: `internal/testutil/fakebin/main.go`, `internal/testutil/fakebin/fakebin_test.go`,
  `internal/testutil/fakebintest/fakebintest.go`.
- Do: implement T§3 exactly (including `pid` in `calls.jsonl`).
- Optional simpler variant (pick one and note it in the PR): instead of `go build`, re-exec the test
  binary itself: `TestMain` runs fakebin's `main` when `os.Getenv("FAKEBIN") == "1"`, and `Install`
  symlinks `os.Args[0]` under each tool name.
- Test: one case per protocol row, including the `<sub>` counter (`makemkvcon reg` then `info` → `info.1`)
  and that `.block` never blocks an `info` scan (per-call stdout/stderr/exit, each `creates` template, block/release,
  SIGTERM → 143 without creates).
- Done when: `go test ./internal/testutil/...` passes.

**P0.4 Synthetic fixtures**
- Files: every `.txt` listed in T§2.
- Done when: the files exist (they are used in P2.2).

**P0.5 CI and security workflows** (the repo starts with **no** workflows; the inherited upstream ones were removed in PR #1)
- Files: `.github/workflows/ci.yml` (jobs `pr-title`, `go`), `.github/workflows/security.yml`.
- Do: L§3, L§7. CodeQL `go` now; `javascript-typescript` is added in P3.3.
- Done when: the PR is green, and CodeQL/OSV/Scorecard results appear under Security → Code scanning.

---

## Phase 1 — Core

**P1.1 `internal/disc`**
- Do: types from C§2.1, including `String`, `MarshalText`, `UnmarshalText` and `DriveID`.
- Test: the string of every value; `json.Marshal` → `json.Unmarshal` round-trips every value; an unknown string → error.
- Done when: `go test ./internal/disc` passes.

**P1.2 `internal/config`**
- Files: `config.go` (struct with every field from C§1.2, plus `UmaskMode()` and `AudioOffsets()`), `validate.go`, `prefix.go` (`NormalizePrefix`), tests.
- Test: one row per validation rule in C§1.2; the prefix examples from C§1.3; a fully valid config
  gives no error; three bad values give one error that mentions all three env names;
  `RIPPER_AUDIO_DRIVE_OFFSETS=sr0=6,sr1=-667` → `{sr0:6, sr1:-667}`; `sr0=x` or a duplicate drive → error.
- Done when: `go test ./internal/config` passes.

**P1.3 `internal/cli`: root, viper, version**
- Files: `cli.go` (`Execute(ctx) int`, C§5.3), `root.go`, `load.go`
  (`load(cmd *cobra.Command) (config.Config, error)`, C§1.1, with the one key table), `version.go` (C§5.7;
  the `version` command prints `version()` and a newline, exit 0), `load_test.go`.
- Test (use `t.Setenv`):
  - defaults;
  - each `RIPPER_*` maps to its field;
  - `RIPPER_DRIVES` unset → empty (auto-discover); `/dev/sr0,/dev/sr1` → two entries; duplicate basenames → error;
  - `RIPPER_AUDIO_FORMATS=flac,ogg` → error; `RIPPER_MAX_PARALLEL_JOBS=-1` → error;
  - **empty env → default**;
  - `--headless` overrides `RIPPER_HEADLESS=false`;
  - comma list for `RIPPER_APPRISE_URLS`;
  - durations;
  - invalid values → one joined error;
  - an unknown legacy name (`EJECTENABLED`) has no effect.
- Done when: `go test ./internal/cli` passes.

**P1.4 `cmd/ripper/main.go`**
- Do: `func main() { os.Exit(cli.Execute(context.Background())) }`.
- Done when: `go build ./cmd/ripper && ./ripper version` prints a version, and `task lint test vuln` passes.

**P1.5 `internal/output`**
- Files: `planner.go` (C§3.5), `perm.go` (C§5.4: `fs.WalkDir(p.root.FS(), rel, …)` + `p.root.Chmod`), tests.
- Test:
  - sanitise: `"Movie: Part 1/2"` → `"Movie_ Part 1_2"`; `".."` and `""` → `disc_<ts>`;
  - Prepare path shape (`<kindDir>/.staging/20060102_150405-<drive>/<sanitised label>`);
  - Finalize rename;
  - collision → `_<ts>`; a second collision → error;
  - Cleanup;
  - CleanStaging;
  - modes for umask `002` and `022`: dirs, plain files, executable files;
  - chown is skipped when UID and GID are −1;
  - a label of `../../etc` stays inside the root;
  - `CheckWritable` succeeds twice concurrently.
- Done when: `task lint test` passes.

**P1.6 `internal/logring`**
- Do: C§2.3. Mutex-guarded; capacity 2000. `Write` copies each `\n`-terminated record (a partial
  trailing line is buffered until its newline) and never returns an error. `Lines(n)` returns
  newest first; `n > len` returns all.
- Test: wraparound; ordering; partial writes; concurrent writes under `-race`.
- Done when: `go test -race ./internal/logring` passes.

---

## Phase 2 — Engine and backends

**P2.0 `internal/telemetry`**
- Do: C§6.5. It is the only package that imports `obs`.
- Test: `Setup` with `Config{Output: &buf, SpanExporter: tracetest.NewInMemoryExporter()}` returns a
  working Logger, Tracer and Meter; a log record reaches both `buf` and the ring; `Shutdown` is idempotent.

**P2.1 `internal/runner` + `execrunner`**
- Do: C§2.2, C§5.1, plus its metrics (C§6.2) and `tool.run` span (C§6.4).
- Test (fakebin):
  - stdout and stderr lines are logged with the `tool` attr;
  - argv never appears in any log record;
  - `Cmd.Dir` is honoured (fakebin records `dir`);
  - exit code → `ExitError` via `errors.AsType`;
  - cancelling during `.block` returns within 6 s, and `syscall.Kill(-call.PID, 0)` returns `ESRCH` afterwards
    (the fake is the process-group leader, so its pid is the pgid);
  - `Output` is capped at 1 MiB;
  - `PRGV:`/`PRGC:`/`PRGT:` lines are not logged; other lines are;
  - `tool finished` / `tool failed` records carry `duration_ms` / `exit_code`;
  - `ripper_tool_runs_total` has the right `result` (in-memory metric reader);
  - a `tool.run` span exists (in-memory span exporter).

**P2.2 `detect/makemkv`**
- Do: `ParseDRV` (C§3.1, all drives) and the `Detector` (C§3.2).
- Test: the T§2 expectation table; detector cases with fakebin: timeout (`.block` with a 1 s
  constructor timeout), the cdparanoia `audio`/`no_audio` paths per device
  (`cdparanoia.sr1.stdout`), a device in `busy` is not probed, makemkvcon exit 1, `no_drives`, `garbage`, `bad_state`.

**P2.3 Rip backends**
- Files: `rip/makemkv` (embed `default.mmcp.xml`, moved from `root/ripper/`), `rip/ddrescue`, `rip/cyanrip`.
- Do: argv exactly as in C§3.6, including the profile resolution and the cyanrip `-s`/`-D`/`-T` flags,
  retry and folder rule. The embedded profile is written by `makemkv.New` to `os.MkdirTemp("", "ripper-")`
  as `default.mmcp.xml` (mode 0o600) and left for the process lifetime (no Close).
  makemkv and ddrescue return `name == ""`; cyanrip returns the album folder name.
- Test (fakebin):
  - `calls.jsonl` argv and `dir`;
  - the profile override wins over the embedded default;
  - ddrescue passes both `.iso` and `.map`;
  - cyanrip: argv equals the C§3.6 Go argv exactly (including the `-D` literal and `-R 1`); success →
    files moved up and `name` returned; first exit 1 → a second call with `-N` instead of `-R 1`;
    both fail → error; zero or two created folders → error; cancel → prompt return.

**P2.3b cyanrip behaviour check (owner, hardware; does not block)**
- Run `cyanrip -d /dev/srN -o flac` once with a CD that **is** in MusicBrainz and once with one that
  **isn't**. Record each exit code and the folder layout.
- If cyanrip doesn't exit non-zero on a MusicBrainz miss, **stop and ask**: the retry rule in C§3.6
  must change.

**P2.4 `eject/execeject`**
- Do: the C§3.6 sequence, with `device` as an argument. Inject waits as
  `sleep func(context.Context, time.Duration) error`.
- Test matrix: success → one `eject` call, `nil`; `eject.exit=1` and sdparm ok → calls `eject`,
  `sdparm --command=unlock`, `sdparm --command=eject`, result `nil`; `eject.exit=1` + `sdparm.2.exit=1`
  → an error wrapping `*runner.ExitError{Tool: "sdparm"}`.

**P2.5 `notify/apprise` + `notify/nop`**
- Do: C§5.2 (with the mutex), C§3.7.
- Test against `json://127.0.0.1:<httptest port>`: title, body and type; a timeout returns
  `context.DeadlineExceeded`; an invalid URL fails `New`; 10 concurrent `Notify` calls are serialised (race-free).

**P2.6 `internal/makemkvkey`**
- Do: C§5.5 (`FetchBetaKey`), `Register` (C§3.6, via `Output` with a 30 s timeout, output discarded) and `RegistrationResult` (C§2.3).
- Test: an httptest page with a key → the key; a page without → error; 500 → error.
  `Register` runs `makemkvcon reg <key>` via fakebin, and the key appears in no log record.

**P2.7 `internal/engine`**
- Files: `engine.go`, `scan.go`, `job.go`, `metrics.go`, `enginetest/fakes.go` (T§1), tests.
- Do: C§2.4, C§3.3 (**follow the concurrency rules exactly**), C§3.4, plus metrics (C§6.2), spans (C§6.4)
  and the log messages at the points C§3.3 names (C§6.3).
- Tests (all under synctest, fakes per T§1):
  - every cell of C§3.4;
  - discovery: drives appear and disappear; a drive with a running job is not removed;
  - `Include` filter;
  - Empty/Open/Loading → idle;
  - per-drive Unknown ×5 → `unusable`, one Failure notification, recovery → `idle`;
  - scanner errors ×5 → one Failure notification; `CheckScanner` fails;
  - success → eject → `awaiting_removal` → no re-rip while Inserted → `idle` on Empty;
  - `Eject: false`, and a failed eject: no re-rip;
  - a failure → Cleanup, Failure notification, eject, `awaiting_removal`;
  - queue: two drives with `MaxParallel 0` run concurrently; with `MaxParallel 1` the second waits
    (`queued`) and starts after the first finishes; FIFO order across three drives;
  - cancel: a running job → Cleanup, Stopped notification via the detached ctx, no eject; queued jobs
    → `cancelled`; `Run` returns nil after the workers finish;
  - `Jobs()` ordering and the history cap (51 jobs → 50 kept); job IDs unique under a fixed clock (seq);
  - a queued job whose disc is removed → `cancelled`, never started;
  - `DriveStatus.Job` shows the current job, then the last finished one;
  - `Started`, `CheckLoop` and `CheckDrives` semantics;
  - metrics (in-memory reader) and spans (in-memory exporter).

**P2.8 `internal/health` + patchbay + `serve` (headless, admin only)**
- Files: `internal/health/health.go`, `internal/patchbay/backends.go`, `internal/patchbay/spec.go`,
  `internal/cli/serve.go`, `internal/cli/healthcheck.go`.
- Do:
  - `health`:
    - `NewAdminServer` and the three Readiness instances, with exactly the checks in C§6.1;
    - `type Handlers struct{ Startup, Health, Ready *httpapi.Readiness; Registry *prometheus.Registry }`;
    - `func Checks(cfg config.Config, eng *engine.Engine, planner *output.Planner, reg *makemkvkey.RegistrationResult, startup *atomic.Bool) Handlers`.
  - `serve` builds the runner with `execrunner.New(tel.Logger, tel.Tracer, tel.Meter)` and passes it on.
  - `func Backends(cfg config.Config, run runner.Runner, tel *telemetry.Telemetry, planner *output.Planner) (engine.Deps, error)`:
    one instance of each backend (C§2.3); `notify/nop` when `AppriseURLs` is empty; `engine.NewMetrics(tel.Meter)`.
  - `func Spec(cfg config.Config, eng *engine.Engine, tel *telemetry.Telemetry, h health.Handlers, ring *logring.Ring) (lifecycle.Spec, error)`:
    - the admin server from `health.NewAdminServer`; `Readiness: h.Ready`;
    - `PropagationDelay: lifecycle.NoPropagationDelay` (comment: single replica + Recreate);
    - `DrainTimeout: 5*time.Second`; `Flush: tel.Shutdown`; `Logger`; `MeterProvider`;
    - **one** worker: `lifecycle.Worker{Name: "engine", Run: eng.Run, FinishCurrentCycle: true, StopTimeout: 15*time.Second}`.
      The kit cancels the context immediately; the 15 s is the cleanup budget for every running job.
  - `serve` follows C§3.8 and sets the startup flag.
  - `healthcheck` per C§6.1.
- Test (patchbay, ephemeral listeners):
  - `/livez` 200;
  - `/startupz` 503 then 200;
  - `/healthz` lists every C§6.1 check; 503 when `PATH` lacks a tool;
  - `/readyz` 200, and 503 when no drives are discovered;
  - `/metrics` serves `ripper_` series;
  - `ripper healthcheck` exits 0 and 1 accordingly.

**P2.9 `ripper detect`**
- Do: C§5.6.
- Test: a JSON array for one and two drives via fakebin; exit 0 for no drives; exit 1 on makemkvcon exit 1.
- **(owner):** run `ripper detect --raw` with a DVD, a BluRay, an audio CD, a data CD, an empty
  drive and an open tray, and with two drives if available. Paste the output into the T§2 fixtures
  and fix the expectation table if the hardware disagrees.

Phase done when: `task lint test vuln` passes (`task check` needs `web/`, which arrives in P3.3).

---

## Phase 3 — HTTP and UI

**P3.1 `internal/api`**
- Files: `api.go` (`Register(h huma.API, prefix string, eng EngineView, logs LogSource)`, with
  `type EngineView interface{ Drives() []engine.DriveStatus; Jobs() []engine.Job }` and
  `type LogSource interface{ Lines(n int) []string }`),
  `auth.go` (C§4.3), tests, `testdata/openapi.json` (golden).
- The log input is ``struct{ Lines int `query:"lines" default:"200" minimum:"1" maximum:"2000"` }``.
- Golden update flag: `var update = flag.Bool("update", false, "rewrite testdata/openapi.json")`.
- Test (`humatest`):
  - status 200 with one entry per drive (sorted by ID) and the queued/running counts;
  - jobs 200 in `engine.Jobs()` order;
  - log 200 with N lines, newest first; `lines=0` → 422; `lines=2001` → 422;
  - auth: no creds → 401 with header; wrong → 401; right → 200; disabled when either is empty.
  - Golden: `OpenAPI().MarshalJSON()` with prefix `""` equals the golden file; `-update` rewrites it.

**P3.2 `internal/apidocs` (Scalar)**
- Files: `scripts/vendor-scalar.sh`, `internal/apidocs/assets/{docs.html,docs-init.js,standalone.js,SHA256}`, `apidocs.go`, tests.
- `vendor-scalar.sh <version>`:
  - download `https://registry.npmjs.org/@scalar/api-reference/-/api-reference-<version>.tgz`;
  - verify its sha512 against `dist.integrity` from `https://registry.npmjs.org/@scalar/api-reference/<version>`
    (`curl … | jq -r .dist.integrity`, strip `sha512-`, `base64 -d | xxd -p -c 256`, compare with `sha512sum`);
  - extract `package/dist/browser/standalone.js`;
  - write the file's sha256 to `SHA256`.
- `Mount(api *httpapi.API, prefix string)` registers the docs and OpenAPI routes from C§4.1, with
  the CSP from C§4.4 (`docs-init.js` exactly as in C§4.4, including `proxyUrl: ''`).
- Test: `/docs` 200 with the exact CSP; an asset 200; `/openapi.json` 200 with valid JSON;
  `standalone.js` matches `SHA256`.

**P3.3 Frontend scaffold (`web/`)**
- Files:
  - `web/package.json` (exact pins; scripts `dev`, `build`, `gen`, `lint`, `format:check`, `typecheck`, `test`);
  - `web/package-lock.json`;
  - `web/vite.config.ts` (`base: './'`, `build.outDir: '../internal/webui/dist'`, `build.emptyOutDir: true`,
    `build.modulePreload.polyfill: false`);
  - `web/tsconfig.json` (strict);
  - `web/eslint.config.js` (flat; typescript-eslint `strictTypeChecked` + react-hooks);
  - `web/.prettierrc`;
  - `web/index.html` (`<meta name="csp-nonce" content="__CSP_NONCE__">`, `<div id="root">`);
  - `internal/webui/dist/.gitkeep`.
- Do: `"gen": "openapi-typescript ../internal/api/testdata/openapi.json -o src/api/schema.d.ts"`, and
  commit `schema.d.ts`. Add the `ui` job to `ci.yml` and `javascript-typescript` to CodeQL (L§7).
- Done when: `task ui:gen ui:lint` passes and `git diff --exit-code web/src/api/schema.d.ts` is clean.

**P3.4 Frontend app (`web/src/`)**
- `main.tsx`: read the nonce from the meta tag; render `<App nonce={…}/>`.
- `App.tsx`:
  - `ConfigProvider` with `csp={{ nonce }}` and `theme={{ algorithm: theme.darkAlgorithm }}`;
  - `Layout` with a Header titled "Ripper", and Content holding a `Row` of `StatusCard`s (one per entry in
    `drives`, `Col` span 24 on xs, 12 on lg), then `JobsTable`, then `LogPanel`. When there are no
    drives, show an AntD `Empty` with "No drives detected".
- `api/client.ts`: `createClient<paths>({ baseUrl: '.' })` (openapi-fetch).
- `hooks/usePoll.ts`: `usePoll(fn, ms)` runs at mount and then every `ms` via `setInterval`, skips
  ticks while `document.hidden`, and clears on unmount.
- `hooks/useStatus.ts`: polls `GET /api/v1/status` every 5000 ms; `hooks/useJobs.ts`: polls `GET /api/v1/jobs` every 5000 ms.
- `components/JobsTable.tsx`: AntD `Table` (`size="small"`, `pagination={{ pageSize: 10 }}`); columns
  job ID, drive, kind, label, state (`Tag`: queued `default`, running `processing`, succeeded `success`,
  failed `error`, cancelled `warning`, skipped `default`), queued/started/finished, paths, error.
- `components/StatusCard.tsx` (props: one `components["schemas"]["DriveStatus"]`; no fetching of its own;
  the job fields come from `status.job`):
  - title = drive ID and device;
  - AntD `Card` + `Descriptions`: state, disc kind, label, device, started at + live elapsed,
    bad responses, last result (outcome, error, paths);
  - state `Tag` colours: idle `default`, queued `blue`, ripping `processing`, ejecting `cyan`,
    awaiting_removal `gold`, unusable `red`;
  - shows the current disc and job ID;
  - an `Alert` (error) when the state is `unusable`.
- `components/LogPanel.tsx`:
  - polls `GET /api/v1/log?lines=200` every 10000 ms;
  - AntD `Table` with `size="small"`, `pagination={false}`, `scroll={{ y: 600 }}`, columns
    time / level / drive / tool / message;
  - each line is `JSON.parse`d into `{time, level, tool, msg, line}`; message = `line ?? msg`;
  - unparseable lines render as `{msg: raw}` in monospace.
- No router, no login form (the browser handles basic auth), no clear button.
- Test (vitest + jsdom + testing-library, `fetch` mocked with `vi.fn`): StatusCard renders a tag
  for each state and the unusable alert; App renders two cards for two drives, and `Empty` for none;
  JobsTable renders each state; LogPanel renders a parsed row and a raw row.
- Done when: `task ui:lint ui:test ui:build` passes and `internal/webui/dist/index.html` exists.

**P3.5 `internal/webui`**
- Files: `webui.go` (`//go:embed all:dist`; `uiAssets, _ = fs.Sub(dist, "dist/assets")`; favicon from
  `dist/favicon.ico`; `Mount(api *httpapi.API, prefix string) error`), tests.
- Do: the C§4.1 routes marked "!headless", with the CSP + nonce from C§4.4. `Mount` returns an
  error when `dist/index.html` is missing, so `serve` without `--headless` fails loudly.
- Test:
  - `/` 200, and the CSP `nonce-` matches the meta tag;
  - an asset has immutable caching;
  - prefix `/ripper` works, and `/ripper` → 301 `/ripper/`;
  - a missing dist returns an error.

**P3.6 Wire HTTP into the patchbay**
- Do: in `Spec`, build the API listener with `httpapi.New` (C§4.1 options) and call `api.Register`
  (EngineView = engine, LogSource = ring) and `apidocs.Mount`; call `webui.Mount` unless headless;
  add the root redirects (`http.Redirect(w, r, P+"/", http.StatusMovedPermanently)` for `GET P`;
  `http.Redirect(w, r, P+"/"` or `P+"/docs", http.StatusFound)` for `GET /{$}`). Add the API server to `Servers`.
  Per-route `http.StripPrefix` (after the mux matched) is fine; only listener-wide StripPrefix is forbidden.
- Test: hit every route in both modes.

**P3.7 End-to-end smoke**
- Do: T§4, every scenario, including `two_drives_parallel` and the observability assertions.
- Phase done when: `task check` passes.

---

## Phase 4 — Cutover

**P4.1 Dockerfile (one image, distroless)**
- Files: `Dockerfile` (repo root), `scripts/build-media.sh`, `scripts/collect-rootfs.sh`, `.dockerignore`.
- `Dockerfile` (fill in the pins; Renovate maintains them, L§6):
```dockerfile
# syntax=docker/dockerfile:1
ARG DEBIAN=trixie-YYYYMMDD-slim     # renovate: datasource=docker depName=debian versioning=regex:^trixie-(?<major>\d{8})-slim$
ARG MAKEMKV_VERSION=X.Y.Z           # renovate: datasource=custom.makemkv depName=makemkv
ARG CYANRIP_VERSION=v0.9.3.1        # renovate: datasource=github-tags depName=cyanreg/cyanrip
ARG FFMPEG_VERSION=8.1.3            # renovate: datasource=github-tags depName=FFmpeg/FFmpeg

FROM node:24-alpine AS ui
WORKDIR /src/web
COPY web/package.json web/package-lock.json ./
RUN npm ci
COPY web/ ./
RUN npm run build

FROM golang:1.27 AS go
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
COPY --from=ui /src/internal/webui/dist internal/webui/dist
RUN CGO_ENABLED=0 go build -trimpath -o /ripper ./cmd/ripper

FROM debian:${DEBIAN} AS media
ARG MAKEMKV_VERSION
ARG CYANRIP_VERSION
ARG FFMPEG_VERSION
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential pkg-config wget git xz-utils ca-certificates gnupg dirmngr nasm meson ninja-build \
      libexpat1-dev libssl-dev zlib1g-dev libmp3lame-dev libopus-dev \
      libcdio-paranoia-dev libmusicbrainz5-dev libcurl4-openssl-dev \
      cdparanoia gddrescue eject sdparm tini \
 && rm -rf /var/lib/apt/lists/*
COPY scripts/build-media.sh /build.sh
RUN /build.sh "${FFMPEG_VERSION}" "${MAKEMKV_VERSION}" "${CYANRIP_VERSION}"
# collect from this stage: it already has every runtime library the binaries link against
COPY scripts/collect-rootfs.sh /collect.sh
RUN /collect.sh /rootfs /usr/bin/cdparanoia /usr/bin/ddrescue /usr/bin/eject /usr/bin/sdparm \
      /usr/bin/tini-static /usr/local/bin/makemkvcon /usr/local/bin/cyanrip

FROM gcr.io/distroless/cc-debian13:latest@sha256:<digest>
COPY --from=media /rootfs /
COPY --from=go /ripper /usr/local/bin/ripper
ENV HOME=/config LANG=C.UTF-8
VOLUME ["/config", "/out"]
EXPOSE 9090 9091
HEALTHCHECK --interval=30s --timeout=3s CMD ["ripper","healthcheck"]
ENTRYPOINT ["/usr/bin/tini-static","--","/usr/local/bin/ripper"]
CMD ["serve"]
```
- `scripts/build-media.sh <ffmpeg-version> <makemkv-version> <cyanrip-version>` (bash, `set -euo pipefail`; builder only; every `make` uses `-j"$(nproc)"`):
  1. **FFmpeg** (static, shared by MakeMKV and cyanrip):
     - Download `https://ffmpeg.org/releases/ffmpeg-<v>.tar.xz` and `.asc`, and verify the signature
       with the FFmpeg release signing key. Record its fingerprint in the script, taken from
       https://ffmpeg.org/download.html#releases; if you can't obtain it, **stop and ask**.
     - Configure exactly:
       ```
       ./configure --prefix=/opt/ffmpeg --enable-static --disable-shared --enable-pic \
         --disable-programs --disable-doc --disable-everything --disable-network --disable-autodetect \
         --enable-libmp3lame --enable-libopus --enable-protocol=file --enable-parser='*' \
         --enable-decoder='pcm*,flac,aac,ac3,eac3,dca,truehd,mlp,mp2,mp3,vorbis,opus,alac,mjpeg,png,webp,bmp' \
         --enable-encoder='pcm*,flac,libmp3lame,libopus,aac,alac' \
         --enable-muxer='flac,mp3,ogg,opus,adts,ipod,mp4,wav,image2' --enable-demuxer='image2' \
         --enable-filter='abuffer,abuffersink,aformat,aresample,anull,anullsink,ebur128,hdcd,aemphasis,volume'
       make -j"$(nproc)" install
       ```
       (`--enable-pic` because `libmakemkv` is a shared object; `anullsink` + `ebur128` for ReplayGain;
       `hdcd`/`aemphasis` for HDCD and pre-emphasis discs; `image2` + image decoders for cover art.)
  2. **MakeMKV:**
     - Download `makemkv-sha-<v>.txt` and verify it with GPG key `2ECF23305F1FC0B32001673394E3083A18042697`.
     - Download `makemkv-oss-<v>.tar.gz` and `makemkv-bin-<v>.tar.gz`, and check their sha256s.
     - oss: `PKG_CONFIG="pkg-config --static" PKG_CONFIG_PATH=/opt/ffmpeg/lib/pkgconfig ./configure --prefix=/usr/local --disable-gui && make -j"$(nproc)" && make install`.
     - bin: `mkdir -p tmp && touch tmp/eula_accepted && make && make install`.
     - If the bin tarball has no binary for the build architecture (arm64), **stop and ask**.
  3. **cyanrip:** `git clone --depth 1 --branch <cyanrip-version> https://github.com/cyanreg/cyanrip`, then
     `meson setup build --prefix=/usr/local --buildtype=release --prefer-static --pkg-config-path=/opt/ffmpeg/lib/pkgconfig && ninja -C build install`.
  4. **tini-static:** if `/usr/bin/tini-static` is missing after apt, download
     `https://github.com/krallin/tini/releases/download/v0.19.0/tini-static-<amd64|arm64>`, check its
     published sha256, and install it to `/usr/bin/tini-static` (mode 0755).

  This replaces `manual-build/install/install.sh` (and fixes its `$version` bug). There is no forum scraping.
- `scripts/collect-rootfs.sh <out> <binaries...>`:
  - copy each binary, and every `/usr/local/lib/*.so*`, with `cp --parents -L`;
  - copy the `ldd` closure of all of them (paths after `=>`), excluding only the glibc family and the
    C++ runtime that `cc-debian13` guarantees (`libc libm libdl libpthread librt libresolv ld-linux* libstdc++ libgcc_s`).
    Copy everything else, including `libssl`, `libcrypto`, `libz` and `libzstd`, so builder and runtime can't drift;
  - for each Debian package owning a copied file (`dpkg -S`), write `dpkg-query -s <pkg>` to
    `<out>/var/lib/dpkg/status.d/<pkg>`, so image scanners (Trivy) can see them.
- `.dockerignore`: `web/node_modules`, `bin`, `internal/webui/dist` (built in-image). Not `.git`.
- **Rule:** the final stage has **no `RUN`**. Every tool ripper execs is an ELF binary copied with its
  library closure. The image has no shell.
- Test (`task image`):
  - `docker run --rm ripper:dev version` works;
  - `docker run --rm --entrypoint /usr/local/bin/cyanrip ripper:dev -o help` lists flac, mp3, opus, aac and alac;
  - `docker run --rm --entrypoint /usr/local/bin/makemkvcon ripper:dev` prints its usage;
  - each copied tool runs `--version`/`-V` via `--entrypoint`;
  - Trivy finds the copied packages.

**P4.2 Delete legacy**
- Delete `root/` entirely (the MakeMKV profile moved in P2.3; `abcde.conf` is replaced by cyanrip).
- Delete `latest/`, `manual-build/` and `docker-compose.yml`; the compose file is replaced in P4.4.

**P4.3 Workflows**
- `ci.yml`: add the `image` job (build the Dockerfile for amd64, no push; Trivy per L§3; the P4.1 smoke tests).
- `release.yml`: add the `publish` job (L§5). Build natively per architecture (`ubuntu-latest` for
  amd64, `ubuntu-24.04-arm` for arm64; no QEMU, because the MakeMKV/ffmpeg build is slow under
  emulation), push by digest, then merge with `docker buildx imagetools create`.
- This PR's title: `feat!: replace the bash/python implementation with the Go ripper`, with a
  `BREAKING CHANGE:` footer linking the migration notes.

**P4.4 Docs**
- Rewrite the README with:
  - the config table (C§1.2);
  - the migration notes (main plan §8);
  - the API (`/api/v1/*`, `/docs`);
  - the known limitations (C§5.5; DVD/BD data discs are treated as video).
- New `docker-compose.yml`:
  ```yaml
  services:
    ripper:
      image: ghcr.io/jacaudi/docker-ripper:latest
      stop_grace_period: 35s
      restart: unless-stopped
      devices:                       # pass each drive's srN AND its sgN node; all are discovered automatically
        - /dev/sr0:/dev/sr0
        - /dev/sg0:/dev/sg0
      ports: ["9090:9090"]
      volumes: ["./config:/config", "./rips:/out"]
      environment:
        RIPPER_UID: "1000"
        RIPPER_GID: "1000"
        RIPPER_APPRISE_URLS: ""
  ```
- `deploy/k8s/ripper.yaml`: `replicas: 1`, `strategy: Recreate`, device-plugin resources for each
  drive's sr and sg nodes, `terminationGracePeriodSeconds: 35`, and on port 9091:
  - `startupProbe` `/startupz` (`failureThreshold: 30`, `periodSeconds: 5`);
  - `livenessProbe` `/livez`;
  - `readinessProbe` `/readyz`.
  
  Add a ServiceMonitor for `/metrics` and an alert example on `/healthz` (via blackbox) or on
  `ripper_scans_total{result="error"}`.

**P4.5 Hardware acceptance (owner)**
- With the built image: one rip each of a DVD, a BluRay (ideally with DTS-HD or TrueHD audio), an
  LPCM DVD, an audio CD and a data CD.
- For the audio CD: set your drive's offset in `RIPPER_AUDIO_DRIVE_OFFSETS` and check the tags, the
  cover, ReplayGain and the AccurateRip result in cyanrip's log. Rip once with `RIPPER_AUDIO_FORMATS=aac`.
  Also rip a CD that isn't in MusicBrainz (the retry rule).
- While one drive rips, confirm the per-tick `makemkvcon info` scan doesn't disturb it.
- Check that `makemkvcon reg` alone registers the key (`~/.MakeMKV/settings.conf` contains `app_Key`).
  If it doesn't, stop and ask.
- Cancel one rip with `docker stop` and confirm the partial output is gone.
- If two drives are available: rip a BluRay and an audio CD at the same time without configuring
  `RIPPER_DRIVES`; both must be discovered. Repeat with `RIPPER_MAX_PARALLEL_JOBS=1` and see the second queue.
- `curl :9091/livez`, `/startupz`, `/healthz` and `/readyz` all return the documented JSON.
- Scrape `:9091/metrics` and confirm the `ripper_` series. If you run an OTel collector, set
  `OTEL_EXPORTER_OTLP_ENDPOINT` and confirm the traces arrive.

Phase done when: CI is green and P4.5 is confirmed.

---

## Future (not planned work)

- **OTLP metrics and logs:** change only `internal/telemetry` (C§6.5), or bump go-service-kit once
  its `obs` supports them. No instrumentation changes are needed.

- Native ioctl backends: `CDROMEJECT`/`CDROM_LOCKDOOR` for eject, and
  `CDROM_DRIVE_STATUS`/`CDROM_DISC_STATUS` for state and audio-vs-data detection. They would replace
  `eject`, `sdparm` and `cdparanoia`.
- `golang.org/x/sys` has no constants for these; take them from `<linux/cdrom.h>`.
- Each one is a new backend package plus one patchbay case. Add a config switch only when two
  backends exist at the same time.
