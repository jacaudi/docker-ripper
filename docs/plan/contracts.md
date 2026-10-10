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
| `RIPPER_DRIVES` | `drives` / `Drives` | []string (comma-sep) | `/dev/sr0` | ≥ 1 entry; each absolute; basenames unique (the basename is the **drive ID**, e.g. `sr0`) |
| `RIPPER_OUTPUT_DIR` | `output_dir` / `OutputDir` | string | `/out/Ripper` | absolute. Fixed subdirs `BluRay/ DVD/ CD/ DATA/` |
| `RIPPER_CONFIG_DIR` | `config_dir` / `ConfigDir` | string | `/config` | absolute. Optional override `default.mmcp.xml`. The image sets `HOME=/config`, so `~/.MakeMKV` lives here too |
| `RIPPER_AUDIO_FORMATS` | `audio_formats` / `AudioFormats` | []string (comma-sep) | `flac,mp3` | ≥ 1 entry; each `flac` or `mp3`; no duplicates |
| `RIPPER_EJECT` | `eject` / `Eject` | bool | `true` | `false` = leave the disc for manual removal |
| `RIPPER_ISO_MODE` | `iso_mode` / `ISOMode` | string | `off` | one of `off`, `also`, `only` |
| `RIPPER_MIN_TITLE_LENGTH` | `min_title_length` / `MinTitleLength` | int (s) | `600` | ≥ 0; MakeMKV `--minlength` |
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

Constants (not configurable): detect timeout 30 s; drive-scan cache 5 s; bad-response threshold 5 (per drive); log ring 2000 records.

**Drive ID** = `filepath.Base(device)` (`/dev/sr0` → `sr0`). It appears in staging paths, the API,
readiness check names, metric labels, log records and notification titles.

### 1.3 `RIPPER_WEB_PATH_PREFIX` normalisation

- `""` or `/` → `""`.
- Otherwise: exactly one leading `/`, no trailing `/`.
- Reject values containing `?`, `#`, `{`, `}` or whitespace.
- Examples: `ripper` → `/ripper`; `/ripper/` → `/ripper`; `/a/b` → `/a/b`.

---

## 2. Go types and interfaces (copy exactly)

### 2.1 `internal/disc` (types only; no parsing here)

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
	Device string `json:"device"`
	Raw    string `json:"raw"`    // the DRV: line that produced this value
}

// String and MarshalText return: "unknown","empty","open","loading","inserted"
func (s State) String() string
func (s State) MarshalText() ([]byte, error)

// String and MarshalText return: "none","bluray","dvd","cd","audio_cd","data"
func (k Kind) String() string
func (k Kind) MarshalText() ([]byte, error)
```

### 2.2 Seams (six; the interface lives in the seam package, backends in sub-packages)

```go
// internal/runner
package runner

type Cmd struct {
	Name string   // executable, resolved via PATH
	Args []string
	Dir  string   // working directory; "" = inherit
	Tool string   // log attribute value, e.g. "makemkvcon"
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
	// Detect returns a Disc whose Kind is never KindCD.
	Detect(ctx context.Context) (disc.Disc, error)
}
```

```go
// internal/rip
package rip

// Ripper writes its output into dir. The engine created dir (a staging dir) and owns it.
// It returns the name the finalized directory should get ("" = keep the name derived from the
// disc label). Only the audio ripper returns a name ("<Artist> - <Album>"), because an audio CD
// has no label and its name is known only after the metadata lookup.
// On ctx cancellation it returns promptly; the engine deletes the staging dir.
type Ripper interface {
	Rip(ctx context.Context, d disc.Disc, dir string) (name string, err error)
}
```

```go
// internal/meta
package meta

type Track struct {
	Number int
	Title  string
	Artist string // equals Album.Artist unless Various
}

type Album struct {
	Artist, Title, Date string // Date: "YYYY" or "YYYY-MM-DD" or ""
	Various             bool
	Tracks              []Track // one per audio track, in order
	CoverURL            string  // "" = no cover
}

type Lookup interface {
	Lookup(ctx context.Context, toc cdda.TOC) (Album, error)
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

### 2.3 Backends and helpers

| Package | Constructor / API | Implements |
|---|---|---|
| `internal/runner/execrunner` | `New(logger *slog.Logger, tracer trace.Tracer, meter metric.Meter) (*Runner, error)` | `runner.Runner` |
| `internal/detect/makemkv` | `NewScanner(r runner.Runner, timeout time.Duration) *Scanner` (one per process, shared by all drives); `New(s *Scanner, r runner.Runner, device string) *Detector`; `func ParseDRV(out []byte, device string) (disc.Disc, error)`; `ErrNoDriveLine`, `ErrMalformed` | `detect.Detector` |
| `internal/rip/makemkv` | `New(r runner.Runner, configDir string, minLength int) (*Ripper, error)`; embeds `default.mmcp.xml` | `rip.Ripper` |
| `internal/cdda` | `type TOC struct{ First, Last int; Offsets []int; Leadout int }` (sector offsets, **without** the 150-sector lead-in); `ParseTOC(cdparanoiaQ []byte) (TOC, error)`; `(TOC) MusicBrainzID() string`; `(TOC) CDDBID() string` | — (pure) |
| `internal/meta/musicbrainz` | `New(d outbound.Doer, baseURL, coverURL string) *Lookup`; `const DefaultBaseURL = "https://musicbrainz.org/ws/2"`, `const DefaultCoverURL = "https://coverartarchive.org"` | `meta.Lookup` |
| `internal/meta/none` | `type Lookup struct{}`: Artist `Unknown Artist`, Title `Disc <CDDBID>`, tracks `Track NN` | `meta.Lookup` |
| `internal/meta` | `type Fallback struct{ Primary, Secondary Lookup; Logger *slog.Logger }`: on a Primary error, logs `metadata lookup failed` and returns Secondary | `meta.Lookup` |
| `internal/rip/audio` | `New(r runner.Runner, m meta.Lookup, d outbound.Doer, device string, formats []string) *Ripper` | `rip.Ripper` |
| `internal/rip/ddrescue` | `New(r runner.Runner, device string) *Ripper` | `rip.Ripper` |
| `internal/eject/execeject` | `New(r runner.Runner, device string) *Ejector` | `eject.Ejector` |
| `internal/notify/apprise` | `New(urls []string) (*Notifier, error)` | `notify.Notifier` |
| `internal/notify/nop` | `type Notifier struct{}` | `notify.Notifier` |
| `internal/makemkvkey` | `FetchBetaKey(ctx, d outbound.Doer, url string) (string, error)`; `const ForumURL`; `Register(ctx, r runner.Runner, key string) error` | — (functions) |
| `internal/output` | `Planner` (§3.5) | — |
| `internal/logring` | `New(capacity int) *Ring`; `(*Ring).Write([]byte) (int, error)`; `(*Ring).Lines(n int) []string` | `io.Writer` |

- **Name clash:** `detect/makemkv` and `rip/makemkv` share the package name `makemkv`. Import them
  in `internal/patchbay` as `makemkvdetect` and `makemkvrip`.
- **Compile-time assertion:** every backend declares one, e.g. `var _ rip.Ripper = (*Ripper)(nil)`.

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
	StateAwaitingRemoval State = "awaiting_removal"
	StateStopped         State = "stopped"
)

type ISOMode string

const (
	ISOOff  ISOMode = "off"
	ISOAlso ISOMode = "also"
	ISOOnly ISOMode = "only"
)

const BadThreshold = 5

// Deps for ONE drive. Detect, Video, Audio, ISO and Eject are built per drive;
// Notify, Output, Logger, Tracer and Metrics are shared by every engine.
type Deps struct {
	Detect  detect.Detector
	Video   rip.Ripper // BluRay and DVD
	Audio   rip.Ripper // audio CD
	ISO     rip.Ripper // ddrescue
	Eject   eject.Ejector
	Notify  notify.Notifier
	Output  *output.Planner
	Logger  *slog.Logger
	Tracer  trace.Tracer
	Metrics *Metrics // §6.2 instruments, created once by the patchbay
	Now     func() time.Time // time.Now in prod
}

type Config struct {
	Drive        string // drive ID, e.g. "sr0"
	ISOMode      ISOMode
	Eject        bool
	PollInterval time.Duration
}

type Engine struct{ /* unexported; mutex-guarded status */ }

func New(d Deps, c Config) *Engine
func (e *Engine) Run(ctx context.Context) error   // lifecycle.Worker.Run; returns nil on ctx cancel
func (e *Engine) Status() Status                  // safe for concurrent use
func (e *Engine) Ready(ctx context.Context) error // readiness check "detector": error iff bad ≥ BadThreshold

type Status struct {
	Drive           string      `json:"drive"` // drive ID
	Device          string      `json:"device"`
	State           State       `json:"state"`
	Disc            *disc.Disc  `json:"disc,omitempty"`
	StartedAt       *time.Time  `json:"started_at,omitempty"`
	LastResult      *LastResult `json:"last_result,omitempty"`
	BadResponses    int         `json:"bad_responses"`
	AwaitingRemoval bool        `json:"awaiting_removal"`
}

type LastResult struct {
	Label      string    `json:"label"`
	Kind       string    `json:"kind"`
	Outcome    string    `json:"outcome"` // "success" | "failure" | "cancelled" | "skipped"
	Error      string    `json:"error,omitempty"`
	Paths      []string  `json:"paths,omitempty"` // finalized paths, relative to OUTPUT_DIR
	FinishedAt time.Time `json:"finished_at"`
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
4. Use the first line whose `[6] == device`. No such line → `ErrNoDriveLine`.
5. Classify, in this order:

| state | flags | → State | → Kind |
|---|---|---|---|
| 0 | any | Empty | None |
| 1 | any | Open | None |
| 3 | any | Loading | None |
| 2 | 12 or 28 | Inserted | BluRay |
| 2 | 1 | Inserted | DVD |
| 2 | 0 | Inserted | CD (the detector resolves it) |
| anything else | | Unknown | None |

An empty label never changes the classification.

### 3.2 Detector (`detect/makemkv`)

1. `out, err := scanner.Scan(ctx)`. The `Scanner` runs
   `Output(ctxWithTimeout(30s), {Name:"makemkvcon", Args:["-r","--cache=1","info","disc:9999"], Tool:"makemkvcon"})`
   under a mutex and caches the result (output and error) for 5 s, so N drives polling on the same
   tick cause one MakeMKV scan. If `err != nil`, return `(Disc{State: StateUnknown, Device: device}, err)`.
2. Call `ParseDRV(out, device)`. If it returns an error, return `StateUnknown` and the error.
3. If `Kind == KindCD` **or** `State == StateEmpty`, run
   `Output(ctx, {Name:"cdparanoia", Args:["-d",device,"-Q"], Tool:"cdparanoia"})` and ignore its exit code.
   - Output contains `audio tracks` → State Inserted, Kind AudioCD.
   - Otherwise, if Kind was CD → Kind Data.
   - Otherwise, if State was Empty → it stays Empty.
4. Return the Disc. DVD/BD **data** discs stay DVD/BluRay; this is a known limitation.

### 3.3 Engine loop (`engine.Run`)

**At start** (staging was already cleaned once by `serve`, before any engine started; C§3.8)
- Create a ticker: `time.NewTicker(cfg.PollInterval)`.
- Run one pass immediately, then one pass per tick, until `ctx.Done()`; then return `nil`.

**Each pass**

1. Set state to detecting. Call `Detect(ctx)`.
2. **Detect error, or `StateUnknown`:**
   - `bad++`; log WARN with `err` and `raw`.
   - When `bad` first reaches `BadThreshold`: `notifyDetached(Failure, "Drive not responding (<drive>)", err)`.
   - While `bad >= BadThreshold`, `Ready` returns an error.
   - Keep `awaitingRemoval` unchanged, set state to idle (or awaiting_removal), end the pass.
   - Never eject here, and never exit.
3. Otherwise set `bad = 0`.
4. **If `awaitingRemoval`:**
   - State Empty or Open → clear the flag, log INFO `"disc removed"`, set state idle, end the pass.
   - Anything else → log DEBUG `"waiting for disc removal"`, set state awaiting_removal, end the pass.
5. **State Empty, Open or Loading** → log DEBUG, set state idle, end the pass.
6. **State Inserted:** set state ripping, `StartedAt = Now()`, then run the rip plan (§3.4) step by step.
   For each step:
   - `dir, err := Output.Prepare(kindDir, cfg.Drive, d.Label, Now())`.
   - `name, err := ripper.Rip(ctx, d, dir)`.
   - If `ctx.Err() != nil` (shutdown):
     - `Output.Cleanup(dir)` using `context.WithTimeout(context.WithoutCancel(ctx), 5*time.Second)`;
     - record outcome `cancelled`;
     - `notifyDetached(Stopped, "Ripper stopped", "shutdown during rip of <label>")`;
     - **do not eject**; return `nil`.
   - If `err != nil`: `Output.Cleanup(dir)`, record outcome `failure`, skip the remaining steps.
   - Else: `path, err := Output.Finalize(kindDir, dir, name)`. Append `path` to the result. A Finalize
     error counts as `failure`.
7. **After the plan:**
   - If `cfg.Eject`: set state ejecting and call `Eject(ctx)`. An error is logged at ERROR and does
     not change the outcome.
   - If `!cfg.Eject`: log INFO `"safe to eject"`.
   - Either way set `awaitingRemoval = true` and state awaiting_removal. This flag is what stops
     a disc being re-ripped every tick when the eject fails, when ejecting is off, or after a failure.
8. Notify `Success` if every step succeeded; `Failure` if any step failed. A plan that is only
   skipped steps sends no notification.

Every engine instruments itself per §6.2 (metrics) and §6.4 (spans), and adds `drive` to every log record
(`logger.With("drive", cfg.Drive)`).

`notifyDetached(e)` calls `Notify(ctx', e)`, where `ctx'` is
`context.WithTimeout(context.WithoutCancel(ctx), 3*time.Second)`. Its errors are logged at WARN
and never change the outcome. All notifications are sent sequentially from the engine goroutine.

### 3.4 Rip plan (`ISOMode` × Kind)

| Kind | `off` | `also` | `only` |
|---|---|---|---|
| BluRay | Video→`BluRay` | Video→`BluRay`, ISO→`DATA` | ISO→`DATA` |
| DVD | Video→`DVD` | Video→`DVD`, ISO→`DATA` | ISO→`DATA` |
| AudioCD | Audio→`CD` | Audio→`CD`; WARN "ISO skipped for audio CD" | *skip*: WARN "audio CDs cannot be imaged"; outcome `skipped` |
| Data | ISO→`DATA` | ISO→`DATA` (once) | ISO→`DATA` |

`X→Dir` means: call ripper X with a staging dir under kind dir `Dir`.

### 3.5 Output (`internal/output`)

```go
type Settings struct {
	OutputDir string
	UID, GID  int         // -1 = unchanged
	Umask     fs.FileMode // e.g. 0o002
}

func NewPlanner(s Settings) (*Planner, error)                   // os.OpenRoot(OutputDir); creates the 4 kind dirs
func (p *Planner) CleanStaging() error
func (p *Planner) Prepare(kindDir, drive, label string, now time.Time) (dir string, err error)
func (p *Planner) Finalize(kindDir, dir, name string) (path string, err error)
func Sanitize(s string) string // the rule below, without the empty-string fallback
func (p *Planner) Cleanup(ctx context.Context, dir string) error
func (p *Planner) Close() error

const (
	DirBluRay = "BluRay"
	DirDVD    = "DVD"
	DirCD     = "CD"
	DirData   = "DATA"
)
```

- **One root.** Every filesystem call goes through the single `os.Root` opened at `OutputDir`
  (`Root.MkdirAll`, `Root.Rename`, `Root.RemoveAll`, `Root.Lchown`, `Root.Chmod`, `Root.Stat`).
  No label can escape it.
- **Sanitise:** replace every rune not in `[A-Za-z0-9 ._-]` with `_`; trim leading/trailing spaces
  and dots; truncate to 100 bytes. If the result is empty, use `disc_<YYYYMMDD_HHMMSS>`.
- **Concurrency:** several engines share one Planner. `os.Root` is safe for concurrent use; `Finalize`
  holds a `sync.Mutex` for its check-then-rename so two drives can't claim the same target.
- **Prepare:** returns the absolute path of `<kindDir>/.staging/<YYYYMMDD_HHMMSS>-<drive>/<name>`, created
  with mode 0o777 (the umask applies). `<name>` is the sanitised label, so the ISO backend can name
  its files `<name>.iso` / `<name>.map`.
- **Finalize** (same for every kind):
  - Final name = `Sanitize(name)` if `name != ""` and that is non-empty, else `base(dir)`.
  - The target is `<kindDir>/<final>`. If that exists, use `<final>_<YYYYMMDD_HHMMSS>_<drive>`.
    If that also exists, fail with an error.
  - One `Root.Rename(dir, target)`, then remove `<kindDir>/.staging/<ts>-<drive>`.
  - Then apply ownership and permissions recursively to the finalized path (§5.4).
  - Staging lives inside its kind dir, so a rename never crosses a mount (no `EXDEV` when kind dirs
    are separate bind mounts).
- **Cleanup:** `Root.RemoveAll(<kindDir>/.staging/<ts>-<drive>)`.

### 3.6 Commands (exact argv)

| Purpose | Name | Args |
|---|---|---|
| detect | `makemkvcon` | `-r --cache=1 info disc:9999` |
| audio check | `cdparanoia` | `-d <DRIVE> -Q` |
| BluRay/DVD rip | `makemkvcon` | `--profile=<profilePath> -r --decrypt --minlength=<MIN_TITLE_LENGTH> mkv disc:<Index> all <dir>` |
| audio extract | `cdparanoia` | `-d <DRIVE> -q -B` with `Cmd.Dir = <dir>/.wav` (writes `trackNN.cdda.wav`) |
| FLAC encode | `flac` | `-s -8 -f -T TITLE=<t> -T ARTIST=<a> -T ALBUM=<al> -T DATE=<d> -T TRACKNUMBER=<n> -T TRACKTOTAL=<N> [--picture=<dir>/cover.jpg] -o <dir>/<file>.flac <wav>` |
| MP3 encode | `lame` | `--quiet -V2 --tt <t> --ta <a> --tl <al> --ty <d> --tn <n>/<N> [--ti <dir>/cover.jpg] <wav> <dir>/<file>.mp3` |
| ISO | `ddrescue` | `<DRIVE> <dir>/<base(dir)>.iso <dir>/<base(dir)>.map` |
| eject | `eject` | `-v <DRIVE>`. On error: wait 2 s, `sdparm --command=unlock <DRIVE>`, wait 1 s, `sdparm --command=eject <DRIVE>`; return the last error |
| register | `makemkvcon` | `reg <KEY>` |

- **profilePath:** `<CONFIG_DIR>/default.mmcp.xml` if it exists. Otherwise the embedded default,
  written once by `rip/makemkv.New` to `os.MkdirTemp("", "ripper-")`.
- The profile is resolved once, at construction. Restart to pick up edits.
- Optional `-T`/`--t*` values that are empty are omitted (no empty tags).

### 3.6a Audio ripper (`rip/audio`)

1. `out := Output(ctx, cdparanoia -d <DRIVE> -Q)`, then `toc, err := cdda.ParseTOC(out)`. Parse the
   table between the `===` line and `TOTAL`. Columns: `N.`, length, `[mm:ss.ff]`, begin, `[…]`, …
   `Leadout = begin(last) + length(last)`. Only audio tracks are listed, so for enhanced CDs the
   disc ID covers the audio session only (documented limitation).
2. `album := meta.Lookup(ctx, toc)`. The patchbay always wraps with `meta.Fallback{MusicBrainz, none}`,
   so a lookup failure never fails the rip. If `len(album.Tracks) != toc.Last-toc.First+1`, use
   `none`'s tracks with the album's artist and title.
3. If `album.CoverURL != ""`: GET it through the outbound Doer into `<dir>/cover.jpg` (max 10 MiB).
   Failure → log `cover fetch failed`, continue without a cover.
4. `mkdir <dir>/.wav`; run the extract command (§3.6).
5. For each track `n` (sequentially) and each format in `RIPPER_AUDIO_FORMATS`, in that order: run the
   encode command. File base name:
   - normal: `NN. <Title>`;
   - Various: `NN. <Artist> - <Title>`.
   
   Each part is passed through `output.Sanitize`; `NN` is two digits.
6. Remove `<dir>/.wav`. Keep `cover.jpg`.
7. Return the name `<Artist> - <Album>`, or `Various - <Album>` when `Various`.

**MusicBrainz backend:**
- GET `<base>/discid/<MusicBrainzID>?inc=recordings+artist-credits&fmt=json`. 404 or no releases → error.
- Pick the first release whose media list contains this disc ID. Use the matching medium's tracks
  (title, credited artist).
- `Various` is true when the release artist is `Various Artists` or any track artist differs from the release artist.
- `Date` is the release date. `CoverURL` = `<coverURL>/release/<release-id>/front-500`.
- Client: kit `outbound` with `Product: "ripper"`, the contact URL, `RequestsPerSecond: 1`, `Burst: 1`,
  `MaxAttempts: 2`, and `Timeout: 15s`. Put a comment at the construction site: MusicBrainz requires
  an identifying User-Agent and at most 1 req/s.

**Disc IDs** (`cdda`):
- `MusicBrainzID`: SHA-1 over the uppercase hex string `%02X` First, `%02X` Last, then 100 × `%08X`
  offsets (index 0 = Leadout+150; 1..99 = track offsets+150, zero for absent tracks). Base64-encode
  it (standard alphabet) and replace `+`→`.`, `/`→`_`, `=`→`-`.
- `CDDBID`: the classic freedb algorithm (digit sum of each track's start seconds mod 255, total
  seconds, track count), as `%08x`.
- Tests: use at least three published test vectors (TOC → MusicBrainz ID and CDDB ID) from the
  libdiscid test suite or the MusicBrainz Disc ID documentation, and cite the source in a comment.
  **Do not invent vectors.**

### 3.7 Notifications

| Kind | Title | Body |
|---|---|---|
| Success | `Ripped <label> (<drive>)` | `<kind> → <paths joined by ", ">` |
| Failure | `Rip failed: <label> (<drive>)` / `Drive not responding (<drive>)` | the error string |
| Stopped | `Ripper stopped (<drive>)` | the reason |

- `notify/apprise` maps Success → `apprise.NotifySuccess`, Failure → `NotifyFailure`,
  Stopped → `NotifyWarning`.
- The patchbay selects `notify/nop` when `AppriseURLs` is empty.
- For automation after a rip (moving files, triggering Plex, …) point an apprise `json://` or
  `form://` target at it. This replaces the old hook scripts.

### 3.8 Startup (`ripper serve`, before `lifecycle.Run`)

1. Load and validate config (§1). On failure, print the joined error and exit 1.
2. Call `syscall.Umask(int(cfg.Umask))` (Linux), before any file is created.
3. `logring.New(2000)`; `obs.Setup` with `ServiceName: "ripper"`, `ServiceVersion: version()`,
   `LogLevel`, and `LogOutput: io.MultiWriter(os.Stdout, ring)`.
4. Get the key: `cfg.MakeMKVKey`, or if that's empty, `makemkvkey.FetchBetaKey(ctx, client, makemkvkey.ForumURL)`
   (client per §5.5). `os.MkdirAll(<home>/.MakeMKV, 0o700)`. Then `makemkvkey.Register`.
   Any error here: log WARN and continue.
5. `output.NewPlanner` → `CleanStaging()` (once, before any engine starts).
6. `patchbay.Backends` builds one `engine.Deps` per drive (sharing one Scanner, Planner, Notifier,
   metadata Lookup (`meta.Fallback{musicbrainz, none}` with its own outbound client, C§3.6a), Logger, Tracer and Metrics) → one `*engine.Engine` per drive → `patchbay.Spec` → `lifecycle.Run`.

Never log the key, the apprise URLs, or the web password. `execrunner` never logs argv (§5.1),
so `makemkvcon reg <key>` is safe to run through it.

---

## 4. HTTP

`P` is the normalised `RIPPER_WEB_PATH_PREFIX`.
- Every route on the API listener is registered with `P` prepended: huma operations use
  `Path: P + "/api/v1/…"`, raw handlers use `api.RawRoute("<METHOD> " + P + "/…", h)`.
- **Never** wrap the listener in `http.StripPrefix` as middleware. It breaks otelhttp route labels.

### 4.1 API listener (`RIPPER_API_ADDR`)

| Pattern | When | Handler | Success | Errors |
|---|---|---|---|---|
| `GET P/api/v1/status` | always | huma; OperationID `getStatus`, Tag `status` | 200 `StatusResponse` | — |
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
	Drives []engine.Status `json:"drives"` // in RIPPER_DRIVES order
}

type LogResponse struct {
	Lines []string `json:"lines"` // raw JSON log records, newest first
}
```

`httpapi.Options` for the kit:
- `DocsEnabled: false` (ripper serves its own docs), `Title: "ripper"`, `Version`, `Logger`,
  `TracerProvider`, `MeterProvider`;
- `Middleware`, outermost first: `http.NewCrossOriginProtection().Handler`, then basic auth (§4.3)
  when credentials are set.

### 4.2 Admin listener (`RIPPER_ADMIN_ADDR`)

```go
ready := httpapi.NewReadiness()
for i, dev := range cfg.Drives {
	id := filepath.Base(dev)
	ready.Register("drive:"+id, func(ctx context.Context) error { _, err := os.Stat(dev); return err })
	ready.Register("detector:"+id, engines[i].Ready)
}
```

Build the admin server with `httpapi.NewAdmin(AdminOptions{Addr, Readiness: ready, Registry: p.PromRegistry, Logger})`.
Pass the same `ready` as `lifecycle.Spec.Readiness`. pprof is off. The admin listener has no auth.

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
cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
cmd.Cancel = func() error {
	err := syscall.Kill(-cmd.Process.Pid, syscall.SIGTERM)
	if errors.Is(err, syscall.ESRCH) {
		return os.ErrProcessDone
	}
	return err
}
cmd.WaitDelay = 5 * time.Second
lw := newLineWriter(r.logger, c.Tool) // io.Writer; splits on '\n'; flushes on Close; lines over 64 KiB are split
cmd.Stdout, cmd.Stderr = lw, lw       // never StdoutPipe
err := cmd.Run()
lw.Close()
if ctx.Err() != nil && cmd.Process != nil {
	_ = syscall.Kill(-cmd.Process.Pid, syscall.SIGKILL) // reap stragglers; ESRCH ignored
}
```

- Log a DEBUG `"exec"` record with `tool` before running. Argv is **not** logged at any level.
- Each output line → `logger.InfoContext(ctx, "tool output", "tool", c.Tool, "line", line)`.
- Each Run/Output is wrapped in a `tool.run` span (§6.4) and records `ripper.tool.runs` / `ripper.tool.duration` (§6.2).
- Constructor: `execrunner.New(logger *slog.Logger, tracer trace.Tracer, meter metric.Meter) (*Runner, error)`.
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

- Build the client once: `a := apprise.New(); err := a.AddAll(urls...)`.
- Engines for several drives can notify at the same time, but apprise-go keeps package-level HTTP
  state and must not be called concurrently. The `Notifier` holds a `sync.Mutex` around `Send`
  (inside the goroutine), so calls are serialised.

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
- Apply recursively to each finalized path with `Root.Chmod`. Then, if `UID >= 0 || GID >= 0`,
  call `Root.Lchown(path, UID, GID)` (−1 leaves that part unchanged).

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
- Known limitation: the key is fetched once per process start. A long-running container needs a
  restart after the beta key rotates. Auto-released MakeMKV bumps (L§5) plus `restart: unless-stopped`
  covers this.

### 5.6 `ripper detect`

- Builds `execrunner`, one `Scanner` and a `Detector` per drive in `RIPPER_DRIVES`, calls `Detect` on
  each, and prints a JSON array of `disc.Disc` (in `RIPPER_DRIVES` order) to stdout, indented.
- Exit 0 for any State, including Empty. Exit 1 if any drive returns a detect error (errors go to stderr).
- `--raw` also prints the raw `makemkvcon` output, then the raw `cdparanoia -Q` output, each under a
  `--- <tool> ---` header. This is how hardware fixtures are captured.

### 5.7 Version

`version()` lives in `internal/cli/version.go`. It returns `debug.ReadBuildInfo().Main.Version`
(stamped from the git tag) plus the short `vcs.revision` when present. Do not use `-ldflags -X`.
Docker builds must include `.git` in the build context.

---

## 6. Observability

**Slot 0 is the default and needs no configuration:** Prometheus pull metrics and JSON logs on
stdout. OTLP export (§6.5) is added on top only when the standard `OTEL_*` variables ask for it.

### 6.1 Health checks (admin listener, provided by go-service-kit)

| Endpoint | Meaning | Response | k8s probe | Docker |
|---|---|---|---|---|
| `GET /healthz` | **Liveness**: the process is running and not wedged. It never checks dependencies, so a drive problem never restarts the container. | `200`, `text/plain`, body `ok\n`. Always, including while draining. | `livenessProbe` | `HEALTHCHECK` via `ripper healthcheck` |
| `GET /readyz` | **Readiness**: should traffic be routed here? | `200` when every check passes, otherwise `503`. JSON schema below. | `readinessProbe` | — |

No startup probe: the listeners start within seconds of process start.

`/readyz` body (the kit's `httpapi.ReadinessReport`; checks are sorted by name; each runs with a 2 s timeout):

```json
{
  "status": "ok | failed | shutting_down",
  "checks": [
    { "name": "detector:sr0", "status": "ok" },
    { "name": "drive:sr0", "status": "failed", "error": "stat /dev/sr0: no such file or directory" }
  ]
}
```

- Checks per drive: `drive:<id>` (the device node exists) and `detector:<id>` (fewer than 5 bad
  replies in a row).
- During shutdown `status` is `shutting_down`, the code is 503, and the checks are not run.

This follows the Kubernetes liveness/readiness convention. (The IETF "health+json" draft expired
and is not a standard.)

### 6.2 Metrics (Slot 0: Prometheus pull at `GET <ADMIN_ADDR>/metrics`)

**Provided by the kit:**
- Go runtime metrics;
- `service_build_info`;
- HTTP request/error/duration metrics for the API listener (otelhttp);
- `lifecycle_worker_restarts_total`.

**Ripper's own instruments.** Create them once in `internal/engine/metrics.go` with
`func NewMetrics(m metric.Meter) (*Metrics, error)`, using `p.Meter` from `obs.Setup`. The kit's
exporter rewrites the instrument name into the exported name below. Use the instrument names and
units exactly; do not add labels.

| Instrument (name, type, unit) | Exported name | Labels | Recorded by | When |
|---|---|---|---|---|
| `ripper.rips`, Int64Counter, `{rip}` | `ripper_rips_total` | `drive`, `kind`, `outcome` | engine | after each rip plan (outcome: success/failure/cancelled/skipped) |
| `ripper.rip.duration`, Float64Histogram, `s` | `ripper_rip_duration_seconds` | `drive`, `kind`, `outcome` | engine | same; buckets `60,300,600,1200,1800,3600,5400,7200,10800` (`metric.WithExplicitBucketBoundaries`) |
| `ripper.detect.results`, Int64Counter, `{result}` | `ripper_detect_results_total` | `drive`, `state` (disc.State or `error`) | engine | each pass |
| `ripper.drive.bad_responses`, Int64Gauge, `{response}` | `ripper_drive_bad_responses` | `drive` | engine | each pass |
| `ripper.engine.state`, Int64Gauge, `{state}` | `ripper_engine_state` | `drive`, `state` | engine | on every transition: 1 for the new state, 0 for the old one |
| `ripper.tool.runs`, Int64Counter, `{run}` | `ripper_tool_runs_total` | `tool`, `result` (`ok`, `exit_error`, `cancelled`, `error`) | execrunner | after each Run/Output |
| `ripper.tool.duration`, Float64Histogram, `s` | `ripper_tool_duration_seconds` | `tool` | execrunner | same; default buckets |
| `ripper.notifications`, Int64Counter, `{notification}` | `ripper_notifications_total` | `kind`, `result` (`ok`, `error`) | engine (`notifyDetached`) | after each Notify |
| `ripper.meta.lookups`, Int64Counter, `{lookup}` | `ripper_meta_lookups_total` | `result` (`hit`, `fallback`) | `meta.Fallback` | after each audio-CD lookup |

- Every label value comes from a fixed set: drive IDs from config, the enums in C§2, and tool
  names. Labels, disc labels, paths and errors are **never** metric labels.

### 6.3 Logs (Slot 0: JSON on stdout)

One JSON object per line, written by `obs.NewLogger` (the slog JSON handler with redaction) and
teed into the log ring (C§2.3) for `GET /api/v1/log`.

| Field | Type | Present | Source |
|---|---|---|---|
| `time` | RFC 3339 with nanoseconds | always | slog |
| `level` | `DEBUG` \| `INFO` \| `WARN` \| `ERROR` | always | slog |
| `msg` | string, **static** (from the catalogue below) | always | caller |
| `service` | `"ripper"` | always | kit |
| `version` | build version | always | kit |
| `trace_id`, `span_id` | hex | inside a span | kit |
| `drive` | drive ID | every engine record | engine logger |
| `state` | engine state | state changes | engine |
| `disc_kind`, `disc_label` | string | rip records | engine |
| `outcome` | success/failure/cancelled/skipped | `rip finished` | engine |
| `path` | string (relative to OUTPUT_DIR) | `rip finished` | engine |
| `tool` | string | runner records | execrunner |
| `line` | string | `tool output` | execrunner |
| `exit_code` | int | `tool failed` | execrunner |
| `duration_ms` | int | `rip finished`, `tool finished` | engine, execrunner |
| `error` | string | any failure | caller (`slog.Any("error", err)`) |

Rules:
- `msg` is a constant (sloglint `static-msg`); keys are snake_case (sloglint `key-naming-case`).
- Always log through the context-aware calls (`InfoContext(ctx, …)`) so `trace_id` is attached.
- Never log argv, the MakeMKV key, apprise URLs or the web password. The kit also redacts by key
  name as a backstop.

**Message catalogue** (the only `msg` values; add new ones here first):

| msg | level | key fields |
|---|---|---|
| `ripper starting` | INFO | version, drives (count) |
| `config invalid` | ERROR | error |
| `makemkv key registration failed` | WARN | error |
| `staging cleaned` | INFO | — |
| `disc detected` | INFO | drive, disc_kind, disc_label |
| `detect failed` | WARN | drive, error |
| `drive not responding` | ERROR | drive |
| `rip started` | INFO | drive, disc_kind, disc_label |
| `rip finished` | INFO | drive, disc_kind, disc_label, outcome, path, duration_ms |
| `rip failed` | ERROR | drive, disc_kind, disc_label, error |
| `rip cancelled` | WARN | drive, disc_label |
| `rip step skipped` | WARN | drive, disc_kind, reason |
| `safe to eject` | INFO | drive |
| `eject failed` | ERROR | drive, error |
| `disc removed` | INFO | drive |
| `waiting for disc removal` | DEBUG | drive |
| `exec` | DEBUG | tool |
| `tool output` | INFO | tool, line |
| `tool finished` | DEBUG | tool, duration_ms |
| `tool failed` | WARN | tool, exit_code, error |
| `notification failed` | WARN | kind, error |
| `cleanup failed` | ERROR | drive, error |
| `metadata lookup failed` | WARN | error |
| `cover fetch failed` | WARN | error |

### 6.4 Traces

Use `p.Tracer` from `obs.Setup`. HTTP server spans come from the kit. Ripper adds:

| Span | Parent | Attributes | Status |
|---|---|---|---|
| `engine.rip` | none (root; one per rip plan) | `ripper.drive`, `ripper.disc.kind`, `ripper.outcome` | `Error` on failure |
| `rip.step` | `engine.rip` | `ripper.ripper` (`video`/`audio`/`iso`), `ripper.kind_dir` | `Error` on failure |
| `tool.run` | the current span (from ctx) | `ripper.tool`, `process.exit.code` | `Error` on non-zero exit |

Detection passes are **not** traced (a span per minute per drive would be noise).

### 6.5 OTLP export (when the `OTEL_*` variables ask for it)

This is implemented **in go-service-kit `obs`**, not in ripper. Ripper pins the kit release that has
it (P2.0). Behaviour of the kit change:

| Signal | Slot 0 (always) | OTLP added when | Exporter |
|---|---|---|---|
| Traces | none (spans dropped) | `OTEL_EXPORTER_OTLP_ENDPOINT` or `…_TRACES_ENDPOINT` is set, and `OTEL_TRACES_EXPORTER` ≠ `none` (unchanged kit behaviour) | otlptrace (gRPC/HTTP per `OTEL_EXPORTER_OTLP_PROTOCOL`) |
| Metrics | Prometheus reader → admin `/metrics` | `OTEL_METRICS_EXPORTER` contains `otlp`, **or** it is unset and an OTLP endpoint variable is set | `PeriodicReader(otlpmetric…)`, added *alongside* the Prometheus reader; interval from `OTEL_METRIC_EXPORT_INTERVAL` |
| Logs | JSON handler → stdout (+ ring) | `OTEL_LOGS_EXPORTER` contains `otlp`, **or** it is unset and an OTLP endpoint variable is set | `sdk/log` `LoggerProvider` + `BatchProcessor(otlplog…)`. slog fans out with `slog.NewMultiHandler(jsonHandler, redact(otelslog.NewHandler(...)))`; redaction applies to both branches |

- `none` in any `OTEL_*_EXPORTER` variable disables that signal's OTLP export. Slot 0 never turns off.
- All configuration (endpoint, headers, protocol, timeout, resource attributes, `OTEL_SERVICE_NAME`)
  comes from the standard OTel variables; the kit adds no config of its own.
- `Providers.Shutdown` flushes all three providers within the kit's flush budget.
- Kit modules to add: `go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc` (+http),
  `go.opentelemetry.io/otel/sdk/log`, `go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploggrpc` (+http),
  `go.opentelemetry.io/contrib/bridges/otelslog`.
- `autoexport` is **not** used: its Prometheus mode starts its own HTTP server, which would
  conflict with the kit's admin `/metrics`.
