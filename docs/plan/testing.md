# Testing

Four layers, cheapest first. Layers 1–3 run in CI; layer 4 needs a real drive.

| Layer | What | Where | Command |
|---|---|---|---|
| 1 Unit | pure functions; seams faked in memory; engine under `synctest` | `*_test.go` next to code | `make test` |
| 2 Integration | real processes via **fakebin** on `PATH` | `internal/.../*_test.go` with `TestMain` building fakebin | `make test` |
| 3 Parity | legacy `ripper.sh` vs `ripper serve` against the same fakebin, in a container | `test/parity/` (build tag `parity`) | `make parity` |
| 4 Hardware | real drive | build tag `hardware` | `RIPPER_TEST_DRIVE=/dev/sr0 go test -tags hardware ./...` |

## 1. Rules

- Table-driven tests; `t.Context()` instead of `context.Background()`; `t.TempDir()` for files.
- **`testing/synctest` only for engine unit tests that use in-memory fakes.** Never use it for
  tests that start processes or real listeners (synctest's package doc says to avoid networks
  and external processes). Pattern:
  ```go
  synctest.Test(t, func(t *testing.T) {
  	eng := engine.New(fakes, engine.Config{PollInterval: time.Minute, ManualEjectPoll: 5 * time.Second, BadThreshold: 5})
  	ctx, cancel := context.WithCancel(t.Context())
  	defer cancel()
  	go func() { _ = eng.Run(ctx) }()
  	synctest.Wait()              // first pass done
  	synctest.Sleep(time.Minute)  // next tick
  	// assert on fakes
  })
  ```
- Tests that run `lifecycle.Run` (only `internal/patchbay`) do **not** use synctest. They pass
  `Spec.Listeners` with `net.Listen("tcp", "127.0.0.1:0")` and cancel the context to stop.
- Every fixture test calls `t.Attr("fixture_source", src)`.
- `go test -race -shuffle=on ./...` must pass.

## 2. Fixtures

```
internal/disc/testdata/drv/<case>.txt     raw `makemkvcon -r --cache=1 info disc:9999` output
internal/disc/testdata/drv/<case>.json    expectations + provenance
internal/detect/makemkv/testdata/cdparanoia/<case>.txt   raw `cdparanoia -d /dev/sr0 -Q` output (stderr)
```

`<case>.json` schema:

```json
{
  "source": "synthetic | public | hardware",
  "source_url": "https://… (required when source = public)",
  "device": "/dev/sr0",
  "want": { "state": "inserted", "kind": "bluray", "label": "MOVIE", "index": 0 },
  "want_err": ""
}
```

`want_err` is `""`, `"no_drive_line"` or `"malformed"`.

Required cases: `empty`, `open`, `loading`, `dvd`, `bluray`, `uhd`, `cd_audio`, `cd_data`,
`blank_label_dvd`, `two_drives` (target on index 1), `garbage`, `no_drive_line`, `index_10`.

A synthetic file looks like real output: `MSG:` lines first, then `DRV:0..15`, with absent
drives as `DRV:N,256,999,0,"","",""`. Example target line:
`DRV:0,2,999,12,"BD-RE HL-DT-ST BD-RE  WH16NS40 1.05","MOVIE","/dev/sr0"`.

**Precedence:** where a hardware fixture exists for a case, it replaces the synthetic one. A
public fixture is kept alongside under the name `<case>_public_<n>`.

## 3. fakebin protocol

`internal/testutil/fakebin/main.go` is one Go program. Tests build it once in `TestMain`
(`go build -o <tmp>/fakebin`) and symlink it as each tool name into `<tmp>/bin`, which goes first
on `PATH`. Tool name = `filepath.Base(os.Args[0])`. Everything is driven by `$FAKEBIN_DIR`:

| File in `$FAKEBIN_DIR` | Meaning |
|---|---|
| `calls.jsonl` | fakebin appends `{"tool":"<name>","args":[...]}` + `\n` (`O_APPEND`) |
| `<tool>.count` | per-tool call counter, maintained by fakebin (1-based) |
| `<tool>.<n>.stdout` / `<tool>.stdout` | stdout for call n, else the default. Missing = no output |
| `<tool>.<n>.stderr` / `<tool>.stderr` | same for stderr |
| `<tool>.<n>.exit` / `<tool>.exit` | exit code as a decimal (default `0`) |
| `<tool>.creates` | one path template per line; each is created as a file containing `fake` (parents created). Templates: `{arg:N}` (0-based arg), `{lastarg}`, `{abcde_outputdir}` (the `OUTPUTDIR=` value read from the file passed to `-c`) |
| `<tool>.block` | if present: after recording, block until `<tool>.release` exists (poll 50 ms) or SIGTERM arrives; on SIGTERM exit 143 **without** running `creates` |
| `sleep.max` | (legacy parity only) when the tool is `sleep` and the arg is `1m`: increment the counter; if it is ≥ `sleep.max`, send SIGTERM to the parent process and exit 0. Other sleeps return at once |

Typical scenario setup (BluRay rip):
- `makemkvcon.1.stdout` = the `bluray` fixture;
- `makemkvcon.3.stdout` = the `empty` fixture (call 2 is the `mkv` rip);
- `makemkvcon.creates` = `{lastarg}/title_t00.mkv`.

fakebin applies `creates` only for a `makemkvcon` call whose args contain `mkv`, and always for
`ddrescue` and `abcde`. It never applies it for other tools, so `info` calls create nothing.

## 4. Parity harness (`test/parity`, build tag `parity`)

- `test/parity/Dockerfile`: `ubuntu:noble` + `bash coreutils grep` + Go test binary.
  `test/parity/legacy/ripper.sh` is a **read-only copy** of `root/ripper/ripper.sh` from the
  `archive` branch (checked by a test comparing its sha256 against `test/parity/legacy/SHA256`).
  fakebin is linked into `/usr/local/bin` for each tool **and at `/usr/bin/abcde`** (the legacy
  script calls that absolute path).
- `make parity` builds the image and runs `go test -tags parity ./test/parity/...` inside it as root.
- Each scenario: fresh `/config`, `/out`, `$FAKEBIN_DIR`.
  - **Legacy run:** `sleep.max=2`, env from the scenario, `bash /legacy/ripper.sh`, timeout 60 s.
  - **Go run:** same fakebin files and env plus `POLL_INTERVAL=1s` and `HEADLESS=true`. Start
    `ripper serve`; when `calls.jsonl` stops growing for 3 s, send SIGTERM. Timeout 60 s.
- **Comparison** (after normalising):
  - tool calls, filtered to `makemkvcon` with arg `mkv`, `abcde`, `ddrescue`, `eject`, `sdparm`,
    and `curl` (mapped to `notify`), in order;
  - Go notifications are recorded by a fake apprise target (`json://127.0.0.1:<port>`) and also
    mapped to `notify`;
  - the `/out` tree (relative paths, file vs dir, mode bits, uid/gid).
  - Normalisation: replace `/tmp/…` path segments with `<tmp>`, timestamps `\d{8}_\d{6}` with
    `<ts>`, and abcde `-c <path>` with `-c <conf>`.
- Expected differences are declared per scenario in `test/parity/scenarios.go` as
  `Deviations []string` (D-numbers from the main plan, §8) plus the exact expected Go-side calls/tree.

### 4.1 Scenarios

| Name | Env | Fixture sequence (makemkvcon info) | Expected legacy | Expected Go | Deviations |
|---|---|---|---|---|---|
| `bluray` | default | bluray, empty | mkv rip, eject | same | — |
| `uhd` | default | uhd, empty | mkv rip, eject | same | — |
| `dvd` | default | dvd, empty | mkv rip, eject | same | — |
| `audio_cd` | default | cd_audio (+cdparanoia audio) | abcde `-x`, eject | abcde without `-x`, eject | D22 |
| `data_cd` | default | cd_data (+cdparanoia none) | abcde (wrong) | ddrescue, eject | D2 |
| `empty` | default | empty ×2 | nothing | nothing | — |
| `open` | default | open ×2 | nothing | nothing | — |
| `garbage_once` | default | garbage, empty | eject (bug) | nothing | D3 |
| `bad_threshold` | `BAD_THRESHOLD=2` | garbage ×2 | eject, exit 1 | eject, notify, exit 1 | D3, D10 |
| `just_iso_dvd` | `JUSTMAKEISO=true` | dvd, empty | ddrescue, eject | same | — |
| `also_iso_bluray` | `ALSOMAKEISO=true` | bluray, empty | mkv rip, ddrescue, eject | same | — |
| `separate_finish` | `SEPARATERAWFINISH=true` | dvd, empty | `/out/.../finished/<label>` | same | — |
| `timestamp` | `TIMESTAMPPREFIX=true` | dvd, empty | `<ts>_<label>` | same | — |
| `blank_label` | default | blank_label_dvd, empty | rip into storage root (bug) | `disc_<ts>` | D7 |
| `eject_fallback` | default; `eject.exit=1` | dvd, empty | eject, sdparm unlock, sdparm eject | same | — |
| `eject_disabled` | `EJECTENABLED=false` | dvd, dvd, empty | no eject; waits | same | — |
| `notify` | Pushover vars (legacy) / `APPRISE_URLS` (Go) | dvd, empty | curl | notify (Success) | D10 |
| `cancel_mid_rip` | default; `makemkvcon.block` | bluray | n/a (legacy has no cancel) | rip started; SIGTERM; dir removed; no eject | D18 (Go-only scenario) |

## 5. Hardware tests (`-tags hardware`)

`RIPPER_TEST_DRIVE` must be set or the test is skipped. Tests:
- `TestHardwareDetect`: prints the `Disc` and saves raw output to `testdata/drv/hw_<ts>.txt`
  (the start of a hardware fixture).
- `TestHardwareEject`
- `TestHardwareISO`: reads the first 16 MiB with `ddrescue -s 16Mi`.

Phase 6 backends are merged only after their hardware test passes on your drive.
