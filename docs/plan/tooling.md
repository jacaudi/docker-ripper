# Tooling, quality gates and releases

Normative, like `contracts.md`. Copy these files exactly. Versions are pinned and Renovate
keeps them current (§6).

**One source of truth:** every check runs through `task <name>`, both locally and in CI. CI
workflows install Go, Node and Task, then call tasks. They never re-implement a check.

## 1. Task runner — `taskfile.yml` (repo root, lowercase)

[Task](https://taskfile.dev) v3.54.0 replaces the Makefile. Install it locally with
`go install github.com/go-task/task/v3/cmd/task@v3.54.0`; CI does the same.

```yaml
version: '3'

vars:
  BIN: '{{.ROOT_DIR}}/bin'
  GOLANGCI_LINT_VERSION: v2.14.0   # renovate: datasource=go depName=github.com/golangci/golangci-lint/v2
  GOVULNCHECK_VERSION: v1.8.0      # renovate: datasource=go depName=golang.org/x/vuln
  SCALAR_VERSION: 1.73.1           # renovate: datasource=npm depName=@scalar/api-reference

env:
  GOFLAGS: -mod=readonly

tasks:
  default:
    desc: List tasks
    silent: true
    cmds: [task --list]

  tools:
    desc: Install pinned Go tools into ./bin (built with this repo's Go toolchain)
    status:
      - test -x {{.BIN}}/golangci-lint && {{.BIN}}/golangci-lint version | grep -q '{{trimPrefix "v" .GOLANGCI_LINT_VERSION}}'
      - test -x {{.BIN}}/govulncheck
    cmds:
      - GOBIN={{.BIN}} go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@{{.GOLANGCI_LINT_VERSION}}
      - GOBIN={{.BIN}} go install golang.org/x/vuln/cmd/govulncheck@{{.GOVULNCHECK_VERSION}}

  fmt:
    desc: Format Go code (gofumpt + goimports)
    deps: [tools]
    cmds: ['{{.BIN}}/golangci-lint fmt ./...']

  lint:
    desc: Go lint, static analysis, modernize, module hygiene
    deps: [tools]
    cmds:
      - '{{.BIN}}/golangci-lint run ./...'
      - go mod tidy -diff
      - go mod verify

  test:
    desc: Go unit + integration tests
    cmds: [go test -race -shuffle=on -count=1 ./...]

  vuln:
    desc: Go vulnerability scan (reachability-aware)
    deps: [tools]
    cmds: ['{{.BIN}}/govulncheck ./...']

  ui:deps:
    dir: web
    sources: [package.json, package-lock.json]
    generates: [node_modules/.package-lock.json]
    cmds: [npm ci]

  ui:gen:
    desc: Regenerate the TS API types from the OpenAPI golden
    deps: [ui:deps]
    dir: web
    cmds: [npm run gen]

  ui:lint:
    deps: [ui:deps]
    dir: web
    cmds: [npm run lint, npm run format:check, npm run typecheck]

  ui:test:
    deps: [ui:deps]
    dir: web
    cmds: [npm test]

  ui:build:
    deps: [ui:deps]
    dir: web
    sources: [src/**/*, index.html, vite.config.ts, package-lock.json]
    generates: [../internal/webui/dist/index.html]
    cmds: [npm run build]

  scalar:vendor:
    desc: Download, verify and vendor the Scalar bundle
    cmds: ['bash scripts/vendor-scalar.sh {{.SCALAR_VERSION}}']

  build:
    desc: Build ./bin/ripper
    deps: [ui:build]
    cmds: ['go build -trimpath -o {{.BIN}}/ripper ./cmd/ripper']

  image:
    desc: Build both container images locally
    cmds:
      - docker build -f latest/Dockerfile -t ripper:dev .
      - docker build -f manual-build/Dockerfile -t ripper:dev-manual .

  check:
    desc: Everything CI runs on a PR, except the image build
    cmds:
      - task: lint
      - task: test
      - task: vuln
      - task: ui:lint
      - task: ui:test
      - task: ui:build
```

Rules: no `Makefile`. Each task does one thing. CI calls the same task names.

## 2. Go lint and static analysis — `.golangci.yml`

golangci-lint **v2.14.0**. `task tools` installs it with `go install` on purpose, so the binary
is built with this repo's Go 1.27 toolchain and can analyse Go 1.27 code. Do not switch to the
prebuilt binary or the golangci-lint GitHub Action.

```yaml
version: "2"

linters:
  default: standard            # errcheck, govet, ineffassign, staticcheck, unused
  enable:
    - bodyclose
    - contextcheck
    - copyloopvar
    - errname
    - errorlint
    - exhaustive
    - fatcontext
    - gocritic
    - gosec
    - intrange
    - misspell
    - modernize                # Go 1.2x idioms (replaces a separate `go fix -diff` gate)
    - nilerr
    - nilnesserr
    - noctx
    - nolintlint
    - perfsprint
    - revive
    - sloglint
    - unconvert
    - unparam
    - usestdlibvars
    - usetesting
  settings:
    govet:
      enable-all: true
      disable: [fieldalignment, shadow]
    exhaustive:
      default-signifies-exhaustive: true
    nolintlint:
      require-explanation: true
      require-specific: true
    sloglint:
      no-global: all
      static-msg: true
      key-naming-case: snake
  exclusions:
    presets: [std-error-handling, common-false-positives]
    rules:
      - path: _test\.go
        linters: [gosec, noctx, perfsprint, unparam]
      - path: internal/runner/execrunner/
        linters: [gosec]
        text: G204             # executing configured tools is this package's job

formatters:
  enable: [gofumpt, goimports]
  settings:
    goimports:
      local-prefixes: [github.com/jacaudi/docker-ripper]
```

What it covers:

| Concern | Tool |
|---|---|
| Go standards and idioms | `gofumpt`, `goimports`, `go vet` (all analyzers except fieldalignment/shadow), `staticcheck` (SA/S/ST/QF checks), `revive`, `gocritic`, `modernize` |
| Bugs | `errcheck`, `errorlint`, `nilerr`, `nilnesserr`, `bodyclose`, `contextcheck`, `noctx`, `exhaustive`, `fatcontext` |
| Security (SAST) | `gosec`, plus CodeQL (§3) |
| Module hygiene | `go mod tidy -diff`, `go mod verify` |
| Suppressions | `nolintlint` requires a specific linter and a reason |

Frontend (`web/`): ESLint 10 flat config with `typescript-eslint` `strictTypeChecked` +
`eslint-plugin-react-hooks`, **Prettier** (`format:check` = `prettier --check .`), and
`tsc --noEmit`. Scripts in `web/package.json`: `lint`, `format:check`, `typecheck`, `test`, `build`, `gen`.

## 3. Vulnerabilities and supply chain

| Layer | Tool | Where | Fails the build when |
|---|---|---|---|
| Go code paths | **govulncheck** v1.8.0 (Go vuln DB, reachability-aware) | `task vuln` in `ci.yml`; weekly in `security.yml` | any reachable vulnerability |
| All lockfiles (go.sum, web/package-lock.json) | **OSV-Scanner** v2 (`google/osv-scanner-action`, its reusable PR + scheduled workflows) | `security.yml` | any known vuln with a fix (PR); report-only (scheduled) |
| SAST | **CodeQL** (`github/codeql-action`), languages `go`, `javascript-typescript` | `security.yml` on PR, push to main, weekly | high/critical alerts (via branch protection on code scanning) |
| Container image | **Trivy** (`aquasecurity/trivy-action`), `severity: CRITICAL,HIGH`, `ignore-unfixed: true` | `ci.yml` job `image` and `release.yml` before push | any match |
| Repo posture | **OpenSSF Scorecard** (`ossf/scorecard-action`), SARIF upload | `security.yml` weekly + push to main | never (report) |
| Provenance | buildx `provenance: mode=max`, `sbom: true`; **cosign** keyless signing (`sigstore/cosign-installer`, `cosign sign --yes <image>@<digest>`) | `release.yml` | sign failure |
| Dependency updates | **Renovate** (§6) | — | — |

Workflow hygiene (OpenSSF/Scorecard practices):
- Every action is pinned by **full commit SHA** with a `# vX.Y.Z` comment. Use the current major
  from the action's README at the time of writing; Renovate keeps the SHAs current.
- Every workflow declares the minimum `permissions:` at the top (`contents: read`) and widens
  only per job.
- No `pull_request_target`. No secrets in PR workflows.

## 4. Conventional Commits

- Commit and PR-title format: `<type>(<optional scope>): <summary>`. Types: `feat`, `fix`,
  `perf`, `refactor`, `docs`, `test`, `build`, `ci`, `chore`, `revert`. Breaking change: `!`
  after the type/scope plus a `BREAKING CHANGE:` footer.
- Repo settings: squash merge only, default message = PR title.
  `amannn/action-semantic-pull-request` in `ci.yml` validates PR titles.
- Plan tasks: `<type>(<package>): <summary> (P<phase>.<n>)`, e.g.
  `feat(disc): parse makemkv DRV lines (P1.2)`.
- The conversion's user-visible breaks (main plan §8) land in the phase 5 PR as `feat!:` with a
  `BREAKING CHANGE:` footer pointing to the migration notes.

## 5. Releases — release-please

`googleapis/release-please-action@v4` (manifest mode). It keeps a release PR open; merging it
tags `vX.Y.Z`, writes `CHANGELOG.md` and creates a GitHub Release.

`release-please-config.json`:
```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "bump-minor-pre-major": true,
  "bump-patch-for-minor-pre-major": false,
  "include-component-in-tag": false,
  "packages": {
    ".": {
      "release-type": "go",
      "package-name": "ripper",
      "changelog-path": "CHANGELOG.md",
      "extra-files": [
        { "type": "json", "path": "web/package.json", "jsonpath": "$.version" }
      ]
    }
  }
}
```

`.release-please-manifest.json`: `{ ".": "0.0.0" }`. The first release is `0.1.0`; `1.0.0` is cut
by hand (`Release-As: 1.0.0` footer) once phase 5 ships.

**Token.** Tags and PRs created with the default `GITHUB_TOKEN` don't trigger other
workflows, so CI wouldn't run on the release PR. Use a **GitHub App** token
(`actions/create-github-app-token`, secrets `RELEASE_APP_ID` and `RELEASE_APP_PRIVATE_KEY`; the
App needs contents + pull-requests write) and pass it as the action's `token` (manual setup, main
plan §9). Image publishing runs in the **same workflow**, gated on the release output, so it
doesn't depend on tag-triggered workflows either.

`release.yml` outline:
```yaml
on:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  release-please:
    runs-on: ubuntu-latest
    outputs:
      created: ${{ steps.rp.outputs.release_created }}
      tag: ${{ steps.rp.outputs.tag_name }}
      version: ${{ steps.rp.outputs.version }}
    steps:
      - uses: actions/create-github-app-token@<sha> # vN
        id: app
        with:
          app-id: ${{ secrets.RELEASE_APP_ID }}
          private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
      - uses: googleapis/release-please-action@<sha> # v4
        id: rp
        with:
          token: ${{ steps.app.outputs.token }}
  publish:
    needs: release-please
    if: needs.release-please.outputs.created == 'true'
    permissions:
      contents: read
      packages: write
      id-token: write   # cosign keyless
    # checkout at the tag with fetch-depth: 0 (debug.ReadBuildInfo needs the tag in .git),
    # buildx multi-arch, Trivy, push, cosign sign
```

Image tags on release `vX.Y.Z`:
- `latest/` Dockerfile (amd64 + arm64): `X.Y.Z`, `X.Y`, `latest`.
- `manual-build/` (amd64): `X.Y.Z-manual`, `manual-latest`.

`rebuild.yml` (weekly, Monday 03:17 UTC, plus manual dispatch): check out the **latest
release tag**, rebuild both images (new MakeMKV beta), Trivy, then push the same tags and sign.
This replaces the old forum-polling and base-image-polling workflows.

## 6. Renovate — `renovate.json`

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", "helpers:pinGitHubActionDigests", ":semanticCommits"],
  "postUpdateOptions": ["gomodTidy"],
  "customManagers": [
    {
      "customType": "regex",
      "managerFilePatterns": ["/^taskfile\\.yml$/"],
      "matchStrings": ["(?<currentValue>v?[\\d.]+)\\s+# renovate: datasource=(?<datasource>\\S+) depName=(?<depName>\\S+)"]
    }
  ],
  "packageRules": [
    { "matchManagers": ["npm"], "matchFileNames": ["web/**"], "groupName": "web" },
    { "matchDepNames": ["@scalar/api-reference"], "postUpgradeTasks": { "commands": ["task scalar:vendor"] } }
  ]
}
```

(`postUpgradeTasks` needs a self-hosted Renovate or an allow-list. On the hosted app, a Scalar
bump PR fails CI until someone runs `task scalar:vendor` on the branch. That's acceptable and
documented in the PR template.)

## 7. Workflows summary

| File | Triggers | Jobs |
|---|---|---|
| `ci.yml` | pull_request, push main | `pr-title` (PRs only), `go` (`task lint test vuln`; includes the e2e smoke), `ui` (`task ui:lint ui:test ui:build` + `git diff --exit-code web/src/api/schema.d.ts`), `image` (build both images + Trivy; no push) |
| `security.yml` | pull_request, push main, weekly | CodeQL (go, javascript-typescript), OSV-Scanner, Scorecard (push/weekly only) |
| `release.yml` | push main | release-please → publish on release |
| `rebuild.yml` | weekly, dispatch | rebuild + push the latest release |

All jobs: `actions/checkout`, `actions/setup-go` with `go-version-file: go.mod`,
`actions/setup-node` with `node-version: 24`, `cache: npm` and `cache-dependency-path: web/package-lock.json`
(UI jobs), then `go install github.com/go-task/task/v3/cmd/task@v3.54.0`.
