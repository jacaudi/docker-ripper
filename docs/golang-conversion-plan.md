# Go conversion plan

Status: **agreed, v3 (executable)** · Baseline: `archive` branch (= `main` @ `21c5796`) ·
Background: [`.claude/CLAUDE.md`](../.claude/CLAUDE.md)

## How to use this plan

This plan is written so that an executor (human or a smaller model) can carry it out **without
making design decisions**.

| File | Contains | Use it to |
|---|---|---|
| this file | goal, decisions, glossary, rules, deployment, behaviour changes | understand *why* and what's forbidden |
| [`plan/contracts.md`](plan/contracts.md) | config table, exact Go types/interfaces, behaviour tables, HTTP routes, CSP, primitives | copy names, signatures and values |
| [`plan/testing.md`](plan/testing.md) | test layers, fixture format, fakebin protocol, parity scenarios | write tests |
| [`plan/tooling.md`](plan/tooling.md) | taskfile, lint/static-analysis config, security scanning, Conventional Commits, release-please, Renovate, workflows | set up and run quality gates and releases |
| [`plan/phases.md`](plan/phases.md) | numbered tasks with files, steps and "done when" commands | do the work, in order |

When something you need is not specified, **stop and ask**. Do not invent behaviour.

## 1. Goal

Replace the bash ripper, the init script and the Python web UI with one Go binary, `ripper`,
that runs as a **headless API service** (`ripper serve --headless`) or as **API + web UI**
(`ripper serve`).
- External tools that can't reasonably be replaced (MakeMKV, abcde, ddrescue) sit behind seams
  and are tested against fake binaries.
- Everything else becomes Go or a library.
- Known legacy bugs are fixed on purpose and listed in §8.

This fork **diverges from upstream permanently**.

Out of scope: multi-drive support, new output formats, re-implementing abcde's audio pipeline.

## 2. Decisions log

| # | Topic | Decision |
|---|---|---|
| 1 | Seam / Patchbay | One interface per capability (seam); one or more backends per seam, each its own package; `internal/patchbay` is the only place that selects backends. |
| 2 | Entrypoint | `cmd/ripper/main.go` → `internal/cli` (cobra). Follows [go.dev module layout](https://go.dev/doc/modules/layout). |
| 3 | Config | **viper only**, confined to `internal/cli`, decoded into `config.Config` with `Validate()` (C§1). |
| 4 | Service framework | go-service-kit v0.3.0: `lifecycle`, `obs`, `httpapi`, `outbound`. |
| 5 | Modes | `ripper serve` = engine + API (+ web UI unless `--headless`). |
| 6 | Ports | API (+UI, +docs) `:9090`; admin (`/healthz`, `/readyz`, `/metrics`, pprof) `:9091`. |
| 7 | Logging | One JSON stream to stdout, teed to `LOG_FILE`; tool output one record per line. |
| 8 | Shutdown mid-rip | Cancel the tool's process group, delete partial output, no eject, notify "stopped". |
| 9 | Backend config | Only `DETECTOR_BACKEND` and `EJECT_BACKEND`; notifications on iff `APPRISE_URLS` set. |
| 10 | Notifications | apprise-go v0.3.3 (`github.com/unraid/apprise-go`). |
| 11 | Fixtures | Synthetic + public (attributed) + hardware (authoritative). |
| 12 | User scripts | **Removed**: `/config/ripper.sh` and the per-disc hooks. New behaviour = new backend; external automation = apprise webhook targets. |
| 13 | Registry | `ghcr.io/jacaudi/docker-ripper` via `GITHUB_TOKEN`. |
| 14 | Web env names | `WEB_PATH_PREFIX`, `WEB_USERNAME`, `WEB_PASSWORD` only. |
| 15 | Upstream | Diverge permanently; drop upstream-only workflows. |
| 16 | Deployment | Target-agnostic binary; Docker/compose documented; k8s example with a device plugin. |
| 17 | API docs | Scalar 1.73.1, vendored + embedded, at `/docs`; OpenAPI at `/openapi.json`. |
| 18 | Web UI | **React 19 + Ant Design 6** (Vite 8, TypeScript 5.9), built into `internal/webui/dist` and embedded; typed client generated from huma's OpenAPI. |
| 19 | Go version | **Go 1.27** (`go 1.27.0`, `toolchain go1.27.2`), the latest stable as of 2026-10-10. |
| 20 | Task runner | **Task** v3.54 with a lowercase `taskfile.yml` at the root; no Makefile; CI calls the same tasks. |
| 21 | Quality gates | golangci-lint v2.14 (standard set + bug/security/idiom linters, `modernize`, gofumpt/goimports), `go mod tidy -diff`/`verify`, govulncheck, OSV-Scanner, CodeQL, Trivy, OpenSSF Scorecard, SHA-pinned actions, signed images with SBOM + provenance. |
| 22 | Releases | **release-please** (manifest mode, release-type `go`, Conventional Commits, GitHub App token); images published from the release workflow. |

## 3. Glossary

- **Seam**: an interface for one capability: `runner.Runner`, `detect.Detector`, `rip.Ripper`,
  `eject.Ejector`, `notify.Notifier`, `mkvkey.Source` (C§2.2).
- **Backend**: one package implementing a seam, e.g. `rip/abcde` (C§2.3).
- **Patchbay**: `internal/patchbay`; `Backends(...)` picks backends, `Spec(...)` assembles the
  HTTP servers and the `lifecycle.Spec`.
- **Engine**: `internal/engine`; the poll → detect → rip → finalize → eject → notify loop; a
  `lifecycle.Worker`.
- **Pass**: one iteration of the engine loop (C§3.3).
- **Rip plan**: the ordered rip steps for a (mode, kind) pair (C§3.4).
- **State / Kind**: drive state and disc kind enums (C§2.1); **engine state**: C§2.4.
- **fakebin**: the fake external-tools binary used in tests (T§3).
- **Fixture**: captured or synthetic tool output plus expectations (T§2).
- **Scenario**: one parity test case (T§4.1).
- **C§n / T§n / L§n**: section n of `contracts.md` / `testing.md` / `tooling.md`.

## 4. Rules (do / do not)

**Do**
- Use Go 1.27 primitives where they fit:
  - `os.Root` for every storage-root file operation;
  - `errors.AsType`;
  - `strings.Lines`;
  - `sync.WaitGroup.Go`;
  - `http.CrossOriginProtection`;
  - `testing/synctest` with `synctest.Sleep`;
  - `t.Context()`, `t.Attr`;
  - `context.WithoutCancel` for cleanup;
  - `debug.ReadBuildInfo` for the version.
- Run `task check` before every commit; `task parity` before every PR from phase 3.
- Use Conventional Commit messages and PR titles (L§4).
- Keep every seam interface at 1–2 methods. Add a compile-time assertion in each backend.
- Wrap errors with `%w` and context (`fmt.Errorf("abcde rip: %w", err)`).

**Do not**
- Import viper or cobra outside `internal/cli`. Use package-level viper. Use `AutomaticEnv`.
- Use package-level mutable state anywhere.
- Use `http.StripPrefix` (or anything that clones the request) as listener middleware.
- Use `cmd.StdoutPipe` with `cmd.Run`/`Wait`.
- Use `-ldflags -X` for the version.
- Add a `Makefile`, or run a check in CI that isn't a `task`.
- Use `encoding/json/v2` (still behind `GOEXPERIMENT` in 1.27).
- Use `synctest` around real processes or real listeners.
- Log the MakeMKV key (except via `obs.RedactAttr`), apprise URLs, or basic-auth credentials.
- Execute any user-supplied script. Call `useradd`/`groupadd`.
- Load anything from a CDN at runtime (UI, docs, fonts).
- Add config keys, endpoints or backends that aren't in `contracts.md`.

## 5. Principles applied

| Principle | Concretely |
|---|---|
| KISS | One binary, one engine loop, one log stream, no router in the UI. |
| YAGNI | Config keys only where contracts list them; native backends only after hardware tests; no build tag for headless. |
| DRY | One `rip.Ripper` interface for every kind; the engine owns output dirs; the TS API types are generated from the Go types. |
| SOLID | Small seams (ISP); substitutable backends (LSP); new backend = new package + one patchbay case (OCP); the engine depends only on seams (DIP); patchbay split into `Backends` and `Spec` (SRP). |
| 12-Factor | Env config (+ one flag); JSON logs to stdout. The `LOG_FILE` tee is a deliberate exception for the UI. Port binding via env; kit `lifecycle` for disposability; one-off admin commands (`detect`, `healthcheck`). |

## 6. Deployment and runtime

- **One instance, pinned to the host that owns the drive.**
  - Docker: `--device /dev/srN --device /dev/sgN`.
  - Kubernetes: a device-plugin DaemonSet ([generic-device-plugin](https://github.com/squat/generic-device-plugin/issues/62),
    [Talos guide](https://docs.siderolabs.com/kubernetes-guides/advanced-guides/device-plugins.md)),
    `replicas: 1`, `strategy: Recreate`, node selector.
- **PID 1 is tini**: `ENTRYPOINT ["tini","--","ripper"]`, `CMD ["serve"]`. Compose uses `init: true`.
- **Shutdown budget**: `lifecycle.Spec` uses `PropagationDelay: NoPropagationDelay` (single
  replica + Recreate, the case the kit documents), `DrainTimeout: 5s`, the default `FlushTimeout`
  (5s), and the engine worker `StopTimeout: 15s`. Total grace = 5 + 5 + 15 + 5 margin = **30 s**.
  Set compose `stop_grace_period: 30s` and k8s `terminationGracePeriodSeconds: 30`.
- **Health**: Docker `HEALTHCHECK CMD ["ripper","healthcheck"]` → admin `/healthz`. k8s probes
  use `/healthz` (liveness) and `/readyz` (readiness; checks the drive).
- Go ≥ 1.25 sets `GOMAXPROCS` from the container CPU limit automatically; nothing to do.

## 7. Dependencies

### 7.1 Go modules

cobra v1.10.2, viper v1.21.0, go-service-kit v0.3.0, apprise-go v0.3.3 (exact; pre-1.0),
`golang.org/x/sys` (phase 6). Notes:
- apprise-go's `Send` has no `context` and uses its own HTTP client. It bypasses kit `outbound`,
  so it is wrapped with a timeout (C§5.2); this is accepted.
- go-service-kit refused to vendor large JS. Ripper vendors Scalar anyway (decision 17), so ripper
  owns the CSP relaxation (`style-src 'unsafe-inline'` on `/docs` only) and the Renovate/sha256 upkeep.

### 7.2 External tools

| Tool | Decision | Backend |
|---|---|---|
| `makemkvcon` | keep (proprietary) | `detect/makemkv`, `rip/makemkv`, registration |
| `abcde` + cdparanoia/lame/flac/eyeD3/metaflac/glyrc | keep | `rip/abcde` |
| `ddrescue` | keep | `rip/ddrescue` |
| `cdparanoia -Q` | keep → ioctl in phase 6 | inside `detect/makemkv` → `detect/native` |
| `eject`, `sdparm` | keep → ioctl in phase 6 | `eject/execeject` → `eject/ioctleject` |
| curl, grep/sed/cut/date/timeout, useradd, python/flask, phusion, syslog-ng | **remove** | apprise-go, kit, stdlib |
| user hook scripts | **remove** | — |

## 8. Behaviour changes (migration notes)

| # | Old | New |
|---|---|---|
| D1 | Unordered pattern map; empty label matched "CD" | Ordered rules on parsed fields (C§3.1) |
| D2 | Data CDs sent to abcde | Data CDs → ISO. DVD/BD data discs unchanged until phase 6 |
| D3 | Unrecognised reply below threshold → "rip" branch + eject | Counted as bad; no rip, no eject |
| D4 | Drive index from `cut -c5` | Parsed integer (indexes ≥ 10 work) |
| D5 | `--profile=/config/default.mmcp.xml` (never existed) | `$CONFIG_DIR/default.mmcp.xml` if present, else the embedded default |
| D6 | `ALSOMAKEISO` imaged audio CDs after abcde had ejected them | Skipped with a warning |
| D7 | Raw/empty labels used as dir names | Sanitised; empty → `disc_<ts>`; collisions suffixed |
| D8 | Bad threshold killed the ripper; the UI kept running | Process exits 1; restart policy restarts it |
| D9 | MakeMKV key printed; `~/.MakeMKV` 777 | Key redacted; dir 0700, file 0600 |
| D10 | Pushover via `POVER_*`, fixed text | `APPRISE_URLS`; Success/Failure/Stopped events |
| D11 | Basic auth realm "FeedCrawler", plain compare | Realm "Ripper", constant-time; CSRF protection |
| D12 | `/config/ripper.sh` and hook scripts executed | Not executed |
| D13 | `useradd`/`groupadd` at start | Name lookup, else numeric IDs |
| D14 | `PREFIX`/`USER`/`PASS` | `WEB_PATH_PREFIX`/`WEB_USERNAME`/`WEB_PASSWORD` |
| D15 | Plain-text log; `DEBUG`/`DEBUGTOWEB` | JSON log; `LOG_LEVEL` |
| D16 | `/api/log/` | `/api/v1/log`, `/api/v1/status`; OpenAPI + Scalar `/docs` |
| D17 | One port 9090 | 9090 API/UI/docs, 9091 admin |
| D18 | `docker stop` left partial output | Clean cancel; partial output removed |
| D19 | phusion base, python, syslog-ng | `ubuntu:noble` + tools + tini + `ripper` |
| D20 | Booleans: only literal `true` | Anything `strconv.ParseBool` accepts |
| D21 | A user `abcde.conf` `OUTPUTDIR` overrode `STORAGE_CD` | `STORAGE_CD` always decides (staging dir) |
| D22 | abcde `-x` / `EJECTCD=y` ejected even with `EJECTENABLED=false` | No `-x`, `EJECTCD=n`; the engine is the only ejector |
| D23 | `rm -rf /tmp/*.tmp` every loop | Removed; ripper cleans its own temp files |
| D24 | petite-vue/bootstrap log page | React + AntD UI (status + log) |
| D25 | `chmod` honoured the umask; symbolic only | Umask ignored; octal also accepted |
| D26 | chown/chmod of whole storage roots and `/config` | Only the finalized output path |
| D27 | `JUSTMAKEISO` tried to image audio CDs | Skipped with a warning |
| D28 | A failed rip still moved/chowned its output | Partial output removed; Failure notification; disc still ejected |

## 9. Repository admin (manual)

- Protect `archive`: Settings → Rules → Rulesets → New branch ruleset → target `archive`;
  enable *Restrict deletions*, *Block force pushes*, *Restrict updates*.
- ghcr publishing needs `permissions: packages: write` in the workflow; nothing else.
- Create a GitHub App for release-please (contents + pull-requests: write), install it on the repo,
  and add the secrets `RELEASE_APP_ID` and `RELEASE_APP_PRIVATE_KEY` (L§5).
- Settings → General: allow squash merging only, with the PR title as the default commit message.
- Settings → Actions → General: allow GitHub Actions to create and approve pull requests.
- Branch protection on `main`: require `ci` jobs, `security` CodeQL, and a linear history.

## 10. Remaining risks

- **apprise-go** is pre-1.0 and not every target is tested upstream. It sits behind the
  `Notifier` seam; re-evaluate at phase 3.
- **golangci-lint vs Go 1.27**: verified that v2.14.0 built with go1.27.2 accepts our config. `task tools`
  builds it from source with the repo toolchain for exactly this reason.
- **DVD/BD data-disc detection** has no known MakeMKV signal; it waits for hardware fixtures (phase 6).

## References

- go-service-kit: https://github.com/leftathome/go-service-kit
- Task: https://taskfile.dev · golangci-lint: https://golangci-lint.run · govulncheck: https://go.dev/doc/security/vuln/
- release-please action: https://github.com/googleapis/release-please-action · Conventional Commits: https://www.conventionalcommits.org
- OSV-Scanner: https://google.github.io/osv-scanner/ · OpenSSF Scorecard: https://github.com/ossf/scorecard · Trivy: https://trivy.dev
- apprise-go: https://pkg.go.dev/github.com/unraid/apprise-go · https://unraid.net/blog/apprise-go
- Scalar configuration: https://github.com/scalar/scalar/blob/main/documentation/configuration.md
- Go module layout: https://go.dev/doc/modules/layout
- Device plugins: https://github.com/squat/generic-device-plugin/issues/62 ·
  https://docs.siderolabs.com/kubernetes-guides/advanced-guides/device-plugins.md
- Docker signals / grace period: https://oneuptime.com/blog/post/2026-01-16-docker-graceful-shutdown-signals/view ·
  https://www.netdata.cloud/guides/docker/docker-exit-code-143/
