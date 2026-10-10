# Testing

The Go code is tested against a **behaviour spec** ([`contracts.md`](contracts.md)), not against
the legacy script. Real external tools are replaced by **fakebin**, so every check runs in CI
without a drive. Real hardware is checked once, at cutover (P4.5).

| Layer | What | Where | Command |
|---|---|---|---|
| Unit | pure functions; seams faked in memory; engine under `synctest` | `*_test.go` next to the code | `task test` |
| Integration | each backend runs real processes via fakebin on `PATH` | backend packages, with `TestMain` building fakebin | `task test` |
| End-to-end smoke | the whole `serve` stack (patchbay `Spec`) against fakebin, over HTTP | `internal/patchbay/e2e_test.go` | `task test` |

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
  	synctest.Wait()             // the first pass is done
  	synctest.Sleep(time.Minute) // next tick
  	// assert on the fakes
  })
  ```
- In-memory fakes live in `internal/engine/enginetest/fakes.go`:
  - a scripted `Detector` (a slice of `{Disc, error}`; the last result repeats);
  - recording `Ripper`, `Ejector` and `Notifier`;
  - a `BlockingRipper` that returns only when its ctx is done.
- Tests that run `lifecycle.Run` (patchbay only) use real listeners on `127.0.0.1:0` passed via
  `Spec.Listeners`, and stop by cancelling the context.

## 2. Fixtures

```
internal/detect/makemkv/testdata/drv/<case>.txt         raw `makemkvcon -r --cache=1 info disc:9999` output
internal/detect/makemkv/testdata/cdparanoia/<case>.txt  raw `cdparanoia -d /dev/sr0 -Q` output
```

- Expectations live in a table in `drv_test.go`: `{file, device, wantState, wantKind, wantLabel, wantIndex, wantErr}`.
- Required DRV cases: `empty`, `open`, `loading`, `dvd`, `bluray`, `uhd`, `cd`, `blank_label_dvd`,
  `two_drives` (target on index 1), `index_10`, `garbage`, `no_drive_line`.
- Required cdparanoia cases: `audio` (a 2-track TOC table), `no_audio`.
- `internal/cdda/testdata/`: published TOC vectors with their MusicBrainz and CDDB IDs (C§3.6a; cite the source).
- `internal/meta/musicbrainz/testdata/`: a recorded `discid` JSON response (single artist) and one
  for Various Artists.
- Synthetic files look like real output: `MSG:` lines first, then `DRV:0..15`, with absent drives
  as `DRV:N,256,999,0,"","",""`. Example target line:
  `DRV:0,2,999,12,"BD-RE HL-DT-ST BD-RE  WH16NS40 1.05","MOVIE","/dev/sr0"`.
- **Hardware captures replace synthetic files 1:1.** The owner runs `ripper detect --raw` (C§5.6)
  per disc type and pastes the output into the matching `.txt`. If the expectation table then
  fails, the hardware is right: fix the table and the contract together.

## 3. fakebin

`internal/testutil/fakebin/main.go` is one Go program. `internal/testutil/fakebintest` provides:
- `Install(t) (binDir string)`: build once per test binary, symlink each tool name, prepend to `PATH`
  via `t.Setenv`;
- `Scenario(t) (dir string)`: sets `FAKEBIN_DIR`;
- `Calls(t, dir) []Call`.

Tool name = `filepath.Base(os.Args[0])`. Everything is driven by `$FAKEBIN_DIR`:

| File in `$FAKEBIN_DIR` | Meaning |
|---|---|
| `calls.jsonl` | fakebin appends `{"tool":"<name>","args":[...]}` + `\n` (`O_APPEND`) |
| `<tool>.count` | per-tool call counter, maintained by fakebin (1-based) |
| `<tool>.<n>.stdout` / `<tool>.<dev>.stdout` / `<tool>.stdout` | stdout for call n, else for the device (`<dev>` = basename of the first arg starting with `/dev/`, e.g. `cdparanoia.sr1.stdout`), else the default. Missing = no output |
| `<tool>.<n>.stderr` / `<tool>.<dev>.stderr` / `<tool>.stderr` | same lookup order, for stderr |
| `<tool>.<n>.exit` / `<tool>.<dev>.exit` / `<tool>.exit` | exit code as a decimal (default `0`), same lookup order |
| `<tool>.creates` | one path template per line, each created as a file containing `fake` (parents created). Templates: `{arg:N}` (0-based arg), `{lastarg}`, `{flag:X}` (the arg after flag `X`, e.g. `{flag:-o}`), `{cwd}` (the working directory). Applied only for a `makemkvcon` call whose args contain `mkv`, a `cdparanoia` call whose args contain `-B`, and always for `ddrescue`, `flac` and `lame` |
| `<tool>.block` | if present: after recording the call, block until `<tool>.release` exists (poll 50 ms) or SIGTERM arrives. On SIGTERM exit 143 **without** applying `creates` |

Typical setups:
- **BluRay:** `makemkvcon.1.stdout` = the bluray fixture; `makemkvcon.3.stdout` = the empty
  fixture (call 2 is the rip); `makemkvcon.creates` = `{lastarg}/title_t00.mkv`.
- **ISO:** `ddrescue.creates` = `{arg:1}` and `{arg:2}` (the `.iso` and `.map` files).
- **Audio CD:** `cdparanoia.creates` = `{cwd}/track01.cdda.wav` and `{cwd}/track02.cdda.wav`;
  `flac.creates` = `{flag:-o}`; `lame.creates` = `{lastarg}`. The `audio` cdparanoia fixture lists 2 tracks.

## 4. End-to-end smoke (`internal/patchbay/e2e_test.go`)

- Build the full stack with `patchbay.Backends` + `patchbay.Spec`:
  - `RIPPER_OUTPUT_DIR` = temp dir; `RIPPER_POLL_INTERVAL=1s`; `RIPPER_HEADLESS=true`;
  - apprise target `json://127.0.0.1:<port>` pointing at an `httptest` server;
  - `patchbay.Options{Meta: fakeLookup}` returning `Artist` / `Album` / tracks `Song 1`, `Song 2`
    (no network in tests);
  - listeners on `127.0.0.1:0`; fakebin on `PATH`.
- Run `lifecycle.Run` in a goroutine.
- Poll `GET /api/v1/status` until every drive has a `last_result` (timeout 20 s), then cancel and wait for `Run` to return.
- **Observability assertions in every scenario:**
  - `GET <admin>/metrics` contains `ripper_rips_total` with the scenario's `kind` and `outcome`;
  - every record in the log ring parses as JSON with `time`, `level`, `msg`, `service`, and its `msg`
    is in the C§6.3 catalogue;
  - no record contains the test key `T-test…` or an apprise URL;
  - with an in-memory span exporter (`obs.Config.SpanExporter`), one `engine.rip` span exists per
    rip, with `tool.run` children.

| Scenario | Setup | Assert |
|---|---|---|
| `bluray` | bluray then empty | calls: info, `mkv … disc:0 all <staging>`, eject; `BluRay/MOVIE/title_t00.mkv` exists; no `.staging` left; Success notification; `awaiting_removal` cleared after empty |
| `audio_cd` | empty + cdparanoia `audio` | cdparanoia `-B`, then flac ×2 and lame ×2 with tags; `CD/Artist - Album/01. Song 1.flac`, `…/02. Song 2.mp3`; no `.wav` left; Success |
| `data_cd` | cd + cdparanoia `no_audio` | ddrescue with `.iso` + `.map`; `DATA/<label>/<label>.iso` |
| `iso_only_dvd` | dvd, `RIPPER_ISO_MODE=only` | ddrescue only; no `mkv` call |
| `cancel_mid_rip` | bluray + `makemkvcon.block` | cancel while blocked; `Run` returns nil; no `BluRay/MOVIE`, no `.staging/<ts>`; no eject call; Stopped notification |
| `eject_disabled` | dvd ×3 then empty, `RIPPER_EJECT=false` | exactly one `mkv` call (no re-rip); no eject call; status `awaiting_removal` until empty |
| `bad_drive` | garbage ×6 | `/readyz` returns 503 with `detector:sr0` failed after the 5th; one Failure notification; process still running |
| `two_drives` | `RIPPER_DRIVES=/dev/sr0,/dev/sr1`; one makemkvcon fixture listing sr0 = bluray and sr1 = cd; `cdparanoia.sr1.stdout` = audio; `makemkvcon.block` released after the sr1 `cdparanoia -B` call is recorded | **both rips run concurrently** (cdparanoia `-B` starts before the BluRay rip finishes); `BluRay/MOVIE/…` and `CD/Artist - Album/…` exist; two Success notifications; status lists both drives |
