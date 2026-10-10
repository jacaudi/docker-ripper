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
  `Spec.Listeners`, and stop by cancelling the context.

## 2. Fixtures

```
internal/detect/makemkv/testdata/drv/<case>.txt         raw `makemkvcon -r --cache=1 info disc:9999` output
internal/detect/makemkv/testdata/cdparanoia/<case>.txt  raw `cdparanoia -d /dev/srN -Q` output
```

- Expectations live in a table in `drv_test.go`: `{file, want []struct{Device, State, Kind, Label, Index}, wantErr}`.
- Required DRV cases:
  - single drive: `empty`, `open`, `loading`, `dvd`, `bluray`, `uhd`, `cd`, `blank_label_dvd`;
  - `two_drives_bd_cd` (sr0 bluray, sr1 cd); `index_10`;
  - `no_drives` (every line has an empty device → empty slice);
  - `garbage` (→ `ErrMalformed`).
- Required cdparanoia cases: `audio`, `no_audio`.
- Synthetic files look like real output: `MSG:` lines first, then `DRV:0..15`, with absent drives
  as `DRV:N,256,999,0,"","",""`. Example target line:
  `DRV:0,2,999,12,"BD-RE HL-DT-ST BD-RE  WH16NS40 1.05","MOVIE","/dev/sr0"`.
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
`/dev/` (e.g. `sr1`). Everything is driven by `$FAKEBIN_DIR`:

| File in `$FAKEBIN_DIR` | Meaning |
|---|---|
| `calls.jsonl` | fakebin appends `{"tool":"<name>","args":[...],"dir":"<cwd>"}` + `\n` (`O_APPEND`) |
| `<tool>.count` | per-tool call counter, maintained by fakebin (1-based) |
| `<tool>.<n>.stdout` / `<tool>.<dev>.stdout` / `<tool>.stdout` | stdout for call n, else for the device, else the default. Missing = no output |
| `<tool>.<n>.stderr` / `<tool>.<dev>.stderr` / `<tool>.stderr` | same lookup order, for stderr |
| `<tool>.<n>.exit` / `<tool>.<dev>.exit` / `<tool>.exit` | exit code as a decimal (default `0`), same lookup order |
| `<tool>.creates` | one path template per line, each created as a file containing `fake` (parents created). Templates: `{arg:N}`, `{lastarg}`, `{flag:X}` (the arg after flag `X`), `{cwd}`. Applied only for a `makemkvcon` call whose args contain `mkv`, and always for `ddrescue` and `cyanrip` |
| `<tool>.block` / `<tool>.<dev>.block` | if present: after recording the call, block until `<tool>[.<dev>].release` exists (poll 50 ms) or SIGTERM arrives. On SIGTERM exit 143 **without** applying `creates` |

Typical setups:
- **BluRay:** `makemkvcon.1.stdout` = the bluray fixture; `makemkvcon.3.stdout` = the empty
  fixture (call 2 is the rip); `makemkvcon.creates` = `{lastarg}/title_t00.mkv`.
- **ISO:** `ddrescue.creates` = `{arg:1}` and `{arg:2}` (the `.iso` and `.map` files).
- **Audio CD:** `cyanrip.creates` = `{cwd}/Artist - Album/01 - Song 1.flac` and
  `{cwd}/Artist - Album/02 - Song 2.mp3`. For the retry: `cyanrip.1.exit` = `1`.

## 4. End-to-end smoke (`internal/patchbay/e2e_test.go`)

- Build the full stack: `telemetry.Setup` → `patchbay.Backends` → `engine.New` → `patchbay.Spec`.
  - `RIPPER_OUTPUT_DIR` = temp dir; `RIPPER_POLL_INTERVAL=1s`; `RIPPER_HEADLESS=true`;
  - apprise target `json://127.0.0.1:<port>` pointing at an `httptest` server;
  - listeners on `127.0.0.1:0`; fakebin on `PATH`;
  - fake device paths are temp files named `sr0`/`sr1`, and the fixtures list those paths.
- Run `lifecycle.Run` in a goroutine.
- Poll `GET /api/v1/jobs` until the expected jobs are finished (timeout 20 s), then cancel and wait
  for `Run` to return.

| Scenario | Setup | Assert |
|---|---|---|
| `bluray` | bluray then empty | `mkv … disc:0 all <staging>`, then eject; `BluRay/MOVIE/title_t00.mkv`; no `.staging` left; job `succeeded`; Success notification; drive back to `idle` after empty |
| `audio_cd` | cd + cdparanoia `audio` | `cyanrip -d <dev> -o flac,mp3` run in the staging dir; `CD/Artist - Album/01 - Song 1.flac`; job `succeeded` |
| `audio_cd_retry` | as above, `cyanrip.1.exit=1` | a second cyanrip call with `-N`; job `succeeded`; WARN `cyanrip retry without musicbrainz` |
| `data_cd` | cd + cdparanoia `no_audio` | ddrescue with `.iso` + `.map`; `DATA/<label>/<label>.iso` |
| `iso_only_dvd` | dvd, `RIPPER_ISO_MODE=only` | ddrescue only; no `mkv` call |
| `cancel_mid_rip` | bluray + `makemkvcon.block` | cancel while blocked; `Run` returns nil; no `BluRay/MOVIE`, no `.staging/<ts>-sr0`; no eject; job `cancelled`; Stopped notification |
| `eject_disabled` | dvd ×3 then empty, `RIPPER_EJECT=false` | exactly one `mkv` call (no re-rip); no eject; drive `awaiting_removal` until empty |
| `bad_drive` | bluray drive whose state field is garbage ×6 | drive `unusable` after the 5th; `/readyz` 503 with `drives` failed (only drive); one Failure notification; process still running |
| `discovery` | scan 1: sr0 only; scan 2: sr0 + sr1; scan 3: sr0 only | `/api/v1/status` lists 1, then 2, then 1 drive; INFO `drive discovered` / `drive removed`; no config needed |
| `two_drives_parallel` | `two_drives_bd_cd`; `makemkvcon.block` released only after the sr1 `cyanrip` call is recorded | both jobs `running` at once; both `succeeded`; two Success notifications |
| `max_parallel_1` | as above, `RIPPER_MAX_PARALLEL_JOBS=1` | the sr1 job stays `queued` until sr0 finishes; `ripper_jobs{state="queued"}` was 1 at some point |
| `health` | default, then remove `cyanrip` from `PATH` | `/livez` 200; `/startupz` 503 before the first scan, then 200; `/readyz` 200; `/healthz` 200, then 503 with `tool:cyanrip` failed |

**Observability assertions in every scenario:**
- `GET <admin>/metrics` contains `ripper_jobs_completed_total` with the scenario's `kind` and `state`;
- every record in the log ring parses as JSON with `time`, `level`, `msg`, `service`, and its `msg`
  is in the C§6.3 catalogue;
- no record contains the test key `T-test…` or an apprise URL;
- with an in-memory span exporter (via the telemetry test option), one `engine.job` span exists per
  job, with `tool.run` children.
