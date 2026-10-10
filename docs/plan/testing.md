# Testing

The Go code is tested against a **behaviour spec** ([`contracts.md`](contracts.md)), not against
the legacy script. Real external tools are replaced by **fakebin**, so every check runs in CI
without a drive. Real hardware is checked once, at cutover (P4.5).

| Layer | What | Where | Command |
|---|---|---|---|
| Unit | pure functions; seams faked in memory; engine under `synctest` | `*_test.go` next to the code | `task test` |
| Integration | each backend runs real processes via fakebin on `PATH` | backend packages, with `TestMain` building fakebin | `task test` |
| End-to-end smoke | the whole `serve` stack (patchbay) against fakebin, over HTTP | `internal/patchbay/e2e_test.go` | `task test` |

## 1. Rules

- Table-driven tests; `t.Context()`; `t.TempDir()`. `go test -race -shuffle=on` must pass.
- **`testing/synctest` is only for engine unit tests that use in-memory fakes.** Never use it
  around processes or real listeners. Pattern:
  ```go
  synctest.Test(t, func(t *testing.T) {
  	eng := engine.New(fakes, engine.Config{ISOMode: engine.ISOOff, Eject: true, PollInterval: time.Minute})
  	ctx, cancel := context.WithCancel(t.Context())
  	defer cancel()
  	go func() { _ = eng.Run(ctx) }()
  	synctest.Wait()             // the first scan is done
  	synctest.Sleep(time.Minute) // next tick
  	// assert on the fakes
  })
  ```
- In-memory fakes live in `internal/engine/enginetest/fakes.go`:
  - a scripted `Detector` (a slice of `{[]disc.Disc, error}`; the last result repeats);
  - recording `Ripper`, `Ejector` and `Notifier`;
  - a `BlockingRipper` that returns only when its ctx is done, or when the test releases it per drive.
- Tests that run `lifecycle.Run` (patchbay only) use real listeners on `127.0.0.1:0` passed via
  `Spec.Listeners` (paired by index with `Spec.Servers`), and stop by cancelling the context.
- Tests that don't assert on metrics or spans pass `noop.NewMeterProvider().Meter("test")` and
  `noop.NewTracerProvider().Tracer("test")` (`go.opentelemetry.io/otel/metric/noop`, `…/trace/noop`).
  Tests that do assert use `sdkmetric.NewManualReader()` and `tracetest.NewInMemoryExporter()` (C§6.5 test-import rule).
- Inside `synctest.Test`, `Deps.Now` is `time.Now` (the bubble's fake clock).

## 2. Fixtures

```
internal/detect/makemkv/testdata/drv/<case>.txt         raw `makemkvcon -r --cache=1 info disc:9999` output
internal/detect/makemkv/testdata/cdparanoia/<case>.txt  raw `cdparanoia -d /dev/srN -Q` output
```

- Expectations live in a table in `drv_test.go`: `{file, want []struct{Device, State, Kind, Label, Index}, wantErr}`.
- Required DRV cases:
  - single drive: `empty`, `open`, `loading`, `dvd`, `bluray`, `uhd`, `cd`, `blank_label_dvd`;
  - `two_drives_bd_cd` (sr0 bluray, sr1 cd); `two_drives_bd_empty` (sr0 bluray, sr1 empty);
    `two_drives_empty` (both empty); `index_10`;
  - `no_drives` (every line has an empty device → empty slice);
  - `garbage`: contains `MSG:` lines and one line `DRV:0,2,999` (fewer than 7 fields) → `ErrMalformed`;
  - `bad_state`: the bluray fixture with the state field replaced by `x` → State Unknown.
- Required cdparanoia cases: `audio`, `no_audio`.
- Synthetic files look like real output. Full example, `bluray.txt`:
  ```
  MSG:1005,0,1,"MakeMKV v1.18.0 linux(x64-release) started","%1 started","MakeMKV v1.18.0 linux(x64-release)"
  MSG:2010,0,1,"Optical drive \"BD-RE HL-DT-ST BD-RE  WH16NS40 1.05\" opened in OS access mode.","",""
  DRV:0,2,999,12,"BD-RE HL-DT-ST BD-RE  WH16NS40 1.05","MOVIE","/dev/sr0"
  DRV:1,256,999,0,"","",""
  …                                   (one line per index up to DRV:15, all like DRV:1)
  ```
- **Hardware captures replace synthetic files 1:1.** The owner runs `ripper detect --raw` (C§5.6)
  per disc type and pastes the output into the matching `.txt`. If the expectation table then fails,
  the hardware is right: fix the table and the contract together.

## 3. fakebin

`internal/testutil/fakebin/main.go` is one Go program. `internal/testutil/fakebintest` provides:
- `Install(t) (binDir string)`: build once per test binary, symlink each tool name, prepend to
  `PATH` via `t.Setenv`;
- `Scenario(t) (dir string)`: sets `FAKEBIN_DIR`;
- `Calls(t, dir) []Call`.

Tool name = `filepath.Base(os.Args[0])`. `<dev>` = the basename of the first arg starting with
`/dev/` (e.g. `sr1`). `<sub>` = the first arg matching `^[a-z]+$` (makemkvcon: `info`, `mkv`, `reg`;
none for the other tools). Everything is driven by `$FAKEBIN_DIR`:

| File in `$FAKEBIN_DIR` | Meaning |
|---|---|
| `calls.jsonl` | fakebin appends `{"tool":"<name>","args":[...],"dir":"<cwd>","pid":<os.Getpid()>}` + `\n` (`O_APPEND`). `fakebintest.Call` is `struct{ Tool string; Args []string; Dir string; PID int }` |
| `<tool>.count`, `<tool>.<sub>.count` | call counters, maintained by fakebin (1-based): `<n>` counts every call of the tool, `<k>` counts only calls with the same `<sub>` |
| stdout files | first match wins: `<tool>.<sub>.<k>.stdout`, `<tool>.<n>.stdout`, `<tool>.<dev>.stdout`, `<tool>.<sub>.stdout`, `<tool>.stdout`. Missing = no output |
| stderr files | same lookup order, `.stderr` |
| exit files | same lookup order, `.exit`: exit code as a decimal (default `0`) |
| `<tool>.creates` | one path template per line, each created as a file containing `fake` (parents created). Templates: `{arg:N}`, `{lastarg}`, `{flag:X}` (the arg after flag `X`), `{cwd}`. Applied only for a `makemkvcon` call whose args contain `mkv`, and always for `ddrescue` and `cyanrip` |
| `<tool>.block` / `<tool>.<dev>.block` | applies **only** to calls that `creates` applies to (a `makemkvcon` `mkv` call; any `ddrescue` or `cyanrip` call); scans, `reg` and `cdparanoia` never block. If present: after recording the call, block until `<tool>[.<dev>].release` exists (poll 50 ms) or SIGTERM arrives (`signal.Notify(c, syscall.SIGTERM)`). On SIGTERM exit 143 **without** applying `creates` |

Typical setups:
- **BluRay:** `makemkvcon.info.1.stdout` = the bluray fixture; `makemkvcon.info.stdout` = the empty
  fixture (every later scan); `makemkvcon.creates` = `{lastarg}/title_t00.mkv`. Using the `info`
  counter means the startup `reg` call and scans that run during a rip don't shift the numbering.
- **ISO:** `ddrescue.creates` = `{arg:1}` and `{arg:2}` (the `.iso` and `.map` files).
- **Audio CD:** `cyanrip.creates` = `{cwd}/Artist - Album/FLAC/01 - Song 1.flac` and
  `{cwd}/Artist - Album/MP3/01 - Song 1.mp3`. For the retry: `cyanrip.1.exit` = `1`.

## 4. End-to-end smoke (`internal/patchbay/e2e_test.go`)

- Build the full stack: `telemetry.Setup` → `patchbay.Backends` → `engine.New` → `patchbay.Spec`.
  - `RIPPER_OUTPUT_DIR` = temp dir; `RIPPER_POLL_INTERVAL=1s`; `RIPPER_HEADLESS=true`;
  - apprise target `json://127.0.0.1:<port>` pointing at an `httptest` server;
  - listeners on `127.0.0.1:0`; fakebin on `PATH`;
  - fake device paths are temp files named `sr0`/`sr1`. The test writes `makemkvcon.N.stdout` by
    replacing `/dev/sr0` and `/dev/sr1` in the fixture text with those temp paths.
- Run `lifecycle.Run` in a goroutine.
- Poll `GET /api/v1/jobs` until the expected jobs are finished (timeout 20 s), then cancel and wait
  for `Run` to return.

| Scenario | Setup | Assert |
|---|---|---|
| `bluray` | `makemkvcon.info.1` = bluray, default `info` = empty | `mkv … disc:0 all <staging>`, then eject; `BluRay/MOVIE/title_t00.mkv`; no `.staging` left; job `succeeded`; Success notification; drive back to `idle` after empty |
| `audio_cd` | `info.1` = cd, default `info` = empty; cdparanoia `audio`; `RIPPER_AUDIO_DRIVE_OFFSETS=sr0=6` | cyanrip argv equals the C§3.6 Go argv with offset `6`, run in the staging dir; `CD/Artist - Album/FLAC/01 - Song 1.flac` and `…/MP3/01 - Song 1.mp3`; job `succeeded` |
| `audio_cd_retry` | as above, `cyanrip.1.exit=1`, no offsets | both calls have `-s 0`; the second replaces `-R 1` with `-N`; job `succeeded`; WARN `cyanrip retry without musicbrainz` |
| `removed_while_queued` | `RIPPER_MAX_PARALLEL_JOBS=1`; `info.1` = `two_drives_bd_cd`, default `info` = `two_drives_bd_empty`; cdparanoia `audio`; `makemkvcon.block`, released after sr1's job is `cancelled` | sr1's job `cancelled`, never started; sr1 `idle` |
| `data_cd` | `info.1` = cd, default `info` = empty; cdparanoia `no_audio` | ddrescue with `.iso` + `.map`; `DATA/<label>/<label>.iso` |
| `iso_only_dvd` | `info.1` = dvd, default `info` = empty; `RIPPER_ISO_MODE=only` | ddrescue only; no `mkv` call |
| `cancel_mid_rip` | `info.1` = bluray, default `info` = bluray; `makemkvcon.block` | cancel while blocked; `Run` returns nil; no `BluRay/MOVIE`, no `.staging/<ts>-sr0`; no eject; job `cancelled`; Stopped notification |
| `eject_disabled` | `makemkvcon.info.1`–`info.4` = dvd, default `info` = empty; `RIPPER_EJECT=false` | exactly one `mkv` call (no re-rip); no eject; drive `awaiting_removal` until empty |
| `bad_drive` | default `info` = the `bad_state` fixture | drive `unusable` after the 5th; `/readyz` 503 with `drives` failed (only drive); one Failure notification; process still running |
| `discovery` | `info.1` = `empty` (sr0 only), `info.2` = `two_drives_empty`, default `info` = `empty` | `/api/v1/status` lists 1, then 2, then 1 drive; INFO `drive discovered` / `drive removed`; no config needed |
| `two_drives_parallel` | `info.1` = `two_drives_bd_cd`, default `info` = `two_drives_empty`; cdparanoia `audio`; `makemkvcon.block` released only after the sr1 `cyanrip` call is recorded | both jobs `running` at once; both `succeeded`; two Success notifications |
| `max_parallel_1` | as above, `RIPPER_MAX_PARALLEL_JOBS=1` | the sr1 job stays `queued` until sr0 finishes; `ripper_jobs{state="queued"}` was 1 at some point |
| `health` | default, then delete the `cyanrip` symlink in the fakebin dir | `/livez` 200; `/startupz` 503 before the first scan, then 200; `/readyz` 200; `/healthz` 200, then 503 with `tool:cyanrip` failed |

**Observability assertions in every scenario:**
- `GET <admin>/metrics` contains `ripper_jobs_completed_total` with the scenario's `kind` and `state`;
- every record in the log ring parses as JSON with `time`, `level`, `msg` and `service`;
- every record that carries a `drive`, `job_id`, `tool` or `check` attribute has a `msg` in the C§6.3
  catalogue. Records whose `msg` starts with `lifecycle: `, `obs: ` or `otel` come from the kit and are ignored;
- no record contains the test key or an apprise URL. The test key is
  `"T-" + strings.Repeat("test", 16) + "xx"` (66 characters after `T-`, so it passes validation);
- with an in-memory span exporter (via the telemetry test option), one `engine.job` span exists per
  job, with `tool.run` children.
