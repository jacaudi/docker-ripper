# Contracts

Everything in this file is **normative**. When a task in [`phases.md`](phases.md) says
"implement X per contracts §N", copy names, signatures, values and orderings exactly.
If something you need is missing here, stop and ask; do not invent it.

Module path: `github.com/jacaudi/docker-ripper`. All packages below live under it.

---

## 1. Configuration

### 1.1 Rules

- **viper is the only config loader**, and only `internal/cli` imports it.
- Use `v := viper.New()`, never the package-level viper.
- For every row in §1.2: `v.SetDefault(key, default)` and `v.BindEnv(key, ENV_NAME)`.
  Bind each name explicitly; do not use `AutomaticEnv` or a key replacer.
- Flags: `serve` defines `--headless` only. Bind it with `v.BindPFlag("headless", cmd.Flags().Lookup("headless"))`
  inside `PersistentPreRunE`. Precedence: flag > env > default.
- **An empty env value counts as unset** and the default applies. This is viper's default
  (`AllowEmptyEnv(false)`) and matches bash `${VAR:=default}`. Do not change it, and test it.
- Decode with `v.Unmarshal(&cfg)`. Viper's default hooks already convert strings to
  `time.Duration` and comma-separated strings to `[]string`.
- Booleans accept everything `strconv.ParseBool` accepts. Legacy bash accepted only `true` (deviation D20).
- After decoding, call `cfg.Validate()`. It returns `errors.Join` of **every** problem.

### 1.2 Keys

| Env name | viper key / Go field | Type | Default | Validation |
|---|---|---|---|---|
| `DRIVE` | `drive` / `Drive` | string | `/dev/sr0` | non-empty, absolute |
| `STORAGE_CD` | `storage_cd` / `StorageCD` | string | `/out/Ripper/CD` | absolute |
| `STORAGE_DATA` | `storage_data` / `StorageData` | string | `/out/Ripper/DATA` | absolute |
| `STORAGE_DVD` | `storage_dvd` / `StorageDVD` | string | `/out/Ripper/DVD` | absolute |
| `STORAGE_BD` | `storage_bd` / `StorageBD` | string | `/out/Ripper/BluRay` | absolute |
| `EJECTENABLED` | `eject_enabled` / `EjectEnabled` | bool | `true` | — |
| `JUSTMAKEISO` | `just_make_iso` / `JustMakeISO` | bool | `false` | not both with `ALSOMAKEISO` |
| `ALSOMAKEISO` | `also_make_iso` / `AlsoMakeISO` | bool | `false` | — |
| `SEPARATERAWFINISH` | `separate_raw_finish` / `SeparateRawFinish` | bool | `false` | — |
| `TIMESTAMPPREFIX` | `timestamp_prefix` / `TimestampPrefix` | bool | `false` | — |
| `MINIMUMLENGTH` | `minimum_length` / `MinimumLength` | int (seconds) | `600` | ≥ 0 |
| `BAD_THRESHOLD` | `bad_threshold` / `BadThreshold` | int | `5` | ≥ 1 |
| `FILEUSER` | `file_user` / `FileUser` | string | `nobody` | non-empty |
| `FILEUSERID` | `file_user_id` / `FileUserID` | int | `321` | ≥ 0 |
| `FILEGROUP` | `file_group` / `FileGroup` | string | `users` | non-empty |
| `FILEGROUPID` | `file_group_id` / `FileGroupID` | int | `4321` | ≥ 0 |
| `FILEMODE` | `file_mode` / `FileMode` | string | `g+rw` | parses with `output.ParseMode` (§5.4) |
| `KEY` | `key` / `Key` | string | `""` | empty or `^T-[A-Za-z0-9@_]{66}$` |
| `APPRISE_URLS` | `apprise_urls` / `AppriseURLs` | []string (comma-sep) | empty | each accepted by `apprise.New().Add` |
| `API_ADDR` | `api_addr` / `APIAddr` | string | `:9090` | `net.SplitHostPort` ok |
| `ADMIN_ADDR` | `admin_addr` / `AdminAddr` | string | `:9091` | `net.SplitHostPort` ok, ≠ `API_ADDR` |
| `HEADLESS` (`--headless`) | `headless` / `Headless` | bool | `false` | — |
| `WEB_PATH_PREFIX` | `web_path_prefix` / `WebPathPrefix` | string | `""` | normalised (§1.3) |
| `WEB_USERNAME` | `web_username` / `WebUsername` | string | `""` | both-or-neither with password |
| `WEB_PASSWORD` | `web_password` / `WebPassword` | string | `""` | both-or-neither with username |
| `API_DOCS_ENABLED` | `api_docs_enabled` / `APIDocsEnabled` | bool | `true` | — |
| `PPROF_ENABLED` | `pprof_enabled` / `PprofEnabled` | bool | `false` | — |
| `CONFIG_DIR` | `config_dir` / `ConfigDir` | string | `/config` | absolute |
| `LOG_FILE` | `log_file` / `LogFile` | string | `/config/Ripper.log` | absolute |
| `LOG_LEVEL` | `log_level` / `LogLevel` | string | `info` | `debug\|info\|warn\|error` (`obs.ParseLevel`) |
| `DETECTOR_BACKEND` | `detector_backend` / `DetectorBackend` | string | `makemkv` | `makemkv` (+ `native` after phase 6) |
| `EJECT_BACKEND` | `eject_backend` / `EjectBackend` | string | `exec` | `exec` (+ `ioctl` after phase 6) |
| `POLL_INTERVAL` | `poll_interval` / `PollInterval` | duration | `60s` | ≥ 1s |
| `MANUAL_EJECT_POLL` | `manual_eject_poll` / `ManualEjectPoll` | duration | `5s` | ≥ 1s |
| `DETECT_TIMEOUT` | `detect_timeout` / `DetectTimeout` | duration | `30s` | ≥ 1s |

Every Go field gets a `mapstructure:"<viper key>"` tag.

**Removed variables.** At startup, if any of these is set (non-empty), log one WARN naming it
and its replacement, then continue:
`DEBUG`, `DEBUGTOWEB` → `LOG_LEVEL`; `POVER_APP_TOKEN`, `POVER_USER_KEY` → `APPRISE_URLS`
(`pover://USER_KEY@APP_TOKEN`); `OPTIONAL_WEB_UI_PATH_PREFIX`, `OPTIONAL_WEB_UI_USERNAME`,
`OPTIONAL_WEB_UI_PASSWORD` → `WEB_*`. Do **not** warn on `PREFIX`/`USER`/`PASS`, because shells set `USER`.

### 1.3 `WEB_PATH_PREFIX` normalisation

`""` or `/` → `""`. Otherwise ensure exactly one leading `/` and strip trailing `/`.
Reject anything containing `?`, `#`, `{`, `}` or whitespace. Examples: `ripper` → `/ripper`;
`/ripper/` → `/ripper`; `/a/b` → `/a/b`.

---

## 2. Go types and interfaces (copy exactly)

### 2.1 `internal/disc`

```go
package disc

// State is the drive state reported by the detector.
type State int

const (
	StateUnknown  State = iota // unparseable or unexpected reply
	StateEmpty                 // no disc
	StateOpen                  // tray open
	StateLoading               // disc spinning up
	StateInserted              // disc present; see Kind
)

// Kind is what is in the drive when State == StateInserted.
type Kind int

const (
	KindNone    Kind = iota
	KindBluRay       // includes UHD
	KindDVD
	KindCD           // parser-only: a CD not yet classified; the detector must resolve it
	KindAudioCD
	KindData
)

type Disc struct {
	Index  int    // makemkv drive index
	State  State
	Kind   Kind
	Label  string // raw label as reported (NOT sanitised)
	Device string // e.g. /dev/sr0
	Raw    string // the DRV: line that produced this value
}

func (s State) String() string // "unknown","empty","open","loading","inserted"
func (k Kind) String() string  // "none","bluray","dvd","cd","audio_cd","data"

var (
	ErrNoDriveLine = errors.New("disc: no DRV line for device")
	ErrMalformed   = errors.New("disc: malformed DRV line")
)

// ParseDRV finds the DRV line whose device field equals device and classifies it per §3.1.
func ParseDRV(out []byte, device string) (Disc, error)
```

### 2.2 Seams (interfaces live in the seam package; backends in sub-packages)

```go
// internal/runner
package runner

type Cmd struct {
	Name string   // executable, resolved via PATH
	Args []string
	Tool string   // log attribute value, e.g. "makemkvcon"
}

type Runner interface {
	// Run streams stdout+stderr line by line to the logger and waits.
	Run(ctx context.Context, c Cmd) error
	// Output returns combined stdout+stderr (capped at 1 MiB) and also logs it at DEBUG.
	Output(ctx context.Context, c Cmd) ([]byte, error)
}

// ExitError reports a non-zero exit. Wrapped; use errors.AsType[*runner.ExitError].
type ExitError struct {
	Tool string
	Code int
}

func (e *ExitError) Error() string // "<tool>: exit status <code>"
```

```go
// internal/detect
package detect

type Detector interface {
	// Detect returns a Disc whose Kind is never KindCD.
	Detect(ctx context.Context) (disc.Disc, error)
}
```

```go
// internal/rip
package rip

// Ripper writes its output into dir, which the engine has created and owns.
// On ctx cancellation it returns promptly; the engine deletes dir.
type Ripper interface {
	Rip(ctx context.Context, d disc.Disc, dir string) error
}
```

```go
// internal/eject
package eject

type Ejector interface {
	Eject(ctx context.Context) error
}
```

```go
// internal/notify
package notify

type Kind int

const (
	Success Kind = iota
	Failure
	Stopped
)

type Event struct {
	Kind  Kind
	Title string
	Body  string
}

type Notifier interface {
	Notify(ctx context.Context, e Event) error
}
```

```go
// internal/mkvkey
package mkvkey

type Source interface {
	Key(ctx context.Context) (string, error)
}
```

### 2.3 Backends (package path → constructor)

| Package | Constructor | Implements |
|---|---|---|
| `internal/runner/execrunner` | `New(logger *slog.Logger) *Runner` | `runner.Runner` |
| `internal/detect/makemkv` | `New(r runner.Runner, device string, timeout time.Duration) *Detector` | `detect.Detector` |
| `internal/rip/makemkv` | `New(r runner.Runner, profilePath string, minLength int) *Ripper` | `rip.Ripper` |
| `internal/rip/abcde` | `New(r runner.Runner, device, baseConfPath string) *Ripper` | `rip.Ripper` |
| `internal/rip/ddrescue` | `New(r runner.Runner, device string) *Ripper` | `rip.Ripper` |
| `internal/eject/execeject` | `New(r runner.Runner, device string) *Ejector` | `eject.Ejector` |
| `internal/notify/apprise` | `New(urls []string) (*Notifier, error)` | `notify.Notifier` |
| `internal/notify/nop` | `New() Notifier` (value type) | `notify.Notifier` |
| `internal/mkvkey/env` | `New(key string) Source` | `mkvkey.Source` |
| `internal/mkvkey/forum` | `New(d outbound.Doer, url string) *Source`; `const DefaultURL` | `mkvkey.Source` |
| phase 6: `internal/eject/ioctleject`, `internal/detect/native` | see phases.md P6 | |

The detect and rip `makemkv` packages share the name `makemkv`. Import them in
`internal/patchbay` as `makemkvdetect "…/internal/detect/makemkv"` and
`makemkvrip "…/internal/rip/makemkv"`. Each package gets a compile-time assertion, e.g.
`var _ rip.Ripper = (*Ripper)(nil)`.

### 2.4 Engine

```go
// internal/engine
package engine

type State string

const (
	StateIdle            State = "idle"
	StateDetecting       State = "detecting"
	StateRipping         State = "ripping"
	StateEjecting        State = "ejecting"
	StateWaitingForEject State = "waiting_for_eject"
	StateStopped         State = "stopped"
)

type Mode int

const (
	ModeNormal  Mode = iota
	ModeJustISO      // JUSTMAKEISO
	ModeAlsoISO      // ALSOMAKEISO
)

type Deps struct {
	Detect  detect.Detector
	Rippers map[disc.Kind]rip.Ripper // keys: KindBluRay, KindDVD, KindAudioCD, KindData
	ISO     rip.Ripper               // the ddrescue ripper (same instance as Rippers[KindData])
	Eject   eject.Ejector
	Notify  notify.Notifier
	Output  *output.Planner
	Logger  *slog.Logger
	Now     func() time.Time // time.Now in prod; tests may override
}

type Config struct {
	Mode            Mode
	EjectEnabled    bool
	BadThreshold    int
	PollInterval    time.Duration
	ManualEjectPoll time.Duration
}

type Engine struct{ /* unexported */ }

func New(d Deps, c Config) *Engine
func (e *Engine) Run(ctx context.Context) error // lifecycle.Worker.Run
func (e *Engine) Status() Status                // safe for concurrent use (sync.Mutex)

var ErrTooManyBadResponses = errors.New("engine: too many consecutive bad detector responses")

type Status struct {
	State        State       `json:"state"`
	Disc         *DiscInfo   `json:"disc,omitempty"`
	StartedAt    *time.Time  `json:"started_at,omitempty"`
	LastResult   *LastResult `json:"last_result,omitempty"`
	BadResponses int         `json:"bad_responses"`
}

type DiscInfo struct {
	Kind   string `json:"kind"`  // disc.Kind.String()
	Label  string `json:"label"`
	Device string `json:"device"`
}

type LastResult struct {
	Label      string    `json:"label"`
	Kind       string    `json:"kind"`
	Outcome    string    `json:"outcome"` // "success" | "failure" | "cancelled" | "skipped"
	Error      string    `json:"error,omitempty"`
	Dir        string    `json:"dir,omitempty"`
	FinishedAt time.Time `json:"finished_at"`
}
```

---

## 3. Behaviour tables

### 3.1 DRV parsing (`disc.ParseDRV`)

1. Iterate lines with `strings.Lines`. Keep lines that start with `DRV:`.
2. Parse `strings.TrimPrefix(line, "DRV:")` with `csv.NewReader` (`LazyQuotes: true`,
   `FieldsPerRecord: -1`). Fewer than 7 fields → `ErrMalformed`.
3. Fields: `[0]` index (int), `[1]` state (int), `[2]` ignored, `[3]` flags (int),
   `[4]` drive name, `[5]` label, `[6]` device.
4. Pick the first line whose `[6] == device`. None → `ErrNoDriveLine`.
5. Classify, **in this order**:

| state | flags | → State | → Kind |
|---|---|---|---|
| 0 | any | Empty | None |
| 1 | any | Open | None |
| 3 | any | Loading | None |
| 2 | 12 or 28 | Inserted | BluRay |
| 2 | 1 | Inserted | DVD |
| 2 | 0 | Inserted | CD (detector resolves) |
| anything else | | Unknown | None |

An empty label never changes the classification (this fixes legacy bug #1).

### 3.2 Detector (`detect/makemkv`)

1. `Output(ctx with DetectTimeout, {Name:"makemkvcon", Args:["-r","--cache=1","info","disc:9999"], Tool:"makemkvcon"})`.
   A timeout or exit error returns `(Disc{State: StateUnknown}, err)`.
2. `disc.ParseDRV(out, device)`. On error return `StateUnknown` and the error.
3. If `Kind == KindCD` **or** `State == StateEmpty`: run
   `Output(ctx, {Name:"cdparanoia", Args:["-d",device,"-Q"], Tool:"cdparanoia"})` and ignore
   its exit code. If the output contains `audio tracks`, the result is State Inserted +
   Kind AudioCD. Otherwise, if Kind was CD, it is Kind Data; if State was Empty, it stays Empty.
4. Return the Disc. DVD/BD **data** discs stay DVD/BluRay until a hardware fixture shows a
   distinguishing signal (phase 6).

### 3.3 Engine loop (`engine.Run`)

`Run` uses `ticker := time.NewTicker(cfg.PollInterval)` and runs one **pass** immediately,
then one per tick, until `ctx.Done()`. A pass:

1. State = detecting. Call `Detect(ctx)`.
2. On error or `StateUnknown`: `bad++`; log WARN with `err` and `raw`. If `bad >= BadThreshold`:
   state ejecting → `Eject` (with `ctx`; errors logged) → `notifyDetached(Stopped, "Ripper stopped", "too many bad detector responses")`
   → state stopped → return `ErrTooManyBadResponses`. Otherwise state idle; end the pass (**no eject**, D3).
3. Otherwise `bad = 0`.
4. `Empty`, `Open`, `Loading` → log INFO `"no disc"` / `"tray open"` / `"disc loading"` → state idle → end the pass.
5. `Inserted` → run the **rip plan** for `(Mode, Kind)` from §3.4. State ripping; `StartedAt = Now()`.
6. Each step of the plan: `dir := Output.Prepare(k, label, Now())` (§3.5), where `k` is the
   disc's Kind for a kind step and `disc.KindData` for every ISO step (also used for
   `Finalize`). Then
   `err := ripper.Rip(ctx, d, dir)`.
   - If `ctx.Err() != nil` (shutdown): `cleanup(dir)` using
     `context.WithTimeout(context.WithoutCancel(ctx), 5*time.Second)`, record outcome `cancelled`,
     `notifyDetached(Stopped, "Ripper stopped", "shutdown during rip of <label>")`, **do not eject**,
     return `nil`.
   - If `err != nil`: record outcome `failure`, `cleanup(dir)`, skip the remaining steps.
   - Else `Output.Finalize(kind, dir)` (§3.5).
7. After the plan: state ejecting.
   - If `EjectEnabled`: `Eject(ctx)`. An error is logged at ERROR; continue.
   - Else state waiting_for_eject: log INFO `"safe to eject"`; then every `ManualEjectPoll`
     call `Detect`. Stop waiting when State is Empty or Open. Errors are ignored and do not
     count toward `bad`. Respect `ctx.Done()` (return `nil`).
8. Notify: `Success` if every step succeeded, otherwise `Failure` (texts in §3.7). Then state idle.

`notifyDetached` = `Notify(context.WithTimeout(context.WithoutCancel(ctx), 3*time.Second), e)`.
Its errors are logged at WARN and never change the outcome.

### 3.4 Rip plan per mode and kind

| Kind | ModeNormal | ModeJustISO | ModeAlsoISO |
|---|---|---|---|
| BluRay | Rippers[BluRay] | ISO | Rippers[BluRay], then ISO |
| DVD | Rippers[DVD] | ISO | Rippers[DVD], then ISO |
| AudioCD | Rippers[AudioCD] | *skip*: WARN "audio CDs cannot be imaged", outcome `skipped` | Rippers[AudioCD] only; WARN "ISO skipped for audio CD" |
| Data | ISO | ISO | ISO (once) |

### 3.5 Output (`internal/output`)

```go
type Planner struct{ /* storage roots, timestamp flag, separate-finish flag, owner, mode */ }

func NewPlanner(cfg Settings) (*Planner, error)
func (p *Planner) Prepare(k disc.Kind, label string, now time.Time) (dir string, err error)
func (p *Planner) Finalize(k disc.Kind, dir string) error
func (p *Planner) Cleanup(ctx context.Context, dir string) error
```

- **Root per kind:** BluRay → `STORAGE_BD`; DVD → `STORAGE_DVD`; AudioCD → `STORAGE_CD`;
  Data and every ISO step → `STORAGE_DATA`.
- **Sanitise label:** replace every rune not in `[A-Za-z0-9 ._-]` with `_`; trim leading/trailing
  space and `.`; truncate to 100 bytes. If empty → `disc_<YYYYMMDD_HHMMSS>`.
- **Name:** `TIMESTAMPPREFIX` → `<YYYYMMDD_HHMMSS>_<label>`, else `<label>`. If `<root>/<name>`
  already exists, append `_<YYYYMMDD_HHMMSS>`. If it still exists, append `_2`, `_3`, …
- **Prepare** returns `<root>/<name>` (created with `MkdirAll`, mode 0o755). The one
  exception is AudioCD, which returns the staging dir `<STORAGE_CD>/.ripper-staging-<YYYYMMDD_HHMMSS>`.
- **ISO file name:** the ddrescue backend writes `<dir>/<base(dir)>.iso`.
- **Finalize:**
  - BluRay/DVD with `SEPARATERAWFINISH`: rename `<root>/<name>` → `<root>/finished/<name>`
    (`MkdirAll` the parent).
  - AudioCD: rename each child of staging except `.wav` into `<STORAGE_CD>/`, using the
    collision rule above, then remove staging.
  - Then chown + chmod the final path recursively (§5.4).
- **Cleanup:** `RemoveAll(dir)`.
- **All filesystem operations** go through `os.OpenRoot(root)` (`Root.MkdirAll`, `Root.Rename`,
  `Root.RemoveAll`, `Root.Chown`, `Root.Chmod`) so no label can escape the storage root.

### 3.6 Commands (exact argv)

| Purpose | Name | Args |
|---|---|---|
| detect | `makemkvcon` | `-r --cache=1 info disc:9999` |
| audio check | `cdparanoia` | `-d <DRIVE> -Q` |
| BD/DVD rip | `makemkvcon` | `--profile=<profilePath> -r --decrypt --minlength=<MINIMUMLENGTH> mkv disc:<Index> all <dir>` |
| audio rip | `abcde` | `-d <DRIVE> -c <perRipConf> -N -l` (**no `-x`**: the engine is the only ejector, D22) |
| ISO | `ddrescue` | `<DRIVE> <dir>/<base(dir)>.iso` |
| eject | `eject` | `-v <DRIVE>`; on error: wait 2 s, `sdparm --command=unlock <DRIVE>`, wait 1 s, `sdparm --command=eject <DRIVE>` (return the last error) |
| register | `makemkvcon` | `reg <KEY>` |

- **abcde per-rip config:** write a temp file containing the base conf (`$CONFIG_DIR/abcde.conf`
  if it exists, else the embedded default), followed by
  `\n# ripper overrides\nOUTPUTDIR=<dir>\nWAVOUTPUTDIR=<dir>/.wav\nEJECTCD=n\n`. Delete it after the rip.
- **profilePath:** `$CONFIG_DIR/default.mmcp.xml` if it exists; else the embedded default,
  written once at startup to `os.MkdirTemp("", "ripper-")`.

### 3.7 Notifications

| Kind | Title | Body |
|---|---|---|
| Success | `Ripped <label>` | `<kind> → <final dir>` |
| Failure | `Rip failed: <label>` | the error string |
| Stopped | `Ripper stopped` | the reason |

`notify/apprise` maps Success → `apprise.NotifySuccess`, Failure → `NotifyFailure`,
Stopped → `NotifyWarning`. The patchbay selects `notify/nop` when `APPRISE_URLS` is empty.

### 3.8 Startup sequence (`ripper serve`, before `lifecycle.Run`)

1. Load + validate config (§1). On failure print the joined error and exit 1.
2. `obs.Setup` with `ServiceName: "ripper"`, `ServiceVersion: buildinfo.Version()`,
   `LogLevel: cfg.LogLevel`, `LogOutput: tee` (§6.1).
3. Warn on removed variables (§1.2).
4. MakeMKV key: `env` source if `KEY` is set, else `forum`. On error, log WARN and skip steps 5–6.
5. `settings.conf` in `<UserHomeDir>/.MakeMKV/`: create the dir with mode 0700; read the lines;
   replace the line starting with `app_Key` or append `app_Key = "<key>"`; write with mode 0600.
   Keep every other line.
6. `Run(makemkvcon reg <key>)`; an error logs WARN and startup continues.
7. Resolve `profilePath` and the abcde base conf (§3.6).
8. `patchbay.Backends` → `patchbay.Spec` → `lifecycle.Run`.

The MakeMKV key is only ever logged via `obs.RedactAttr("key", k)`. Apprise URLs are never logged.

---

## 4. HTTP

`P` = normalised `WEB_PATH_PREFIX`. Every route on the API listener is registered with
`P` prepended. huma operations use `Path: P + "/api/v1/…"`; raw handlers use
`api.RawRoute("<METHOD> " + P + "/…", h)`. **Never** wrap the handler in `http.StripPrefix`
as middleware; it breaks otelhttp's route labels.

### 4.1 API listener (`API_ADDR`)

| Pattern | When | Handler | Success | Errors |
|---|---|---|---|---|
| `GET P/api/v1/status` | always | huma, OperationID `getStatus`, Tag `status` | 200 `engine.Status` | — |
| `GET P/api/v1/log` | always | huma, OperationID `getLog`, Tag `log`; query `lines` int default 100, min 1, max 1000 | 200 `LogResponse` | 422 (huma validation) |
| `DELETE P/api/v1/log` | always | huma, OperationID `clearLog`, Tag `log`; `os.Truncate(LOG_FILE, 0)` | 204 | 500 problem |
| `GET P/openapi.json` | docs | raw; `api.Huma.OpenAPI().MarshalJSON()` | 200 `application/json` | — |
| `GET P/openapi.yaml` | docs | raw; `api.Huma.OpenAPI().YAML()` | 200 `application/yaml` | — |
| `GET P/docs` | docs | raw; Scalar HTML (§4.4) | 200 | — |
| `GET P/docs/assets/` | docs | raw; `http.StripPrefix(P+"/docs/assets/", http.FileServerFS(scalarFS))` | 200 | 404 |
| `GET P/{$}` | !headless | raw; UI `index.html` with CSP nonce (§4.4), `Cache-Control: no-store` | 200 | — |
| `GET P/assets/` | !headless | raw; `http.StripPrefix(P+"/assets/", http.FileServerFS(uiAssets))`, `Cache-Control: public, max-age=31536000, immutable` | 200 | 404 |
| `GET P/favicon.ico` | !headless | raw | 200 | — |
| `GET P` | P ≠ "" | raw; 301 → `P/` | — | — |
| `GET /{$}` | P ≠ "" or headless | raw; 302 → `P/` (UI) or `P/docs` (headless + docs). Not registered if headless and no docs. | — | — |

`kit httpapi.Options`: `DocsEnabled: false` (ripper serves its own docs), plus `Title: "ripper"`,
`Version`, `Logger`, `TracerProvider`, `MeterProvider`, and `Middleware` (outermost first):
1. `http.NewCrossOriginProtection().Handler` (CSRF guard for `DELETE`);
2. basic auth (§4.3) when credentials are configured.

```go
type LogResponse struct {
	Lines []string `json:"lines"` // raw lines, newest first
	Size  int64    `json:"size"`  // bytes
	Large bool     `json:"large"` // size > 1_000_000
}
```

Tail algorithm: open the file; seek from EOF in 64 KiB chunks until N newlines are found
or the start is reached; split; reverse. A missing file returns `{lines: [], size: 0}`.

### 4.2 Admin listener (`ADMIN_ADDR`)

`httpapi.NewAdmin(AdminOptions{Addr, Readiness: ready, Registry: p.PromRegistry, PprofEnabled, Logger})`.
`ready := httpapi.NewReadiness()`;
`ready.Register("drive", func(ctx) error { _, err := os.Stat(cfg.Drive); return err })`.
Pass the same `ready` to `lifecycle.Spec.Readiness`. No auth on admin.

### 4.3 Basic auth middleware

Enabled iff `WEB_USERNAME` and `WEB_PASSWORD` are both non-empty. `u, p, ok := r.BasicAuth()`.
Compare `sha256.Sum256` of each against the configured values with `subtle.ConstantTimeCompare`.
On failure: `w.Header().Set("WWW-Authenticate", `Basic realm="Ripper", charset="UTF-8"`)` then
`httpapi.WriteProblem(w, http.StatusUnauthorized, "authentication required", httpapi.ProblemOptions{})`.
Never call `r.WithContext` or otherwise clone the request.

### 4.4 Content-Security-Policy (exact)

- **UI index:** `default-src 'none'; script-src 'self'; style-src 'self' 'nonce-<N>'; img-src 'self' data:; font-src 'self'; connect-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'`.
  `<N>` = 16 random bytes, base64 (`crypto/rand`), new on every response. The server replaces the
  placeholder `__CSP_NONCE__` in `index.html` with `<N>`; it appears in `<meta name="csp-nonce" content="__CSP_NONCE__">`.
- **Scalar `/docs`:** `default-src 'none'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'`.
- Both also set `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer`.

The Scalar page (`internal/apidocs/assets/docs.html`) loads `assets/standalone.js` and then
`assets/docs-init.js`. `docs-init.js` calls `Scalar.createApiReference('#app', {url: './openapi.json',
withDefaultFonts: false, telemetry: false, hideTestRequestButton: true, hideClientButton: true})`.
The page uses no inline script.

---

## 5. Process and filesystem primitives

### 5.1 `execrunner` (Linux only: file `execrunner_linux.go`)

```go
cmd := exec.CommandContext(ctx, c.Name, c.Args...)
cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
cmd.Cancel = func() error {
	err := syscall.Kill(-cmd.Process.Pid, syscall.SIGTERM)
	if errors.Is(err, syscall.ESRCH) {
		return os.ErrProcessDone
	}
	return err
}
cmd.WaitDelay = 5 * time.Second
lw := newLineWriter(r.logger, c.Tool) // io.Writer; splits on '\n'; flush on Close; lines > 64 KiB are split
cmd.Stdout, cmd.Stderr = lw, lw       // never StdoutPipe
err := cmd.Run()
lw.Close()
if ctx.Err() != nil && cmd.Process != nil {
	_ = syscall.Kill(-cmd.Process.Pid, syscall.SIGKILL) // reap stragglers; ESRCH ignored
}
```

Error mapping:
- `ctx.Err() != nil` → return `fmt.Errorf("%s: %w", c.Tool, ctx.Err())`.
- `*exec.ExitError` (via `errors.AsType`) → `&ExitError{Tool, Code: ee.ExitCode()}`.
- Anything else → wrap with the tool name.

Each log line: `logger.Info("tool output", "tool", c.Tool, "line", line)`.

### 5.2 Apprise wrapper

```go
func (n *Notifier) Notify(ctx context.Context, e notify.Event) error {
	ctx, cancel := context.WithTimeout(ctx, 15*time.Second)
	defer cancel()
	done := make(chan error, 1) // buffered: the goroutine never blocks if we time out
	go func() {
		done <- n.a.Send(e.Body, apprise.WithTitle(e.Title), apprise.WithNotifyType(typeOf(e.Kind)))
	}()
	select {
	case err := <-done:
		return err
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

The engine calls `Notify` sequentially (apprise-go keeps package-level HTTP state; never call it
concurrently). Build the client once: `a := apprise.New(); err := a.AddAll(urls...)`.

### 5.3 Exit codes (`cli.Execute(ctx) int`)

`0` success; `1` runtime or config error; `2` usage error. Root command: `SilenceUsage: true`,
`SilenceErrors: true`. The error is printed to stderr once. `main`:
`os.Exit(cli.Execute(context.Background()))`. `serve` passes the context to `lifecycle.Run`, which
owns SIGTERM/SIGINT. `detect` and `healthcheck` wrap their context in `signal.NotifyContext`.

### 5.4 Mode and ownership

- **`ParseMode(s string) (func(fs.FileMode, isDir bool) fs.FileMode, error)`**
  - Grammar: octal `^0?[0-7]{3,4}$`, or clauses separated by `,`, each `[ugoa]*[-+=][rwxX]*`.
  - Empty who = `a`. Umask is ignored (deviation D25).
  - `X` adds execute only to directories, or to files that already have any execute bit.
  - Table tests: `g+rw`, `u=rwx,g=rx,o=`, `a+X`, `0775`, `775`; invalid: `g+q`, `z+r`, ``.
- **Ownership:** `user.Lookup(FILEUSER)` → uid; on `user.UnknownUserError` use `FILEUSERID`.
  Same for the group via `user.LookupGroup` / `FILEGROUPID`. Apply recursively with
  `Root.Lchown` + `Root.Chmod` (directories and files) to the finalized path only (D26).

---

## 6. Logging and build info

### 6.1 Log tee

Open `LOG_FILE` with `O_WRONLY|O_CREATE|O_APPEND`, mode 0644. `LogOutput = io.MultiWriter(os.Stdout, bestEffort{f})`,
where `bestEffort.Write` always returns `len(p), nil` and logs a file error once to stderr.
Every record is JSON (kit `obs`). Ripper adds these attribute keys:
`tool`, `line`, `state`, `disc_kind`, `disc_label`, `dir`, `outcome`, `err`.

### 6.2 Version

`internal/buildinfo.Version()` returns `debug.ReadBuildInfo().Main.Version` (stamped from the git
tag since Go 1.24; `(devel)` otherwise) plus `vcs.revision` (short) when present. No `-ldflags -X`.
Docker builds must include `.git` in the build context (do not list it in `.dockerignore`).
