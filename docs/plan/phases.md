# Phases — executable task list

## How to execute

- Do tasks **in order**. Each task lists **Files**, **Do**, **Test**, and **Done when**.
- A task is done only when its "Done when" command exits 0. Commit after every task with the
  message `P<phase>.<n>: <task title>`.
- Contracts live in [`contracts.md`](contracts.md) (cited as C§n) and tests in
  [`testing.md`](testing.md) (cited as T§n). Copy names and values from there exactly.
- If a task needs a decision that isn't written down, **stop and ask**; do not guess.
- One PR per phase. CI must be green before the next phase starts.

Pinned versions (Renovate keeps them current afterwards):

| Thing | Version |
|---|---|
| Go | `go 1.27.0` + `toolchain go1.27.2` in go.mod; `golang:1.27` image |
| Node | 24 LTS (`node:24-alpine`); `engines.node: ">=24"` |
| cobra / viper | v1.10.2 / v1.21.0 |
| go-service-kit | v0.3.0 |
| apprise-go | v0.3.3 (exact) |
| react / react-dom | 19.3.0 |
| antd | 6.6.5 |
| vite / @vitejs/plugin-react | 8.3.4 / 6.1.2 |
| vitest / @testing-library/react / jsdom | 5.0.3 / 16.3.3 / latest compatible |
| typescript | 5.9.x (**not 7.x**: openapi-typescript and typescript-eslint need 5.x) |
| eslint / typescript-eslint / eslint-plugin-react-hooks | 10.12.0 / latest compatible |
| openapi-typescript / openapi-fetch | 7.13.0 / 0.17.0 |
| @scalar/api-reference | 1.73.1 |

---

## Phase 0 — Safety net

**P0.1 Go module and tooling**
- Files: `go.mod`, `Makefile`, `.golangci.yml`, `renovate.json`, `.gitignore` (append), `.dockerignore`.
- Do:
  - `go mod init github.com/jacaudi/docker-ripper`; set `go 1.27.0` and `toolchain go1.27.2`.
  - Copy `.golangci.yml` and the Makefile structure from go-service-kit (pinned tool installs
    through `scripts/retry.sh`, verdicts never retried). Copy `scripts/retry.sh` too.
  - Targets: `help lint test vulncheck modernize parity ui-deps ui-gen ui-lint ui-test ui vendor-scalar build image`.
    `modernize` = `go fix -diff ./...` and fails if the output isn't empty. `test` =
    `go test -race -shuffle=on ./...`. `build` depends on `ui`.
  - **Check that the pinned golangci-lint release is built with Go ≥ 1.27** (`golangci-lint version`).
    If it isn't, pin the newest release that is.
  - `.gitignore`: `internal/webui/dist/*`, `!internal/webui/dist/.gitkeep`, `web/node_modules/`, `bin/`.
  - `.dockerignore`: `web/node_modules`, `bin`. Do **not** ignore `.git` (C§6.2).
- Done when: `make lint test` exits 0 (nothing to test yet).

**P0.2 Synthetic DRV fixtures**
- Files: `internal/disc/testdata/drv/<case>.{txt,json}` for every required case in T§2.
- Do: write them per T§2, with `source: "synthetic"`.
- Done when: `ls internal/disc/testdata/drv/*.json | wc -l` ≥ 13.

**P0.3 Public fixtures**
- Do: search `github.com/rix1337/docker-ripper/issues` and the MakeMKV forum for pasted `DRV:` lines.
  Save each as `<case>_public_<n>` with `source: "public"` and `source_url`. Skip anything you can't attribute.
- Done when: the files are committed (zero found is acceptable; say so in the PR).

**P0.4 Hardware capture script**
- Files: `scripts/capture-fixtures.sh` (bash, `set -euo pipefail`).
- Do: arg 1 = case name. Env: `DRIVE` (default `/dev/sr0`), `SG` (default `/dev/sg0`), `IMAGE`
  (default `rix1337/docker-ripper:latest`, the legacy image). Runs
  `docker run --rm --device "$DRIVE" --device "$SG" --entrypoint makemkvcon "$IMAGE" -r --cache=1 info disc:9999 > <case>.txt`
  and `docker run --rm --device "$DRIVE" --device "$SG" --entrypoint cdparanoia "$IMAGE" -d "$DRIVE" -Q > <case>.cdparanoia.txt 2>&1 || true`,
  then prints: "copy <case>.txt to internal/disc/testdata/drv/, write <case>.json with source=hardware".
- Done when: `bash -n scripts/capture-fixtures.sh` and `shellcheck` pass.

**P0.5 fakebin**
- Files: `internal/testutil/fakebin/main.go`, `internal/testutil/fakebin/fakebin_test.go`,
  `internal/testutil/fakebintest/fakebintest.go` (helper: `Install(t) (binDir string)` builds
  and symlinks; `Scenario(t) (dir string)`; `Calls(t, dir) []Call`).
- Do: implement T§3 exactly.
- Test: each protocol row (stdout per call, exit code, creates templates, block/release, SIGTERM → 143, sleep.max).
- Done when: `go test ./internal/testutil/...` passes.

**P0.6 Parity harness (legacy only)**
- Files: `test/parity/Dockerfile`, `test/parity/legacy/ripper.sh` + `SHA256`,
  `test/parity/harness.go`, `test/parity/scenarios.go`, `test/parity/parity_test.go` (`//go:build parity`).
- Do: implement T§4 with the legacy run only. Get `ripper.sh` with
  `git show archive:root/ripper/ripper.sh`. Scenarios: every T§4.1 row except `cancel_mid_rip`.
  The assertions encode the **legacy** column.
- Done when: `make parity` passes.

**P0.7 CI**
- Files: `.github/workflows/ci.yml`.
- Do: on `pull_request` and push to `main`: job `go` (`actions/setup-go` with
  `go-version-file: go.mod`; `make lint test vulncheck modernize`); job `parity` (`make parity`).
  `permissions: contents: read`.
- Done when: the PR is green.

---

## Phase 1 — Skeleton that boots

**P1.1 `internal/buildinfo`** — `Version() string` per C§6.2. Test: returns non-empty.

**P1.2 `internal/disc`**
- Files: `disc.go` (types C§2.1), `drv.go` (`ParseDRV` C§3.1), `drv_test.go`.
- Test: loop over `testdata/drv/*.json`; `t.Attr("fixture_source", …)`; compare state/kind/label/index/err.
- Done when: `go test ./internal/disc` passes for every fixture.

**P1.3 `internal/config`**
- Files: `config.go` (struct C§1.2, `Settings()` helpers), `validate.go` (`Validate() error` covering every rule in C§1.2 and C§1.3), `prefix.go` (`NormalizePrefix`), tests.
- Test: one row per validation rule; prefix examples from C§1.3.
- Done when: `go test ./internal/config` passes.

**P1.4 `internal/cli` root + viper**
- Files: `cli.go` (`Execute(ctx) int`, C§5.3), `root.go`, `load.go` (`load(cmd) (config.Config, error)` per C§1.1), `version.go`, `load_test.go`.
- Test (use `t.Setenv`): defaults; each env name maps to its field; **empty env → default**;
  `--headless` overrides `HEADLESS=false`; comma list for `APPRISE_URLS`; durations; invalid → joined errors.
- Done when: `go test ./internal/cli` passes.

**P1.5 `cmd/ripper/main.go`** — `func main() { os.Exit(cli.Execute(context.Background())) }`.
Done when: `go build ./cmd/ripper && ./ripper version`.

**P1.6 `serve` skeleton**
- Files: `internal/cli/serve.go`, `internal/patchbay/spec.go`.
- Do: C§3.8 steps 1–3, then `lifecycle.Run` with the admin listener only (C§4.2),
  `PropagationDelay: lifecycle.NoPropagationDelay` (with comment: single replica + Recreate),
  `DrainTimeout: 5*time.Second`, `Logger`, `MeterProvider`, `Flush: p.Shutdown`.
- Files: `internal/cli/healthcheck.go`: GET `http://127.0.0.1:<port of ADMIN_ADDR>/healthz`
  (host defaults to 127.0.0.1 when empty), timeout 2 s, exit 1 on non-200.
- Test: `internal/patchbay` test runs Spec with ephemeral listeners and asserts `/healthz` is 200
  and `/readyz` reflects the drive check (point `DRIVE` at a temp file).
- Done when: `ripper serve` started locally answers `ripper healthcheck` with exit 0 and exits 0 on SIGTERM.

**P1.7 `internal/output`**
- Files: `mode.go` (`ParseMode` C§5.4), `planner.go` (C§3.5), tests.
- Test: mode table (C§5.4); sanitise table (`"Movie: Part 1/2"` → `"Movie_ Part 1_2"`, `".."` → `disc_<ts>`,
  `""` → `disc_<ts>`); collision suffixes; staging for AudioCD; `Finalize` with/without separate-finish;
  traversal attempt `../../etc` stays inside the root.
- Done when: `make test modernize lint` passes.

---

## Phase 2 — HTTP surface and UI

**P2.1 `internal/api`**
- Files: `api.go` (`Register(h huma.API, prefix string, st StatusSource, logFile string)` where
  `type StatusSource interface{ Status() engine.Status }`), `log.go` (tail C§4.1), tests,
  `testdata/openapi.json` (golden).
- Test: `humatest` for each route in C§4.1 (status 200; log 200 with N lines newest-first;
  `lines=0` → 422; delete → 204 and the file is empty; missing file → empty list).
  Golden test: `api.Huma.OpenAPI().MarshalJSON()` with prefix `""` equals the golden; `-update` rewrites it.
- Done when: `go test ./internal/api` passes.

**P2.2 Auth + CSRF middleware** — `internal/httpmw/basicauth.go`, `basicauth_test.go` (C§4.3);
`http.NewCrossOriginProtection()` wired in the patchbay. Test: no creds → 401 with header;
wrong → 401; right → 200; disabled when either is empty. Done when: tests pass.

**P2.3 `internal/apidocs` (Scalar)**
- Files: `scripts/vendor-scalar.sh`, `internal/apidocs/assets/{docs.html,docs-init.js,standalone.js,SHA256}`, `apidocs.go`, tests.
- Do: `vendor-scalar.sh <version>` downloads `https://registry.npmjs.org/@scalar/api-reference/-/api-reference-<version>.tgz`,
  verifies the tarball's sha512 against `dist.integrity` from `https://registry.npmjs.org/@scalar/api-reference/<version>`,
  extracts `package/dist/browser/standalone.js`, and writes its sha256 to `SHA256`. `Mount(api *httpapi.API, prefix string)`
  registers the docs routes from C§4.1 with the CSP in C§4.4.
- Test: `/docs` 200 with exact CSP; asset 200; `/openapi.json` 200 valid JSON; a test checks
  `standalone.js` against `SHA256`.
- Done when: tests pass.

**P2.4 Frontend scaffold** (`web/`)
- Files: `web/package.json` (exact pins above; scripts: `dev`, `build`, `gen`, `lint`, `typecheck`, `test`),
  `web/package-lock.json`, `web/vite.config.ts` (`base: './'`, `build.outDir: '../internal/webui/dist'`,
  `build.emptyOutDir: true`, `build.modulePreload.polyfill: false`), `web/tsconfig.json` (strict),
  `web/eslint.config.js` (flat; typescript-eslint + react-hooks), `web/index.html` (contains
  `<meta name="csp-nonce" content="__CSP_NONCE__">`, `<div id="root">`), `internal/webui/dist/.gitkeep`.
- Do: `"gen": "openapi-typescript ../internal/api/testdata/openapi.json -o src/api/schema.d.ts"`; commit `schema.d.ts`.
- Done when: `make ui-deps ui-gen ui-lint` passes and `git diff --exit-code web/src/api/schema.d.ts` is clean.

**P2.5 Frontend app**
- Files (`web/src/`):
  - `main.tsx`: read the nonce from the meta tag; render `<App nonce={…}/>`.
  - `App.tsx`: `ConfigProvider` with `csp={{ nonce }}` and `theme={{ algorithm: theme.darkAlgorithm }}`;
    `Layout` with a Header titled "Ripper" and a Content holding `StatusCard` and `LogPanel`.
  - `api/client.ts`: `createClient<paths>({ baseUrl: '.' })` (openapi-fetch).
  - `hooks/usePoll.ts`: `usePoll(fn, ms)` runs at mount and then every `ms` with `setInterval`,
    skips ticks while `document.hidden`, and clears on unmount.
  - `components/StatusCard.tsx`: polls `GET /api/v1/status` every 5000 ms. AntD `Card` +
    `Descriptions` (state, disc kind, label, device, started at + live elapsed, last result outcome/error/dir).
    State `Tag` colours: idle `default`, detecting `blue`, ripping `processing`, ejecting `cyan`,
    waiting_for_eject `gold`, stopped `red`.
  - `components/LogPanel.tsx`: polls `GET /api/v1/log?lines=200` every 10000 ms. AntD `Table`
    `size="small"`, `pagination={false}`, `scroll={{ y: 600 }}`, columns time / level / tool / message.
    Each line is `JSON.parse`d into `{time, level, tool, msg, line}`. Message = `line ?? msg`.
    Unparseable lines render as `{msg: raw}` in monospace. An `Alert` (warning) shows when `large`.
  - `components/ClearLogButton.tsx`: `Popconfirm` "Clear the log?" → `DELETE /api/v1/log` → refetch the log.
    On error `message.error("Could not clear log")`.
  - No router; no login form (the browser handles basic auth).
- Test (vitest + jsdom + testing-library, `fetch` mocked with `vi.fn`): StatusCard renders a tag
  for each state; LogPanel renders a parsed row and a raw row and the large alert; ClearLogButton
  sends DELETE and then GET.
- Done when: `make ui-lint ui-test ui` passes and `internal/webui/dist/index.html` exists.

**P2.6 `internal/webui`**
- Files: `webui.go` (`//go:embed all:dist`; `Mount(api *httpapi.API, prefix string) error`), tests.
- Do: routes from C§4.1 marked "!headless", CSP + nonce from C§4.4. `Mount` returns an error if
  `dist/index.html` is missing (so `serve` without `--headless` fails loudly).
- Test: `/` 200, CSP contains `nonce-` matching the meta tag; an asset has immutable caching;
  prefix `/ripper` works and `/ripper` → 301 `/ripper/`; a missing dist returns an error.
- Done when: `make ui test` passes.

**P2.7 Wire HTTP into the patchbay** — the API listener with `httpapi.New` options from C§4.1,
`api.Register`, `apidocs.Mount` if docs, `webui.Mount` unless headless, root redirects. The engine is
still absent: StatusSource returns `{state: "idle"}`. Test: patchbay test hits each route in both
modes. Done when: `make lint test modernize` passes.

---

## Phase 3 — Engine and backends

**P3.1 `internal/runner` + `execrunner`** — C§2.2 and C§5.1. Integration tests with fakebin:
stdout and stderr lines logged with the `tool` attr; exit code → `ExitError` via `errors.AsType`;
cancel during `makemkvcon.block` → returns within 6 s and the process group is gone (`kill -0` fails);
`Output` caps at 1 MiB.

**P3.2 `detect/makemkv`** — C§3.2. Fakebin tests per fixture, including timeout (`makemkvcon.block` +
`DETECT_TIMEOUT=1s`) and the cdparanoia fallback cases.

**P3.3 `rip/makemkv`, `rip/ddrescue`, `rip/abcde`** — argv exactly per C§3.6. abcde: the per-rip
conf is rendered and deleted afterwards; its contents are asserted (base + overrides). Fakebin tests
check `calls.jsonl`.

**P3.4 `eject/execeject`** — C§3.6 fallback sequence. Tests: success; `eject.exit=1` → sdparm
unlock + eject; inject the waits as `sleep func(context.Context, time.Duration) error` so tests run fast.

**P3.5 `notify/apprise`, `notify/nop`** — C§5.2 and C§3.7. Test against
`json://127.0.0.1:<httptest port>`: title/body/type; timeout returns `context.DeadlineExceeded`;
invalid URL fails `New`.

**P3.6 `mkvkey/env`** — returns the key. (`mkvkey/forum` comes in P4.1.)

**P3.7 `internal/engine`** — C§2.4, C§3.3, C§3.4. Fakes in `internal/engine/enginetest/fakes.go`
(scripted detector: a slice of `{Disc, error}` results; recording ripper/ejector/notifier; a ripper
that blocks until ctx is done). Every test uses synctest (T§1). One test per row of C§3.4 and per
branch of C§3.3: bad threshold, garbage once, empty/open/loading, success, failure (cleanup + Failure
notify + eject), cancel mid-rip (cleanup, Stopped notify via the detached ctx, no eject, Run returns
nil), manual eject wait (stops on Empty), `Status()` transitions.

**P3.8 Patchbay backends** — `internal/patchbay/backends.go`:
`func Backends(cfg config.Config, run runner.Runner, logger *slog.Logger) (engine.Deps, error)`,
with `newDetector` (switch `DetectorBackend`), `newEjector` (switch `EjectBackend`), and `newNotifier`
(nop when `AppriseURLs` is empty). An unknown backend name → error naming the env var. Add the engine
as a `lifecycle.Worker{Name: "engine", Run: eng.Run, FinishCurrentCycle: true, StopTimeout: 15*time.Second}`
in `Spec`. StatusSource = the engine.

**P3.9 Parity, Go side** — extend the harness with the Go run (T§4) and add `cancel_mid_rip`.
Done when: `make parity` passes with every deviation declared.

Phase done when: `make lint test vulncheck modernize parity` passes.

---

## Phase 4 — Startup work

**P4.1 `mkvkey/forum`** — `forum.New(d outbound.Doer, url string) *Source`. GET `url`; regex
`T-[\w@]{66}`; first match; no match → error `mkvkey: no beta key found`. The patchbay builds the
client and passes `forum.DefaultURL` (`https://forum.makemkv.com/forum/viewtopic.php?f=5&t=1053`):
```go
// MakeMKV forum: public page, fetched once per process start; no API, no published quota.
// One attempt, 1 rps: a failed fetch falls back to the existing settings.conf key.
c, err := outbound.New(outbound.Config{Product: "ripper", Version: buildinfo.Version(),
	ContactURL: "https://github.com/jacaudi/docker-ripper", Timeout: 15 * time.Second,
	RequestsPerSecond: 1, Burst: 1, MaxAttempts: 1})
```
Tests pass `httptest.NewServer(...).Client()` as the `Doer` and the server URL: page with a key
→ key; page without → error; 500 → error.

**P4.2 MakeMKV settings + registration** — C§3.8 steps 4–6, in `internal/makemkv/settings.go`
(`UpdateKey(dir, key string) error`). Tests: create; replace; preserve other lines; modes 0700/0600.

**P4.3 Embedded defaults** — `internal/assets/assets.go` embeds `default.mmcp.xml` and `abcde.conf`
(moved from `root/ripper/`, with `EJECTCD=y` left as is; overridden per rip). Resolution per C§3.6.
Test: override file wins; the default is written once.

**P4.4 Removed-variable warnings** — C§1.2 list. Test: a WARN for each set var; none when unset.

Phase done when: `make lint test vulncheck modernize parity` passes.

---

## Phase 5 — Image cutover

**P5.1 Dockerfiles** — replace both:
```dockerfile
# syntax=docker/dockerfile:1
FROM node:24-alpine AS ui
WORKDIR /src/web
COPY web/package.json web/package-lock.json ./
RUN npm ci
COPY web/ ./
COPY internal/api/testdata/openapi.json /src/internal/api/testdata/openapi.json
RUN npm run build

FROM golang:1.27 AS go
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
COPY --from=ui /src/internal/webui/dist internal/webui/dist
RUN CGO_ENABLED=0 go build -trimpath -o /ripper ./cmd/ripper

FROM ubuntu:noble
ENV DEBIAN_FRONTEND=noninteractive LANG=C.UTF-8 HOME=/root
# latest/: add ppa:heyarje/makemkv-beta and install makemkv-bin makemkv-oss
# manual-build/: keep the MakeMKV-from-source stage and copy /usr/local from it
RUN apt-get update && apt-get install -y --no-install-recommends \
      abcde ca-certificates cdparanoia eject eyed3 flac gddrescue glyrc id3 id3v2 lame \
      mkcue sdparm speex tini vorbis-tools vorbisgain ccextractor \
 && rm -rf /var/lib/apt/lists/*
COPY --from=go /ripper /usr/local/bin/ripper
EXPOSE 9090 9091
HEALTHCHECK --interval=30s --timeout=3s CMD ["ripper","healthcheck"]
ENTRYPOINT ["tini","--","ripper"]
CMD ["serve"]
```

**P5.2 Delete legacy** — `root/etc/my_init.d/`, `root/etc/syslog-ng/`, `root/web/`,
`root/ripper/ripper.sh`, `root/ripper/settings.conf` (`default.mmcp.xml` and `abcde.conf` were
moved in P4.3).

**P5.3 Workflows**
- `ci.yml` adds jobs `ui` (setup-node 24, `cache: npm`, `cache-dependency-path: web/package-lock.json`;
  `make ui-deps ui-gen ui-lint ui-test` + `git diff --exit-code web/src/api/schema.d.ts`) and
  `image` (build both Dockerfiles, no push, smoke test: `docker run --rm image version`).
- New `publish.yml`: on push to `main` and tags `v*`; `permissions: contents: read, packages: write`;
  `docker/login-action` to `ghcr.io` with `GITHUB_TOKEN`; buildx multi-arch (amd64+arm64 for
  `latest/`, amd64 for `manual-build/`); tags `latest`, `ppa-latest`, `manual-latest`, `sha-<short>`,
  semver on tags; GHA cache.
- Delete `BuildImages.yml`, `IssueModerator.yml`, `LabelSponsors.yml`, `UpdateOnBaseImageChange.yml`
  and `ManualBuildOnBetaRelease.yml`. Renovate handles base images; a scheduled `publish.yml`
  run (weekly) rebuilds for new MakeMKV.

**P5.4 Docs** — README rewrite: config table (C§1.2), the removed-vars list, migration notes
(main plan §8), compose:
```yaml
services:
  ripper:
    image: ghcr.io/jacaudi/docker-ripper:latest
    init: true
    stop_grace_period: 30s
    devices: ["/dev/sr0:/dev/sr0", "/dev/sg0:/dev/sg0"]
    ports: ["9090:9090"]
    volumes: ["./config:/config", "./rips:/out"]
    environment:
      APPRISE_URLS: ""
    restart: unless-stopped
```
and a k8s example (`replicas: 1`, `strategy: Recreate`, a device-plugin resource,
`terminationGracePeriodSeconds: 30`, liveness `/healthz` and readiness `/readyz` on 9091).

Phase done when: CI is green, the image smoke test passes, and **you** confirm one real rip per disc
type on hardware.

---

## Phase 6 — Native backends (optional, hardware-gated)

**P6.1 `internal/linuxcdrom`** — constants (x/sys has none): `CDROMEJECT=0x5309`,
`CDROMCLOSETRAY=0x5319`, `CDROM_DRIVE_STATUS=0x5326`, `CDROM_DISC_STATUS=0x5327`,
`CDROM_LOCKDOOR=0x5329`; `CDS_NO_DISC=1`, `CDS_TRAY_OPEN=2`, `CDS_DRIVE_NOT_READY=3`,
`CDS_DISC_OK=4`; `CDS_AUDIO=100`, `CDS_DATA_1=101`, `CDS_DATA_2=102`, `CDS_XA_2_1=103`,
`CDS_XA_2_2=104`, `CDS_MIXED=105`; `CDSL_CURRENT = math.MaxInt32`. Open with
`unix.Open(dev, unix.O_RDONLY|unix.O_NONBLOCK, 0)`. Files `*_linux.go`.

**P6.2 `eject/ioctleject`** — `unix.IoctlSetInt(fd, CDROM_LOCKDOOR, 0)` then
`unix.IoctlSetInt(fd, CDROMEJECT, 0)`. Add the `ioctl` case to the patchbay. Hardware test required.

**P6.3 `detect/native`** — drive status via
`unix.Syscall(unix.SYS_IOCTL, fd, CDROM_DRIVE_STATUS, CDSL_CURRENT)`:
NO_DISC → Empty, TRAY_OPEN → Open, DRIVE_NOT_READY → Loading, DISC_OK → `CDROM_DISC_STATUS`:
AUDIO → AudioCD (label `""`); DATA_*/XA_*/MIXED → call `detect/makemkv` for Kind and Label
(it distinguishes DVD/BD), and if that reports CD → Data. Add the `native` case. Hardware test required.

**P6.4** Once both have run on hardware for a release cycle, delete the exec backends and their
config values (YAGNI), with a deviation note.
