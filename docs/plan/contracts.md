# Contracts

Everything in this file is **normative**. When a task in [`phases.md`](phases.md) says
"implement X per C§n", copy names, signatures, values and orderings exactly. If something you
need is missing here, stop and ask; do not invent it.

Module path: `github.com/jacaudi/docker-ripper`. All packages below live under it.

---

## 1. Configuration

### 1.1 Rules

- **viper is the only config loader**, and only `internal/cli` imports it. Always create it with
  `v := viper.New()`; do not use the package-level viper instance.
- `v.SetEnvPrefix("RIPPER")`. For **every** row in §1.2: `v.SetDefault(key, default)` and
  `v.BindEnv(key)`. Do not call `v.AutomaticEnv()`.
- Flags: `serve` defines one flag, `--headless`. Bind it with
  `v.BindPFlag("headless", cmd.Flags().Lookup("headless"))` in `PersistentPreRunE`.
  Precedence: flag > env > default.
- **An empty env value counts as unset**, so the default applies. This is viper's default
  (`AllowEmptyEnv(false)`); do not change it, and test it.
- Decode with `v.Unmarshal(&cfg)`. Each field has a `mapstructure:"<key>"` tag. Viper's default
  hooks already turn strings into `time.Duration` and comma-separated strings into `[]string`.
- Booleans accept anything `strconv.ParseBool` accepts.
- After decoding, call `cfg.Validate()`. It returns `errors.Join` of **every** problem it finds.

### 1.2 Keys (complete list)

| Env | viper key / Go field | Type | Default | Validation / meaning |
|---|---|---|---|---|
| `RIPPER_DRIVES` | `drives` / `Drives` | []string | empty | **Optional filter.** Empty = use every drive MakeMKV discovers. Otherwise each entry is absolute; basenames unique |
| `RIPPER_OUTPUT_DIR` | `output_dir` / `OutputDir` | string | `/out/Ripper` | absolute. Fixed subdirs `BluRay/ DVD/ CD/ DATA/` |
| `RIPPER_CONFIG_DIR` | `config_dir` / `ConfigDir` | string | `/config` | absolute. Optional override `default.mmcp.xml`. The image sets `HOME=/config`, so `~/.MakeMKV` lives here too |
| `RIPPER_EJECT` | `eject` / `Eject` | bool | `true` | `false` = leave the disc for manual removal |
| `RIPPER_ISO_MODE` | `iso_mode` / `ISOMode` | string | `off` | one of `off`, `also`, `only` |
| `RIPPER_MIN_TITLE_LENGTH` | `min_title_length` / `MinTitleLength` | int (s) | `600` | ≥ 0; MakeMKV `--minlength` |
| `RIPPER_AUDIO_FORMATS` | `audio_formats` / `AudioFormats` | []string | `flac,mp3` | ≥ 1 entry, no duplicates; each one of `flac`, `mp3`, `opus`, `aac`, `alac` (passed to cyanrip `-o`) |
| `RIPPER_MAX_PARALLEL_JOBS` | `max_parallel_jobs` / `MaxParallelJobs` | int | `0` | ≥ 0; 0 = no cap (every drive may rip at once) |
| `RIPPER_UID` | `uid` / `UID` | int | `-1` | ≥ −1; −1 = don't change the owner |
| `RIPPER_GID` | `gid` / `GID` | int | `-1` | ≥ −1; −1 = don't change the group |
| `RIPPER_UMASK` | `umask` / `Umask` | string (octal) | `002` | `strconv.ParseUint(s, 8, 32)` ≤ `0o777` |
| `RIPPER_MAKEMKV_KEY` | `makemkv_key` / `MakeMKVKey` | string | `""` | empty or `^T-[A-Za-z0-9@_]{66}$`. Empty = fetch the beta key |
| `RIPPER_APPRISE_URLS` | `apprise_urls` / `AppriseURLs` | []string | empty | each accepted by `apprise.New().Add`. Empty = no notifications |
| `RIPPER_POLL_INTERVAL` | `poll_interval` / `PollInterval` | duration | `60s` | ≥ 1s |
| `RIPPER_LOG_LEVEL` | `log_level` / `LogLevel` | string | `info` | `debug\|info\|warn\|error` (`obs.ParseLevel`) |
| `RIPPER_API_ADDR` | `api_addr` / `APIAddr` | string | `:9090` | `net.SplitHostPort` ok |
| `RIPPER_ADMIN_ADDR` | `admin_addr` / `AdminAddr` | string | `:9091` | `net.SplitHostPort` ok; ≠ `APIAddr` |
| `RIPPER_HEADLESS` / `--headless` | `headless` / `Headless` | bool | `false` | no web UI when true |
| `RIPPER_WEB_PATH_PREFIX` | `web_path_prefix` / `WebPathPrefix` | string | `""` | normalised (§1.3) |
| `RIPPER_WEB_USERNAME` | `web_username` / `WebUsername` | string | `""` | both-or-neither with the password |
| `RIPPER_WEB_PASSWORD` | `web_password` / `WebPassword` | string | `""` | both-or-neither with the username |

**Constants** (not configurable):

| Constant | Value |
|---|---|
| detect timeout | 30 s |
| bad-response threshold (per drive and for the scanner) | 5 |
| log ring | 2000 records |
| job history | 50 finished jobs |
| health staleness | 3 × poll interval |

**Drive ID** = `filepath.Base(device)` (`/dev/sr0` → `sr0`). It appears in staging paths, job IDs,
the API, readiness, metric labels, log records and notification titles.

### 1.3 `RIPPER_WEB_PATH_PREFIX` normalisation

- `""` or `/` → `""`.
- Otherwise: exactly one leading `/`, no trailing `/`.
- Reject values containing `?`, `#`, `{`, `}` or whitespace.
- Examples: `ripper` → `/ripper`; `/ripper/` → `/ripper`; `/a/b` → `/a/b`.

---

## 2. Go types and interfaces (copy exactly)

### 2.1 `internal/disc` (types only)

```go
package disc

type State int

const (
	StateUnknown  State = iota // unparseable or unexpected reply
	StateEmpty                 // no disc
	StateOpen                  // tray open
	StateLoading               // disc spinning up
	StateInserted              // disc present; see Kind
)

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
	Index  int    `json:"index"`
	State  State  `json:"state"`
	Kind   Kind   `json:"kind"`
	Label  string `json:"label"`  // raw label as reported (NOT sanitised)
	Device string `json:"device"` // e.g. /dev/sr0
	Raw    string `json:"raw"`    // the DRV: line that produced this value
}

func (d Disc) DriveID() string // filepath.Base(d.Device)

// String and MarshalText return: "unknown","empty","open","loading","inserted"
func (s State) String() string
func (s State) MarshalText() ([]byte, error)

// String and MarshalText return: "none","bluray","dvd","cd","audio_cd","data"
func (k Kind) String() string
func (k Kind) MarshalText() ([]byte, error)
```

### 2.2 Seams (five; the interface lives in the seam package, backends in sub-packages)

```go
// internal/runner
package runner

type Cmd struct {
	Name string   // executable, resolved via PATH
	Args []string
	Dir  string   // working directory; "" = inherit
	Tool string   // log/metric attribute value, e.g. "makemkvcon"
}

type Runner interface {
	// Run streams stdout+stderr line by line to the logger (one record per line) and waits.
	Run(ctx context.Context, c Cmd) error
	// Output returns combined stdout+stderr (capped at 1 MiB).
	Output(ctx context.Context, c Cmd) ([]byte, error)
}

// ExitError reports a non-zero exit. Callers use errors.AsType[*runner.ExitError].
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
	// Detect scans once and returns every drive the system sees, in index order.
	// No returned Disc has Kind KindCD.
	Detect(ctx context.Context) ([]disc.Disc, error)
}
```

```go
// internal/rip
package rip

// Ripper rips disc d into dir. The engine created dir (a staging dir) and owns it.
// It returns the name the finalized directory should get ("" = keep base(dir)).
// Only the audio ripper returns a name (the album folder cyanrip created), because an
// audio CD has no label. On ctx cancellation it returns promptly; the engine deletes dir.
type Ripper interface {
	Rip(ctx context.Context, d disc.Disc, dir string) (name string, err error)
}
```

```go
// internal/eject
package eject

type Ejector interface {
	Eject(ctx context.Context, device string) error
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

### 2.3 Backends and helpers

Every backend is a **single shared instance**. The device comes from `disc.Disc.Device` or the
`device` argument, so nothing is built per drive.

| Package | Constructor / API | Implements |
|---|---|---|
| `internal/runner/execrunner` | `New(logger *slog.Logger, tracer trace.Tracer, meter metric.Meter) (*Runner, error)` | `runner.Runner` |
| `internal/detect/makemkv` | `New(r runner.Runner, timeout time.Duration) *Detector`; `func ParseDRV(out []byte) ([]disc.Disc, error)`; `ErrMalformed` | `detect.Detector` |
| `internal/rip/makemkv` | `New(r runner.Runner, configDir string, minLength int) (*Ripper, error)`; embeds `default.mmcp.xml` | `rip.Ripper` |
| `internal/rip/cyanrip` | `New(r runner.Runner, formats []string) *Ripper` | `rip.Ripper` |
| `internal/rip/ddrescue` | `New(r runner.Runner) *Ripper` | `rip.Ripper` |
| `internal/eject/execeject` | `New(r runner.Runner) *Ejector` | `eject.Ejector` |
| `internal/notify/apprise` | `New(urls []string) (*Notifier, error)` | `notify.Notifier` |
| `internal/notify/nop` | `type Notifier struct{}` | `notify.Notifier` |
| `internal/makemkvkey` | `FetchBetaKey(ctx, d outbound.Doer, url string) (string, error)`; `const ForumURL`; `Register(ctx, r runner.Runner, key string) error` | — (functions) |
| `internal/output` | `Planner` (§3.5) | — |
| `internal/logring` | `New(capacity int) *Ring`; `(*Ring).Write([]byte) (int, error)`; `(*Ring).Lines(n int) []string` | `io.Writer` |
| `internal/telemetry` | §6.5 | — |
| `internal/health` | §6.1 | — |

- **Name clash:** `detect/makemkv` and `rip/makemkv` share the package name `makemkv`. Import them
  in `internal/patchbay` as `makemkvdetect` and `makemkvrip`.
- **Compile-time assertion:** every backend declares one, e.g. `var _ rip.Ripper = (*Ripper)(nil)`.

### 2.4 Engine (drive watchers + job queue)

```go
// internal/engine
package engine

type ISOMode string

const (
	ISOOff  ISOMode = "off"
	ISOAlso ISOMode = "also"
	ISOOnly ISOMode = "only"
)

const (
	BadThreshold = 5
	HistoryLimit = 50
)

type DriveState string

const (
	DriveIdle            DriveState = "idle"
	DriveQueued          DriveState = "queued"
	DriveRipping         DriveState = "ripping"
	DriveEjecting        DriveState = "ejecting"
	DriveAwaitingRemoval DriveState = "awaiting_removal"
	DriveUnusable        DriveState = "unusable" // ≥ BadThreshold unknown replies in a row
)

type JobState string

const (
	JobQueued    JobState = "queued"
	JobRunning   JobState = "running"
	JobSucceeded JobState = "succeeded"
	JobFailed    JobState = "failed"
	JobCancelled JobState = "cancelled"
	JobSkipped   JobState = "skipped"
)

type Deps struct {
	Detect  detect.Detector
	Video   rip.Ripper // BluRay and DVD (makemkv)
	Audio   rip.Ripper // audio CD (cyanrip)
	ISO     rip.Ripper // ddrescue
	Eject   eject.Ejector
	Notify  notify.Notifier
	Output  *output.Planner
	Logger  *slog.Logger
	Tracer  trace.Tracer
	Metrics *Metrics         // §6.2, created once by the patchbay
	Now     func() time.Time // time.Now in prod
}

type Config struct {
	ISOMode      ISOMode
	Eject        bool
	PollInterval time.Duration
	MaxParallel  int      // 0 = no cap
	Include      []string // device paths; empty = all discovered drives
}

type Engine struct{ /* unexported; one sync.Mutex guards drives, queue and history */ }

func New(d Deps, c Config) *Engine
func (e *Engine) Run(ctx context.Context) error // the single lifecycle.Worker; returns nil on ctx cancel
func (e *Engine) Drives() []DriveStatus        // sorted by drive ID
func (e *Engine) Jobs() []Job                  // queued + running first (oldest first), then up to HistoryLimit finished (newest first)
func (e *Engine) Started() bool                // true after the first scan has completed (success or failure)
func (e *Engine) CheckScanner(ctx context.Context) error // error if the last successful scan is older than 3 × PollInterval, or scanner failures ≥ BadThreshold
func (e *Engine) CheckLoop(ctx context.Context) error    // error if the loop heartbeat is older than 3 × PollInterval
func (e *Engine) CheckDrives(ctx context.Context) error  // error if no discovered drive is in a state other than unusable

type DriveStatus struct {
	Drive        string     `json:"drive"`
	Device       string     `json:"device"`
	State        DriveState `json:"state"`
	Disc         *disc.Disc `json:"disc,omitempty"`
	JobID        string     `json:"job_id,omitempty"`
	BadResponses int        `json:"bad_responses"`
	LastSeen     time.Time  `json:"last_seen"`
}

type Job struct {
	ID         string     `json:"id"`    // "<drive>-<YYYYMMDDHHMMSS>"
	Drive      string     `json:"drive"`
	Disc       disc.Disc  `json:"disc"`
	State      JobState   `json:"state"`
	QueuedAt   time.Time  `json:"queued_at"`
	StartedAt  *time.Time `json:"started_at,omitempty"`
	FinishedAt *time.Time `json:"finished_at,omitempty"`
	Paths      []string   `json:"paths,omitempty"` // finalized, relative to OUTPUT_DIR
	Error      string     `json:"error,omitempty"`
}
```

---

## 3. Behaviour tables

### 3.1 DRV parsing (`makemkv.ParseDRV` in `detect/makemkv`)

1. Iterate lines with `strings.Lines`. Keep lines that start with `DRV:`.
2. Parse `strings.TrimPrefix(line, "DRV:")` with `csv.NewReader` (`LazyQuotes: true`,
   `FieldsPerRecord: -1`). Fewer than 7 fields → `ErrMalformed`.
3. Fields: `[0]` index (int), `[1]` state (int), `[2]` ignored, `[3]` flags (int),
   `[4]` drive name, `[5]` label, `[6]` device.
4. **Skip** lines whose device field is empty (absent drives, e.g. `DRV:5,256,999,0,"","",""`).
5. Classify each remaining line, in this order:

| state | flags | → State | → Kind |
|---|---|---|---|
| 0 | any | Empty | None |
| 1 | any | Open | None |
| 3 | any | Loading | None |
| 2 | 12 or 28 | Inserted | BluRay |
| 2 | 1 | Inserted | DVD |
| 2 | 0 | Inserted | CD (the detector resolves it) |
| anything else | | Unknown | None |

6. Return the discs in index order. Zero drives is **not** an error: return an empty slice.

An empty label never changes the classification.

### 3.2 Detector (`detect/makemkv`)

1. Call `Output(ctxWithTimeout(30s), {Name:"makemkvcon", Args:["-r","--cache=1","info","disc:9999"], Tool:"makemkvcon"})`.
   On error (including a timeout), return `nil, err`. That is a **scanner** failure.
2. `discs, err := ParseDRV(out)`. On error, return `nil, err`.
3. For each disc with `Kind == KindCD` or `State == StateEmpty`, run
   `Output(ctx, {Name:"cdparanoia", Args:["-d", device, "-Q"], Tool:"cdparanoia"})` and ignore its exit code.
   - Output contains `audio tracks` → State Inserted, Kind AudioCD.
   - Otherwise, if Kind was CD → Kind Data.
   - Otherwise, if State was Empty → it stays Empty.
4. Return the discs. DVD/BD **data** discs stay DVD/BluRay; this is a known limitation.

### 3.3 Engine

**`Run(ctx)`**
- Create a ticker: `time.NewTicker(cfg.PollInterval)`.
- Run one **scan** immediately, then one per tick. Each scan updates the loop heartbeat.
- On `ctx.Done()`:
  - stop the ticker;
  - mark every queued job `cancelled`;
  - wait for the running jobs (they see the cancelled ctx and clean up);
  - return `nil`.

**Scan**
1. `discs, err := Detect(ctx)`.
   - On error: `scanFailures++`; log WARN `detect failed`. When `scanFailures` first reaches
     `BadThreshold`: `notifyDetached(Failure, "Drive scan failing", err)`. Set `started = true`. End the scan.
   - On success: `scanFailures = 0`, `lastScanOK = Now()`, `started = true`.
2. If `cfg.Include` is non-empty, keep only the discs whose `Device` is in it.
3. **Drive discovery:**
   - For each disc whose drive ID has no watcher, create a watcher in state `idle` and log INFO
     `drive discovered`.
   - For each watcher whose drive is absent from `discs`: if it has no queued or running job,
     delete it and log INFO `drive removed`; otherwise keep it.
4. Call `observe(disc)` on each watcher (below).
5. Call `dispatch()`.

**`observe(d)` — per drive, in this order**
1. `State == Unknown`:
   - `bad++`; log WARN `detect failed` with `drive` and `raw`.
   - When `bad` first reaches `BadThreshold`: set the drive state to `unusable` and
     `notifyDetached(Failure, "Drive not responding (<drive>)", "<raw>")`.
   - Return.
2. Otherwise `bad = 0`. If the drive state is `unusable`, set it to `idle`.
3. If the drive has a queued or running job, update `LastSeen` and return.
4. **If `awaitingRemoval`:**
   - State Empty or Open → clear the flag, set state `idle`, log INFO `disc removed`.
   - Otherwise log DEBUG `waiting for disc removal`.
   - Return.
5. **State Empty, Open or Loading** → set state `idle`; return.
6. **State Inserted** → create `Job{ID: "<drive>-<YYYYMMDDHHMMSS>", State: queued, Disc: d, QueuedAt: Now()}`.
   Append it to the FIFO queue, set the drive state to `queued`, and log INFO `job queued`.

**`dispatch()`** (under the mutex): while the queue is non-empty and
(`MaxParallel == 0` or running < `MaxParallel`), pop the oldest job, mark it `running`, and start
`wg.Go(func() { e.execute(ctx, job) })`. It is called after every scan and after every finished job.

**`execute(ctx, job)`**
1. Set the drive state to `ripping`; `StartedAt = Now()`; start the `engine.job` span.
2. Run the rip plan (§3.4) step by step. For each step:
   - `dir, err := Output.Prepare(kindDir, job.Drive, job.Disc.Label, Now())`.
   - `name, err := ripper.Rip(ctx, job.Disc, dir)`.
   - **If `ctx.Err() != nil` (shutdown):**
     - `Output.Cleanup(cctx, dir)`, where `cctx` is `context.WithTimeout(context.WithoutCancel(ctx), 5*time.Second)`;
     - set the job `cancelled`;
     - `notifyDetached(Stopped, "Ripper stopped (<drive>)", "shutdown during rip of <label>")`;
     - **do not eject**; return.
   - If `err != nil`: `Output.Cleanup`, set the job `failed` with the error, and skip the remaining steps.
   - Else: `path, err := Output.Finalize(kindDir, dir, name)` and append `path`. A Finalize error is a failure.
3. If every step was skipped, the job is `skipped`. Otherwise it is `succeeded` if no step failed.
4. **Eject:**
   - If `cfg.Eject`: set the drive state to `ejecting` and call `Eject(ctx, device)`. An error is
     logged at ERROR and doesn't change the job state.
   - If `!cfg.Eject`: log INFO `safe to eject`.
5. Set `awaitingRemoval = true` and the drive state to `awaiting_removal`. This flag is what
   prevents re-rip loops after a failure, a failed eject, or when ejecting is off.
6. Notify: `Success` (succeeded) or `Failure` (failed); nothing for skipped. `FinishedAt = Now()`.
   Move the job to history (keep at most `HistoryLimit`). Call `dispatch()`.

`notifyDetached(e)` calls `Notify(ctx', e)`, where `ctx'` is
`context.WithTimeout(context.WithoutCancel(ctx), 3*time.Second)`. Its errors are logged at WARN and
never change a job.

All log records from a job carry `drive` and `job_id` (`logger.With(...)`).

### 3.4 Rip plan (`ISOMode` × Kind)

| Kind | `off` | `also` | `only` |
|---|---|---|---|
| BluRay | Video→`BluRay` | Video→`BluRay`, ISO→`DATA` | ISO→`DATA` |
| DVD | Video→`DVD` | Video→`DVD`, ISO→`DATA` | ISO→`DATA` |
| AudioCD | Audio→`CD` | Audio→`CD`; WARN `rip step skipped` (ISO of audio CD) | *skip*: WARN `rip step skipped` (audio CDs cannot be imaged) |
| Data | ISO→`DATA` | ISO→`DATA` (once) | ISO→`DATA` |

### 3.5 Output (`internal/output`)

```go
type Settings struct {
	OutputDir string
	UID, GID  int         // -1 = unchanged
	Umask     fs.FileMode // e.g. 0o002
}

func NewPlanner(s Settings) (*Planner, error) // os.OpenRoot(OutputDir); creates the 4 kind dirs
func (p *Planner) CleanStaging() error
func (p *Planner) Prepare(kindDir, drive, label string, now time.Time) (dir string, err error)
func (p *Planner) Finalize(kindDir, dir, name string) (path string, err error)
func (p *Planner) Cleanup(ctx context.Context, dir string) error
func (p *Planner) CheckWritable(ctx context.Context) error // create + remove "<OutputDir>/.healthcheck-<pid>"
func (p *Planner) Close() error
func Sanitize(s string) string // the rule below, without the empty-string fallback

const (
	DirBluRay = "BluRay"
	DirDVD    = "DVD"
	DirCD     = "CD"
	DirData   = "DATA"
)
```

- **One root.** Every filesystem call goes through the single `os.Root` opened at `OutputDir`.
  No label can escape it.
- **Concurrency:** several jobs share the Planner. `os.Root` is safe for concurrent use. `Finalize`
  holds a `sync.Mutex` for its check-then-rename.
- **Sanitise:** replace every rune not in `[A-Za-z0-9 ._-]` with `_`; trim leading/trailing spaces
  and dots; truncate to 100 bytes. Empty → `disc_<YYYYMMDD_HHMMSS>`.
- **Prepare:** returns the absolute path of `<kindDir>/.staging/<YYYYMMDD_HHMMSS>-<drive>/<name>`,
  where `<name>` is the sanitised label. It is created with mode 0o777, so the umask applies.
- **Finalize:**
  - Final name = `Sanitize(name)` if `name != ""` and that is non-empty, else `base(dir)`.
  - The target is `<kindDir>/<final>`. If it exists, use `<final>_<YYYYMMDD_HHMMSS>_<drive>`. If
    that also exists, fail.
  - One `Root.Rename(dir, target)`, then remove `<kindDir>/.staging/<ts>-<drive>`.
  - Apply permissions and ownership recursively to the target (§5.4).
  - Staging lives inside its kind dir, so a rename never crosses a mount.
- **Cleanup:** `Root.RemoveAll(<kindDir>/.staging/<ts>-<drive>)`.
- **CleanStaging:** remove `<kindDir>/.staging` for every kind dir. It is called once by `serve`,
  before the engine starts.

### 3.6 Commands (exact argv)

| Purpose | Name | Args |
|---|---|---|
| scan | `makemkvcon` | `-r --cache=1 info disc:9999` |
| audio check | `cdparanoia` | `-d <device> -Q` |
| BluRay/DVD rip | `makemkvcon` | `--profile=<profilePath> -r --decrypt --minlength=<MIN_TITLE_LENGTH> mkv disc:<Index> all <dir>` |
| audio rip | `cyanrip` | `-d <device> -o <formats joined by ",">` with `Cmd.Dir = <dir>` (retry rule below) |
| ISO | `ddrescue` | `<device> <dir>/<base(dir)>.iso <dir>/<base(dir)>.map` |
| eject | `eject` | `-v <device>`. On error: wait 2 s, `sdparm --command=unlock <device>`, wait 1 s, `sdparm --command=eject <device>`; return the last error |
| register | `makemkvcon` | `reg <KEY>` |

- **profilePath:** `<CONFIG_DIR>/default.mmcp.xml` if it exists. Otherwise the embedded default,
  written once by `rip/makemkv.New` to `os.MkdirTemp("", "ripper-")`.
- **cyanrip:**
  - It runs with its defaults: MusicBrainz lookup, Cover Art Archive cover, AccurateRip and EAC CRC
    verification, ReplayGain, and its default folder and file naming.
  - It never gets `-Q` (eject); the engine ejects.
  - **Retry rule:** if the first run exits non-zero, run it once more with `-N` added (no MusicBrainz
    lookup; placeholder names) and log WARN `cyanrip retry without musicbrainz`. This covers discs
    that aren't in MusicBrainz. The exact miss behaviour is checked on hardware in P2.3b.
  - **After success:** cyanrip has created exactly one directory `D` in `<dir>`. Move every entry
    of `<dir>/D` into `<dir>` (`os.Rename`, same directory tree), remove `D`, and return `name = D`.
    Zero or several directories → error.

### 3.7 Notifications

| Kind | Title | Body |
|---|---|---|
| Success | `Ripped <label or name> (<drive>)` | `<kind> → <paths joined by ", ">` |
| Failure | `Rip failed: <label> (<drive>)` / `Drive not responding (<drive>)` / `Drive scan failing` | the error string |
| Stopped | `Ripper stopped (<drive>)` | the reason |

- `notify/apprise` maps Success → `apprise.NotifySuccess`, Failure → `NotifyFailure`,
  Stopped → `NotifyWarning`.
- The patchbay selects `notify/nop` when `AppriseURLs` is empty.
- Automation after a rip uses an apprise `json://` or `form://` target.

### 3.8 Startup (`ripper serve`, before `lifecycle.Run`)

1. Load and validate config (§1). On failure, print the joined error and exit 1.
2. `syscall.Umask(int(cfg.Umask))` (Linux), before any file is created.
3. `logring.New(2000)`; `tel, err := telemetry.Setup(ctx, telemetry.Config{...}, ring)` (§6.5).
4. Get the key: `cfg.MakeMKVKey`, or if that's empty, `makemkvkey.FetchBetaKey` (client per §5.5).
   `os.MkdirAll(<home>/.MakeMKV, 0o700)`; `makemkvkey.Register`. Record the result for the
   `makemkv_registration` health check. Errors → log WARN and continue.
5. `output.NewPlanner` → `CleanStaging()`.
6. `patchbay.Backends` → `engine.New` → `patchbay.Spec` → `lifecycle.Run`.

`/startupz` turns OK once steps 1–6 have run **and** `engine.Started()` is true (§6.1).

Never log the key, the apprise URLs or the web password. `execrunner` never logs argv (§5.1).

---

## 4. HTTP

`P` is the normalised `RIPPER_WEB_PATH_PREFIX`.
- Every route on the API listener is registered with `P` prepended: huma operations use
  `Path: P + "/api/v1/…"`, raw handlers use `api.RawRoute("<METHOD> " + P + "/…", h)`.
- **Never** wrap the listener in `http.StripPrefix` as middleware.

### 4.1 API listener (`RIPPER_API_ADDR`)

| Pattern | When | Handler | Success | Errors |
|---|---|---|---|---|
| `GET P/api/v1/status` | always | huma; OperationID `getStatus`, Tag `status` | 200 `StatusResponse` | — |
| `GET P/api/v1/jobs` | always | huma; OperationID `listJobs`, Tag `jobs` | 200 `JobsResponse` | — |
| `GET P/api/v1/log` | always | huma; OperationID `getLog`, Tag `log`; query `lines` int, default 200, min 1, max 2000 | 200 `LogResponse` | 422 |
| `GET P/openapi.json` | always | raw; `api.Huma.OpenAPI().MarshalJSON()` | 200 `application/json` | — |
| `GET P/openapi.yaml` | always | raw; `api.Huma.OpenAPI().YAML()` | 200 `application/yaml` | — |
| `GET P/docs` | always | raw; Scalar HTML (§4.4) | 200 | — |
| `GET P/docs/assets/` | always | raw; `http.StripPrefix(P+"/docs/assets/", http.FileServerFS(scalarFS))` | 200 | 404 |
| `GET P/{$}` | !headless | raw; UI `index.html` with a CSP nonce (§4.4), `Cache-Control: no-store` | 200 | — |
| `GET P/assets/` | !headless | raw; `http.StripPrefix(P+"/assets/", http.FileServerFS(uiAssets))`, `Cache-Control: public, max-age=31536000, immutable` | 200 | 404 |
| `GET P/favicon.ico` | !headless | raw | 200 | — |
| `GET P` | P ≠ "" | raw; 301 → `P/` | — | — |
| `GET /{$}` | P ≠ "" or headless | raw; 302 → `P/` (UI) or `P/docs` (headless) | — | — |

```go
type StatusResponse struct {
	Drives  []engine.DriveStatus `json:"drives"`
	Queued  int                  `json:"queued"`
	Running int                  `json:"running"`
}

type JobsResponse struct {
	Jobs []engine.Job `json:"jobs"` // engine.Jobs() order
}

type LogResponse struct {
	Lines []string `json:"lines"` // raw JSON log records, newest first
}
```

`httpapi.Options` for the kit:
- `DocsEnabled: false`, `Title: "ripper"`, `Version`, `Logger`, `TracerProvider`, `MeterProvider`;
- `Middleware`, outermost first: `http.NewCrossOriginProtection().Handler`, then basic auth (§4.3)
  when credentials are set.

### 4.2 Admin listener (`RIPPER_ADMIN_ADDR`)

Built by `health.NewAdminServer` (§6.1). It reuses the kit's `/readyz` and `/metrics` handlers,
adds `/livez`, `/startupz` and a dependency-checking `/healthz`, and has no auth.

### 4.3 Basic auth middleware (in `internal/api`)

- Enabled iff both `RIPPER_WEB_USERNAME` and `RIPPER_WEB_PASSWORD` are non-empty.
- `u, p, ok := r.BasicAuth()`. Compare `sha256.Sum256` of each value against the configured ones
  with `subtle.ConstantTimeCompare`.
- On failure: `w.Header().Set("WWW-Authenticate", `Basic realm="Ripper", charset="UTF-8"`)`, then
  `httpapi.WriteProblem(w, http.StatusUnauthorized, "authentication required", httpapi.ProblemOptions{})`.
- Never clone the request (no `r.WithContext`).

### 4.4 Content-Security-Policy (exact)

**UI index:**
```
default-src 'none'; script-src 'self'; style-src 'self' 'nonce-<N>'; img-src 'self' data:; font-src 'self'; connect-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
```
- `<N>` is 16 bytes from `crypto/rand`, base64-encoded, new for every response.
- The server replaces `__CSP_NONCE__` in `index.html`
  (`<meta name="csp-nonce" content="__CSP_NONCE__">`) with `<N>`.

**Scalar `/docs`:**
```
default-src 'none'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
```

Both also set `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer`.

**Scalar page** (`internal/apidocs/assets/docs.html`):
- Loads `assets/standalone.js` and then `assets/docs-init.js`. No inline script.
- `docs-init.js` calls `Scalar.createApiReference('#app', {url: './openapi.json', withDefaultFonts: false, telemetry: false, hideTestRequestButton: true, hideClientButton: true})`.

---

## 5. Primitives

### 5.1 `execrunner` (Linux only: `execrunner_linux.go`)

```go
cmd := exec.CommandContext(ctx, c.Name, c.Args...)
cmd.Dir = c.Dir
cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
cmd.Cancel = func() error {
	err := syscall.Kill(-cmd.Process.Pid, syscall.SIGTERM)
	if errors.Is(err, syscall.ESRCH) {
		return os.ErrProcessDone
	}
	return err
}
cmd.WaitDelay = 5 * time.Second
lw := newLineWriter(ctx, r.logger, c.Tool) // io.Writer; splits on '\n'; flushes on Close; lines over 64 KiB are split
cmd.Stdout, cmd.Stderr = lw, lw            // never StdoutPipe
err := cmd.Run()
lw.Close()
if ctx.Err() != nil && cmd.Process != nil {
	_ = syscall.Kill(-cmd.Process.Pid, syscall.SIGKILL) // reap stragglers; ESRCH ignored
}
```

- Log DEBUG `exec` with `tool`. **Argv is never logged.**
- Each output line → `logger.InfoContext(ctx, "tool output", "tool", c.Tool, "line", line)`.
- Each Run/Output is wrapped in a `tool.run` span (§6.4) and records `ripper.tool.runs` and
  `ripper.tool.duration` (§6.2).
- Errors:
  - `ctx.Err() != nil` → `fmt.Errorf("%s: %w", c.Tool, ctx.Err())`;
  - `*exec.ExitError` (`errors.AsType`) → `&runner.ExitError{Tool, Code: ee.ExitCode()}`;
  - anything else → wrapped with the tool name.

### 5.2 Apprise wrapper

```go
func (n *Notifier) Notify(ctx context.Context, e notify.Event) error {
	ctx, cancel := context.WithTimeout(ctx, 15*time.Second)
	defer cancel()
	done := make(chan error, 1) // buffered: the goroutine never blocks if we time out
	go func() {
		n.mu.Lock() // apprise-go keeps package-level HTTP state: never Send concurrently
		defer n.mu.Unlock()
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

Build the client once: `a := apprise.New(); err := a.AddAll(urls...)`.

### 5.3 Exit codes (`cli.Execute(ctx) int`)

- `0` success; `1` runtime or config error; `2` usage error.
- Root command: `SilenceUsage: true`, `SilenceErrors: true`. Print the error to stderr once.
- `main` is `os.Exit(cli.Execute(context.Background()))`.
- `serve`: `lifecycle.Run` owns SIGTERM/SIGINT.
- `detect` and `healthcheck`: use `signal.NotifyContext`.

### 5.4 Permissions and ownership (`output`)

Let `u` be the umask.
- **Directories:** `0o777 &^ u`.
- **Files:** `0o777 &^ u` if the file already has any execute bit, else `0o666 &^ u`.
- Never add execute to a file.
- Apply recursively with `Root.Chmod`. Then, if `UID >= 0 || GID >= 0`, call
  `Root.Lchown(path, UID, GID)` (−1 leaves that part unchanged).

### 5.5 Outbound client for the key fetch

```go
// MakeMKV forum: public page, fetched once per process start; no API, no published quota.
// One attempt at 1 rps: a failed fetch only skips registration.
c, err := outbound.New(outbound.Config{Product: "ripper", Version: version(),
	ContactURL: "https://github.com/jacaudi/docker-ripper", Timeout: 15 * time.Second,
	RequestsPerSecond: 1, Burst: 1, MaxAttempts: 1})
```

- `FetchBetaKey`: GET the url; regex `T-[\w@]{66}`; return the first match; no match →
  `errors.New("makemkvkey: no beta key found")`.
- Known limitation: the key is fetched once per process start. Auto-released MakeMKV bumps (L§5)
  plus `restart: unless-stopped` cover rotation.

### 5.6 `ripper detect`

- Builds `execrunner` + `detect/makemkv`, calls `Detect` once, and prints a JSON array of every
  `disc.Disc` (indented) to stdout. `RIPPER_DRIVES` filters it, as in the engine.
- Exit 0 for any result, including no drives. Exit 1 on a detect error (to stderr).
- `--raw` also prints the raw `makemkvcon` output, then each raw `cdparanoia -Q` output, each under
  a `--- <tool> <device> ---` header. This is how hardware fixtures are captured.

### 5.7 Version

`version()` lives in `internal/cli/version.go`. It returns `debug.ReadBuildInfo().Main.Version`
(stamped from the git tag) plus the short `vcs.revision` when present. Do not use `-ldflags -X`.
Docker builds must include `.git` in the build context.

---

## 6. Observability

**Slot 0 is the default and needs no configuration:** Prometheus pull metrics, JSON logs on stdout,
and health endpoints. OTLP traces are exported when the standard `OTEL_*` variables are set. That
is go-service-kit's current behaviour; §6.5 explains how to extend it later.

### 6.1 Health probes (admin listener, `internal/health`)

| Endpoint | Question | Checks | 200 / 503 | Use |
|---|---|---|---|---|
| `GET /livez` | Is the process alive? | none (static) | always 200, `text/plain` `ok\n` | k8s `livenessProbe` |
| `GET /startupz` | Has startup finished? | `startup` | 200 once §3.8 has completed and `engine.Started()` | k8s `startupProbe` |
| `GET /healthz` | **Am I a healthy service?** (dependencies) | `tool:makemkvcon`, `tool:cyanrip`, `tool:cdparanoia`, `tool:ddrescue`, `tool:eject`, `tool:sdparm` (`exec.LookPath`); `output_dir` (`Planner.CheckWritable`); `config_dir` (create + remove a temp file); `makemkv_registration` (the startup result; fails if registration failed or there was no key); `scanner` (`engine.CheckScanner`); `engine_loop` (`engine.CheckLoop`) | 200 iff all pass | Docker `HEALTHCHECK` (`ripper healthcheck`), monitoring, alerts |
| `GET /readyz` | Am I ready to do work? | `started` (`engine.Started()`); `drives` (`engine.CheckDrives`); plus the kit's shutdown gate | 200 iff all pass and not draining | k8s `readinessProbe` |
| `GET /metrics` | — | — | Prometheus exposition (kit) | scrape |

- `/startupz`, `/healthz` and `/readyz` each use one `httpapi.NewReadiness()` instance, so they all
  return the kit's JSON schema:
  ```json
  {
    "status": "ok | failed | shutting_down",
    "checks": [
      { "name": "drives", "status": "ok" },
      { "name": "tool:cyanrip", "status": "failed", "error": "exec: \"cyanrip\": executable file not found in $PATH" }
    ]
  }
  ```
  Checks are sorted by name, each runs with a 2 s timeout, and the HTTP status is 200/503.
- Only the **readiness** instance is passed to `lifecycle.Spec.Readiness`, so only `/readyz` reports
  `shutting_down` during a drain.
- `health.NewAdminServer(addr string, h Handlers, logger *slog.Logger) *http.Server`:
  - `m := httpapi.AdminHandlers(httpapi.AdminOptions{Readiness: ready, Registry: promRegistry, Logger: logger})`;
  - **replace** `m["GET /healthz"]` with the health instance's handler;
  - add `"GET /livez"` (static `ok`) and `"GET /startupz"`;
  - register every entry on a new `http.ServeMux`;
  - build `&http.Server{Addr, Handler: mux, ReadHeaderTimeout, ReadTimeout, WriteTimeout, IdleTimeout, MaxHeaderBytes}`
    from `httpapi.Timeouts{}.WithDefaults()`.
- `ripper healthcheck` GETs `http://127.0.0.1:<port of RIPPER_ADMIN_ADDR>/healthz` with a 2 s
  timeout, prints the body, and exits 0 on 200, else 1.

This follows the Kubernetes probe convention (liveness / readiness / startup), plus a dependency
health endpoint as you defined it. Liveness deliberately checks nothing, so a dependency failure
(a missing tool, a full disk) shows up as unhealthy without causing a restart loop.

### 6.2 Metrics (Slot 0: Prometheus pull at `GET <ADMIN_ADDR>/metrics`)

**Provided by the kit:** Go runtime metrics, `service_build_info`, HTTP request/error/duration
metrics for the API listener, and `lifecycle_worker_restarts_total`.

**Ripper's own instruments.** Create them once in `internal/engine/metrics.go` with
`func NewMetrics(m metric.Meter) (*Metrics, error)`; execrunner creates its own two. Use the
instrument names and units exactly; do not add labels.

| Instrument (name, type, unit) | Exported name | Labels | Recorded by | When |
|---|---|---|---|---|
| `ripper.jobs.completed`, Int64Counter, `{job}` | `ripper_jobs_completed_total` | `drive`, `kind`, `state` | engine | job finished (succeeded/failed/cancelled/skipped) |
| `ripper.job.duration`, Float64Histogram, `s` | `ripper_job_duration_seconds` | `drive`, `kind`, `state` | engine | same; buckets `60,300,600,1200,1800,3600,5400,7200,10800` |
| `ripper.job.queue_wait`, Float64Histogram, `s` | `ripper_job_queue_wait_seconds` | — | engine | when a job starts; default buckets |
| `ripper.jobs`, Int64Gauge, `{job}` | `ripper_jobs` | `state` (`queued`, `running`) | engine | on every queue change |
| `ripper.drive.state`, Int64Gauge, `{state}` | `ripper_drive_state` | `drive`, `state` | engine | on every transition: 1 for the new state, 0 for the old one |
| `ripper.drive.bad_responses`, Int64Gauge, `{response}` | `ripper_drive_bad_responses` | `drive` | engine | each scan |
| `ripper.scans`, Int64Counter, `{scan}` | `ripper_scans_total` | `result` (`ok`, `error`) | engine | each scan |
| `ripper.drives`, Int64Gauge, `{drive}` | `ripper_drives` | — | engine | after discovery |
| `ripper.tool.runs`, Int64Counter, `{run}` | `ripper_tool_runs_total` | `tool`, `result` (`ok`, `exit_error`, `cancelled`, `error`) | execrunner | after each Run/Output |
| `ripper.tool.duration`, Float64Histogram, `s` | `ripper_tool_duration_seconds` | `tool` | execrunner | same; default buckets |
| `ripper.notifications`, Int64Counter, `{notification}` | `ripper_notifications_total` | `kind`, `result` (`ok`, `error`) | engine | after each Notify |

Every label value comes from a fixed set: drive IDs, the enums in §2, and tool names. Disc labels,
paths, job IDs and errors are **never** metric labels.

### 6.3 Logs (Slot 0: JSON on stdout)

One JSON object per line, from the kit's logger (slog JSON handler with redaction), teed into the
log ring for `GET /api/v1/log`.

| Field | Type | Present | Source |
|---|---|---|---|
| `time` | RFC 3339 with nanoseconds | always | slog |
| `level` | `DEBUG` \| `INFO` \| `WARN` \| `ERROR` | always | slog |
| `msg` | string, **static** (catalogue below) | always | caller |
| `service` | `"ripper"` | always | kit |
| `version` | build version | always | kit |
| `trace_id`, `span_id` | hex | inside a span | kit |
| `drive` | drive ID | drive and job records | engine |
| `job_id` | string | job records | engine |
| `state` | drive or job state | transitions | engine |
| `disc_kind`, `disc_label` | string | job records | engine |
| `path` | string (relative to OUTPUT_DIR) | `job finished` | engine |
| `tool` | string | runner records | execrunner |
| `line` | string | `tool output` | execrunner |
| `exit_code` | int | `tool failed` | execrunner |
| `duration_ms` | int | `job finished`, `tool finished` | engine, execrunner |
| `check` | string | `health check failed` | health |
| `error` | string | any failure | caller (`slog.Any("error", err)`) |

Rules:
- `msg` is a constant (sloglint `static-msg`); keys are snake_case.
- Always log through `…Context(ctx, …)` so `trace_id` is attached.
- Never log argv, the MakeMKV key, apprise URLs or the web password.

**Message catalogue** (the only allowed `msg` values; add new ones here first):

| msg | level |
|---|---|
| `ripper starting` | INFO |
| `config invalid` | ERROR |
| `makemkv key registration failed` | WARN |
| `staging cleaned` | INFO |
| `drive discovered` | INFO |
| `drive removed` | INFO |
| `detect failed` | WARN |
| `drive not responding` | ERROR |
| `disc detected` | INFO |
| `job queued` | INFO |
| `job started` | INFO |
| `job finished` | INFO |
| `job failed` | ERROR |
| `job cancelled` | WARN |
| `rip step skipped` | WARN |
| `cyanrip retry without musicbrainz` | WARN |
| `safe to eject` | INFO |
| `eject failed` | ERROR |
| `disc removed` | INFO |
| `waiting for disc removal` | DEBUG |
| `exec` | DEBUG |
| `tool output` | INFO |
| `tool finished` | DEBUG |
| `tool failed` | WARN |
| `notification failed` | WARN |
| `cleanup failed` | ERROR |
| `health check failed` | WARN |

### 6.4 Traces

Use the Tracer from `telemetry.Setup`. HTTP server spans come from the kit. Ripper adds:

| Span | Parent | Attributes | Status |
|---|---|---|---|
| `engine.job` | none (root; one per job) | `ripper.drive`, `ripper.job_id`, `ripper.disc.kind`, `ripper.job.state` | `Error` on failure |
| `rip.step` | `engine.job` | `ripper.ripper` (`video`, `audio`, `iso`), `ripper.kind_dir` | `Error` on failure |
| `tool.run` | the current span (from ctx) | `ripper.tool`, `process.exit.code` | `Error` on non-zero exit |

Scans are **not** traced (a span per minute would be noise).

### 6.5 Telemetry wiring (`internal/telemetry`) — current behaviour, built to extend

**Today** this matches go-service-kit `obs` exactly; no kit change is needed:

| Signal | Slot 0 (always) | OTLP |
|---|---|---|
| Traces | none (spans dropped) | exported when `OTEL_EXPORTER_OTLP_ENDPOINT` or `…_TRACES_ENDPOINT` is set (kit) |
| Metrics | Prometheus reader → admin `/metrics` | not yet |
| Logs | JSON → stdout (+ log ring) | not yet |

**Extension seam.** `internal/telemetry` is the **only** package that imports `obs` or any OTel
SDK/exporter. Everything else depends only on the OTel API (`trace.Tracer`, `metric.Meter`) and
`*slog.Logger`, which work with any exporter.

```go
package telemetry

type Config struct {
	ServiceName, ServiceVersion, LogLevel string
	SpanExporter sdktrace.SpanExporter // tests only; passed to obs.Config.SpanExporter. nil in production
}

type Telemetry struct {
	Logger         *slog.Logger
	Tracer         trace.Tracer
	Meter          metric.Meter
	TracerProvider trace.TracerProvider
	MeterProvider  metric.MeterProvider
	PromRegistry   *prometheus.Registry
	Shutdown       func(context.Context) error // flushes everything; wired as lifecycle.Spec.Flush
}

// Setup calls obs.Setup with LogOutput = io.MultiWriter(os.Stdout, ring) and adapts the result.
func Setup(ctx context.Context, c Config, ring io.Writer) (*Telemetry, error)
```

**To add OTLP metrics or logs later**, change only `telemetry.Setup` (or bump go-service-kit once
its `obs` supports it):
- Metrics: add a `sdkmetric.NewPeriodicReader(otlpmetric…)` next to the Prometheus reader when
  `OTEL_METRICS_EXPORTER` includes `otlp`.
- Logs: when `OTEL_LOGS_EXPORTER` includes `otlp`, fan out with
  `slog.NewMultiHandler(json, redact(otelslog.NewHandler(…)))`.

No instrumentation code changes in either case. This is recorded under "Future" in phases.md.
