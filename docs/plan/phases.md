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
| vitest / @testing-library/react / jsdom | 5.0.3 / 16.3.3 / latest compatible |
| typescript | 5.9.x (**not 7.x**: openapi-typescript and typescript-eslint need 5.x) |
| eslint / typescript-eslint / eslint-plugin-react-hooks / prettier | 10.12.0 / latest compatible |
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
- Done when: `task tools && task lint test vuln` exits 0, and `bin/golangci-lint version` prints `built with go1.27`.

**P0.2 Releases and commit conventions**
- Files: `release-please-config.json`, `.release-please-manifest.json`, `.github/workflows/release.yml`
  (the `release-please` job only), `.github/pull_request_template.md` (checklist: conventional title,
  `task lint test vuln` run (`task check` once `web/` exists), migration note if user-visible).
- Do: L§4, L§5.
- **(owner):** create the GitHub App and its secrets; set squash-only merging; allow Actions to
  create PRs (main plan §9).
- Done when: after merge, release-please opens a release PR.

**P0.3 fakebin**
- Files: `internal/testutil/fakebin/main.go`, `internal/testutil/fakebin/fakebin_test.go`,
  `internal/testutil/fakebintest/fakebintest.go`.
- Do: implement T§3 exactly.
- Test: one case per protocol row (per-call stdout/stderr/exit, each `creates` template, block/release,
  SIGTERM → 143 without creates).
- Done when: `go test ./internal/testutil/...` passes.

**P0.4 Synthetic fixtures**
- Files: every `.txt` listed in T§2.
- Done when: the files exist (they are used in P2.2).

**P0.5 CI and security workflows**
- Files: `.github/workflows/ci.yml` (jobs `pr-title`, `go`), `.github/workflows/security.yml`.
- Do: L§3, L§7. CodeQL `go` now; `javascript-typescript` is added in P3.3.
- Done when: the PR is green, and CodeQL/OSV/Scorecard results appear under Security → Code scanning.

---

## Phase 1 — Core

**P1.1 `internal/disc`**
- Do: types from C§2.1, including `String` and `MarshalText`.
- Test: the string of every value.
- Done when: `go test ./internal/disc` passes.

**P1.2 `internal/config`**
- Files: `config.go` (struct with every field from C§1.2), `validate.go`, `prefix.go` (`NormalizePrefix`), tests.
- Test: one row per validation rule in C§1.2; the prefix examples from C§1.3; a fully valid config
  gives no error; three bad values give one error that mentions all three.
- Done when: `go test ./internal/config` passes.

**P1.3 `internal/cli`: root, viper, version**
- Files: `cli.go` (`Execute(ctx) int`, C§5.3), `root.go`, `load.go`
  (`load(cmd *cobra.Command) (config.Config, error)`, C§1.1), `version.go` (C§5.7 plus the `version`
  command), `load_test.go`.
- Test (use `t.Setenv`):
  - defaults;
  - each `RIPPER_*` maps to its field;
  - **empty env → default**;
  - `--headless` overrides `RIPPER_HEADLESS=false`;
  - comma list for `RIPPER_APPRISE_URLS`;
  - durations;
  - invalid values → one joined error;
  - an unknown legacy name (`EJECTENABLED`) has no effect.
- Done when: `go test ./internal/cli` passes.

**P1.4 `cmd/ripper/main.go`**
- Do: `func main() { os.Exit(cli.Execute(context.Background())) }`.
- Done when: `go build ./cmd/ripper && ./ripper version` prints a version.

**P1.5 `internal/output`**
- Files: `planner.go` (C§3.5), `perm.go` (C§5.4), tests.
- Test:
  - sanitise: `"Movie: Part 1/2"` → `"Movie_ Part 1_2"`; `".."` and `""` → `disc_<ts>`;
  - Prepare path shape;
  - Finalize rename;
  - collision → `_<ts>`; a second collision → error;
  - CD multi-child move, with `.wav` excluded;
  - Cleanup;
  - CleanStaging;
  - modes for umask `002` and `022`: dirs, plain files, executable files;
  - chown is skipped when UID and GID are −1;
  - a label of `../../etc` stays inside the root.
- Done when: `task lint test` passes.

**P1.6 `internal/logring`**
- Do: C§2.3. Mutex-guarded; capacity 2000. `Write` copies each `\n`-terminated record (a partial
  trailing line is buffered until its newline) and never returns an error. `Lines(n)` returns
  newest first; `n > len` returns all.
- Test: wraparound; ordering; partial writes; concurrent writes under `-race`.
- Done when: `go test -race ./internal/logring` passes.

---

## Phase 2 — Engine and backends

**P2.1 `internal/runner` + `execrunner`**
- Do: C§2.2, C§5.1.
- Test (fakebin):
  - stdout and stderr lines are logged with the `tool` attr;
  - argv never appears in any log record;
  - exit code → `ExitError` via `errors.AsType`;
  - cancelling during `.block` returns within 6 s, and `syscall.Kill(-pgid, 0)` returns `ESRCH` afterwards;
  - `Output` is capped at 1 MiB.

**P2.2 `detect/makemkv`**
- Do: `ParseDRV` (C§3.1) and the `Detector` (C§3.2).
- Test: the T§2 expectation table; detector cases with fakebin: timeout (`.block` with a 1 s
  constructor timeout), the cdparanoia `audio` and `no_audio` paths, and makemkvcon exit 1.

**P2.3 Rip backends**
- Files: `rip/makemkv` (embed `default.mmcp.xml`, moved from `root/ripper/`), `rip/abcde`
  (embed `abcde.conf`, moved from `root/ripper/`), `rip/ddrescue`.
- Do: argv exactly as in C§3.6, and the override resolution in C§3.6.
- Test (fakebin):
  - `calls.jsonl` argv;
  - an override file wins over the embedded default;
  - the abcde per-rip conf contents (base + overrides) are asserted, and the file is deleted afterwards;
  - ddrescue passes both the `.iso` and the `.map`.

**P2.4 `eject/execeject`**
- Do: the C§3.6 sequence. Inject waits as `sleep func(context.Context, time.Duration) error`.
- Test: success; `eject.exit=1` → sdparm unlock, then eject; the last error is returned.

**P2.5 `notify/apprise` + `notify/nop`**
- Do: C§5.2, C§3.7.
- Test against `json://127.0.0.1:<httptest port>`: title, body and type; a timeout returns
  `context.DeadlineExceeded`; an invalid URL fails `New`.

**P2.6 `internal/makemkvkey`**
- Do: C§5.5 (`FetchBetaKey`) and `Register` (C§3.6).
- Test: an httptest page with a key → the key; a page without → error; 500 → error.
  `Register` runs `makemkvcon reg <key>` via fakebin, and the key appears in no log record.

**P2.7 `internal/engine`**
- Do: C§2.4, C§3.3, C§3.4. Fakes per T§1.
- Tests (all under synctest):
  - every cell of C§3.4;
  - repeated Empty/Open/Loading;
  - bad replies: Ready fails at 5, one Failure notification, recovery resets the count;
  - success → eject → awaiting removal → no re-rip while Inserted → cleared on Empty;
  - `Eject: false`, and a failed eject: both give no re-rip;
  - a rip failure: Cleanup called, Failure notification, eject, awaiting removal;
  - cancel mid-rip: Cleanup, Stopped notification via the detached ctx, no eject, `Run` returns nil;
  - `Status()` transitions.

**P2.8 Patchbay + `serve` (headless engine, admin only)**
- Files: `internal/patchbay/backends.go`, `internal/patchbay/spec.go`, `internal/cli/serve.go`,
  `internal/cli/healthcheck.go`.
- Do:
  - `func Backends(cfg config.Config, run runner.Runner, logger *slog.Logger) (engine.Deps, error)`:
    build every backend from C§2.3; `notify/nop` when `AppriseURLs` is empty.
  - `func Spec(cfg config.Config, eng *engine.Engine, p *obs.Providers, ring *logring.Ring) (lifecycle.Spec, error)`
    with:
    - the admin listener and readiness (C§4.2);
    - `PropagationDelay: lifecycle.NoPropagationDelay` (comment: single replica + Recreate);
    - `DrainTimeout: 5*time.Second`;
    - `Flush: p.Shutdown`, `Logger`, `MeterProvider`;
    - the engine as `lifecycle.Worker{Name: "engine", Run: eng.Run, FinishCurrentCycle: true, StopTimeout: 15*time.Second}`.
      The kit cancels the context **immediately** in both modes; `FinishCurrentCycle` only means
      "wait up to 15 s for the cleanup to finish".
  - `serve` follows C§3.8.
  - `healthcheck`: GET `http://127.0.0.1:<port of RIPPER_ADMIN_ADDR>/healthz`, 2 s timeout, exit 1 on non-200.
- Test: a patchbay test with ephemeral listeners: `/healthz` 200; `/readyz` 503 when the drive
  path is missing.
- Done when: `task lint test vuln` passes.

**P2.9 `ripper detect`**
- Do: C§5.6.
- Test: JSON output for the bluray fixture via fakebin; exit 0 for empty; exit 1 on makemkvcon exit 1.
- **(owner):** run `ripper detect --raw` with a DVD, a BluRay, an audio CD, a data CD, an empty
  drive and an open tray. Paste the output into the T§2 fixtures and fix the expectation table if
  the hardware disagrees.

Phase done when: `task lint test vuln` passes (`task check` needs `web/`, which arrives in P3.3).

---

## Phase 3 — HTTP and UI

**P3.1 `internal/api`**
- Files: `api.go` (`Register(h huma.API, prefix string, st StatusSource, logs LogSource)`, with
  `type StatusSource interface{ Status() engine.Status }` and `type LogSource interface{ Lines(n int) []string }`),
  `auth.go` (C§4.3), tests, `testdata/openapi.json` (golden).
- Test (`humatest`):
  - status 200;
  - log 200 with N lines, newest first; `lines=0` → 422; `lines=2001` → 422;
  - auth: no creds → 401 with header; wrong → 401; right → 200; disabled when either is empty.
  - Golden: `OpenAPI().MarshalJSON()` with prefix `""` equals the golden file; `-update` rewrites it.

**P3.2 `internal/apidocs` (Scalar)**
- Files: `scripts/vendor-scalar.sh`, `internal/apidocs/assets/{docs.html,docs-init.js,standalone.js,SHA256}`, `apidocs.go`, tests.
- `vendor-scalar.sh <version>`:
  - download `https://registry.npmjs.org/@scalar/api-reference/-/api-reference-<version>.tgz`;
  - verify its sha512 against `dist.integrity` from `https://registry.npmjs.org/@scalar/api-reference/<version>`;
  - extract `package/dist/browser/standalone.js`;
  - write the file's sha256 to `SHA256`.
- `Mount(api *httpapi.API, prefix string)` registers the docs and OpenAPI routes from C§4.1, with
  the CSP from C§4.4.
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
  - `Layout` with a Header titled "Ripper", and Content holding `StatusCard` above `LogPanel`.
- `api/client.ts`: `createClient<paths>({ baseUrl: '.' })` (openapi-fetch).
- `hooks/usePoll.ts`: `usePoll(fn, ms)` runs at mount and then every `ms` via `setInterval`, skips
  ticks while `document.hidden`, and clears on unmount.
- `components/StatusCard.tsx`:
  - polls `GET /api/v1/status` every 5000 ms;
  - AntD `Card` + `Descriptions`: state, disc kind, label, device, started at + live elapsed,
    bad responses, last result (outcome, error, paths);
  - state `Tag` colours: idle `default`, detecting `blue`, ripping `processing`, ejecting `cyan`,
    awaiting_removal `gold`, stopped `red`;
  - an `Alert` (error) when `bad_responses >= 5`.
- `components/LogPanel.tsx`:
  - polls `GET /api/v1/log?lines=200` every 10000 ms;
  - AntD `Table` with `size="small"`, `pagination={false}`, `scroll={{ y: 600 }}`, columns
    time / level / tool / message;
  - each line is `JSON.parse`d into `{time, level, tool, msg, line}`; message = `line ?? msg`;
  - unparseable lines render as `{msg: raw}` in monospace.
- No router, no login form (the browser handles basic auth), no clear button.
- Test (vitest + jsdom + testing-library, `fetch` mocked with `vi.fn`): StatusCard renders a tag
  for each state and the bad-drive alert; LogPanel renders a parsed row and a raw row.
- Done when: `task ui:lint ui:test ui:build` passes and `internal/webui/dist/index.html` exists.

**P3.5 `internal/webui`**
- Files: `webui.go` (`//go:embed all:dist`; `Mount(api *httpapi.API, prefix string) error`), tests.
- Do: the C§4.1 routes marked "!headless", with the CSP + nonce from C§4.4. `Mount` returns an
  error when `dist/index.html` is missing, so `serve` without `--headless` fails loudly.
- Test:
  - `/` 200, and the CSP `nonce-` matches the meta tag;
  - an asset has immutable caching;
  - prefix `/ripper` works, and `/ripper` → 301 `/ripper/`;
  - a missing dist returns an error.

**P3.6 Wire HTTP into the patchbay**
- Do: in `Spec`, build the API listener with `httpapi.New` (C§4.1 options) and call `api.Register`
  (StatusSource = engine, LogSource = ring) and `apidocs.Mount`; call `webui.Mount` unless headless;
  add the root redirects. Add the API server to `Servers`.
- Test: hit every route in both modes.

**P3.7 End-to-end smoke**
- Do: T§4, every scenario.
- Phase done when: `task check` passes.

---

## Phase 4 — Cutover

**P4.1 Dockerfiles** — replace both:
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
      abcde ca-certificates ccextractor cdparanoia eject eyed3 flac gddrescue glyrc id3 id3v2 \
      lame mkcue sdparm speex tini vorbis-tools vorbisgain \
 && rm -rf /var/lib/apt/lists/*
COPY --from=go /ripper /usr/local/bin/ripper
EXPOSE 9090 9091
HEALTHCHECK --interval=30s --timeout=3s CMD ["ripper","healthcheck"]
ENTRYPOINT ["tini","--","ripper"]
CMD ["serve"]
```

**P4.2 Delete legacy**
- Delete `root/` entirely; the defaults were moved in P2.3.
- Delete `docker-compose.yml`; it is replaced in P4.4.

**P4.3 Workflows**
- `ci.yml`: add the `image` job (build both Dockerfiles, no push, Trivy per L§3, smoke test
  `docker run --rm <image> version`).
- `release.yml`: add the `publish` job (L§5).
- Add `rebuild.yml` (L§5).
- Delete `BuildImages.yml`, `IssueModerator.yml`, `LabelSponsors.yml`, `UpdateOnBaseImageChange.yml`
  and `ManualBuildOnBetaRelease.yml`.
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
      init: true
      stop_grace_period: 30s
      restart: unless-stopped
      devices: ["/dev/sr0:/dev/sr0", "/dev/sg0:/dev/sg0"]
      ports: ["9090:9090"]
      volumes: ["./config:/config", "./rips:/out"]
      environment:
        RIPPER_UID: "1000"
        RIPPER_GID: "1000"
        RIPPER_APPRISE_URLS: ""
  ```
- `deploy/k8s/ripper.yaml`: `replicas: 1`, `strategy: Recreate`, a device-plugin resource,
  `terminationGracePeriodSeconds: 30`, liveness `/healthz` and readiness `/readyz` on 9091.

**P4.5 Hardware acceptance (owner)**
- With the built image: one rip each of a DVD, a BluRay, an audio CD and a data CD.
- Check that `makemkvcon reg` alone registers the key (`~/.MakeMKV/settings.conf` contains `app_Key`).
  If it doesn't, stop and ask.
- Cancel one rip with `docker stop` and confirm the partial output is gone.

Phase done when: CI is green and P4.5 is confirmed.

---

## Future (not planned work)

- Native ioctl backends: `CDROMEJECT`/`CDROM_LOCKDOOR` for eject, and
  `CDROM_DRIVE_STATUS`/`CDROM_DISC_STATUS` for state and audio-vs-data detection. They would replace
  `eject`, `sdparm` and `cdparanoia`.
- `golang.org/x/sys` has no constants for these; take them from `<linux/cdrom.h>`.
- Each one is a new backend package plus one patchbay case. Add a config switch only when two
  backends exist at the same time.
