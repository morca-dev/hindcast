# Hindcast

[![Mojo version](https://mojourney.dev/badge/morca-dev/hindcast/mojo.svg)](https://mojourney.dev/repos/morca-dev/hindcast)

How good were NOAA's weather forecasts? Hindcast fetches each day's GFS
forecast of 2 m temperature, 0 to 7 days ahead, and scores it against what
the next days' analyses say happened.

It's a [morca](https://github.com/morca-dev/morca) project with one
pipeline and a step in Rust: the fetching and GRIB2 decoding are a Rust
worker built from crates.io crates (`grib`, `ureq` with rustls), loaded over
the C ABI by `morca.rust_step` ([morca's docs/rust-workers.md](https://github.com/morca-dev/morca/blob/main/docs/rust-workers.md));
the scoring is ordinary Mojo steps over Iceberg tables. It began as morca's
examples/hindcast, and runs on a Hetzner stack of its own, apart from the
census, so the census never compiles Rust.

```text
pipelines/hindcast.mojo, partitions: forecast cycles (00z), Daily from 2021-03-23, 06:00 UTC for the day before

forecasts [mapped, Rust]                      skill [gather, Mojo]
  rust_step ──▶ libhindcast_worker              for each valid day V the run touches:
    per lead f000 … f168:                         analysis = f000 of cycle V
      GET gfs…f024.idx                            for each lead L: forecast of cycle V - L
      GET gfs…f024, Range: one message            bias, MAE, RMSE (cos-latitude weighted)
      decode (grib, template 5.3)                 and the city scorecard
      ──▶ one Arrow batch, 65,160 rows      ──▶ hindcast.skill, hindcast.cities
  ──▶ hindcast.t2m                                (replace V's partition)
      (replaces the cycle's partitions)
```

| Path | What it is |
|---|---|
| `pipelines/hindcast.mojo` | the pipeline, its steps and `hindcast report` |
| `src/skill.mojo` | the scoring: bias, MAE, RMSE and the city scorecard |
| `worker/` | the Rust worker (`Cargo.toml` and `Cargo.lock` at the root) |
| `schemas/t2m.json` | `hindcast.t2m`'s schema, which the worker and the pipeline both embed |
| `fixtures/` | two cycles' f000 and f024 messages, cut from NOAA's files, for the tests |
| `morca.toml` | how the stack runs it: control placement, its request |
| `Dockerfile.morca` | the image, on `ghcr.io/morca-dev/morca-runtime` |

## What it does

- **Data.** GFS at 1° (`pgrb2.1p00`), the field `TMP:2 m above ground`, the
  00z run of each day, leads f000, f024 … f168. A file is 45 MB; the worker
  reads the file's `.idx`, finds the message (it ends where the next line's
  starts), and fetches only that with an HTTP Range request: about 74 KB a
  lead, from NOAA's open bucket
  `https://noaa-gfs-bdp-pds.s3.amazonaws.com/gfs.YYYYMMDD/00/atmos/gfs.t00z.pgrb2.1p00.fNNN`.
  No credentials.
- **`forecasts`**, the Rust step (worker/lib.rs, worker/gfs.rs). Each lead
  becomes one batch of (cycle, lead_hours, valid_time, lat, lon, t2m_k),
  made when morca pulls it, so the worker holds one lead at a time. The
  worker embeds schemas/t2m.json with `include_str!`, the pipeline with
  scripts/embed-json.sh; `cargo test` checks the worker against it, and
  morca checks the stream against the table before writing. The library
  links only libc: `grib` without its C codecs (2 m temperature is complex
  packing, template 5.3, which `grib` decodes itself) and `ureq` over
  rustls with webpki's roots, so no OpenSSL and no CA files, which suits
  the distroless runtime image.
- **`skill`**, the Mojo step (src/skill.mojo). A forecast of cycle C at lead
  L is valid at V = C + L, and is scored against V's analysis, the f000 of
  cycle V: grid-wide bias, MAE and RMSE of forecast minus analysis, each
  point weighted by cos(latitude), and the error at the grid point nearest
  each of ten cities.
- **Tables**: `hindcast.t2m` (partitioned by cycle and lead),
  `hindcast.skill` (a row per cycle and lead) and `hindcast.cities` (a row
  per cycle, lead and city), both partitioned by valid date.

### Scores that need later days

A forecast can only be scored once its valid day's analysis is fetched: a
cycle's 7-day forecast waits a week. A run's scoring step works out the
valid days its cycles touch (each cycle's own day, and the seven after),
recomputes everything valid on each from what `hindcast.t2m` has, and
replaces that day's partition in the score tables. A forecast whose
analysis isn't there yet is skipped, and scored by the run that fetches it.

So the scores don't depend on the order cycles arrive in: daily runs, a
backfill's runs in any order, or a rerun all end with every (cycle, lead)
whose two cycles are fetched. A cycle is **complete** once all seven leads
are scored; `hindcast report` lists those that aren't yet.

A cycle that isn't in the bucket fails the worker with "not published yet".
For a cycle of the last two days the step then defers half an hour
(`morca.defer`), spending no retry; for an older one it fails.

## Deploying: its own stack

Hindcast runs on a Hetzner stack of its own, with morca's
[deploy/hetzner](https://github.com/morca-dev/morca/tree/main/deploy/hetzner)
scripts ([docs/hetzner.md](https://github.com/morca-dev/morca/blob/main/docs/hetzner.md)):
one small ARM server (a CAX11 should do: a run is one CPU and under a GB), its own bucket, Postgres and
tunnel for the UI. Its settings file is a copy of `hetzner.env.example`
with, besides the usual server, bucket and tunnel settings:

```sh
NAME=hindcast
SERVER_TYPE_SMALL=cax11
BUCKET=hindcast-warehouse
MORCA_IMAGE=ghcr.io/morca-dev/hindcast
MORCA_TAG=<short sha>          # from main's CI, the push job's summary
APPS=                           # no apps, the UI only: see below
GHCR_USER=...                   # while the package is private
GHCR_TOKEN_FILE=...
```

Then, from morca's `deploy/hetzner`:

```sh
./up.sh                                                    # the server and the stack
ssh root@<ip> docker exec scheduler morca register         # the image's morca.toml
./deploy.sh <short sha>                                    # each new image afterwards
```

Hindcast has no apps, so the tunnel needs only the UI's hostname
(`http://ui:8270`). morca 0.27.0's `lib.sh` reads an empty `APPS` as
`console:8080`, which this image hasn't, so a stack with no apps needs
morca's scripts to accept one first.

**The image.** CI builds `Dockerfile.morca` on every pull request, for
arm64 (Neoverse N1, Hetzner's CAX) and amd64 (x86-64-v3), and tests it:
`morca config check`, the libraries resolve, `describe` reports the commit,
and two cycles of the fixtures run inside it. A push to main pushes
`ghcr.io/morca-dev/hindcast:<short sha>`; a tag `v<version>` pushes
`:<version>` too and makes the GitHub release from CHANGELOG.md. In the
image:

- `/opt/morca/pipelines/hindcast`, the pipeline, with its source map;
- `/opt/morca/lib/libhindcast_worker.so`, the worker, found by its bare
  name in `$CONDA_PREFIX/lib` (`/opt/morca`);
- `/opt/morca/morca.toml`, which `MORCA_CONFIG` names, and
  `MORCA_PIPELINES_DIR=/opt/morca/pipelines`, so `morca register` alone
  registers it;
- `MORCA_COMMIT` and `MORCA_SOURCE` in the environment, from build args.

morca's own `morca`, `morca_maintain`, the UI and the rest are the runtime
image's: `ghcr.io/morca-dev/morca-runtime:0.33.0`, the morca-mojo pixi.toml
pins. Moving to a new morca moves three pins together: pixi.toml's
`morca-mojo` revision, Cargo.toml's `morca-worker` tag and
Dockerfile.morca's `MORCA_VERSION`.

## The backfill

The schedule runs yesterday's cycle each day at 06:00 UTC, from the day
the pipeline is registered (no `fills`: since 2021 is about 2,000 cycles).
Earlier days are backfilled by hand, as many as wanted:

```sh
ssh root@<ip> docker exec scheduler morca backfill hindcast 2026-10-01..2026-10-30
ssh root@<ip> docker exec scheduler morca status hindcast
ssh root@<ip> docker exec scheduler hindcast report
```

The backfill queues one run per day. Each run fetches its cycle (eight
Range requests, about 600 KB) and rescores the valid days it touches, so the
scores fill in as runs finish, in whatever order. Cycles to 2026-10-23 are
complete when this backfill is done; the last seven wait for analyses from
November, which the daily schedule or a backfill of those days brings.
`morca backfill --rerun hindcast <days>` fetches and scores days again,
replacing what was there.

## Running it locally

```sh
pixi run -e rust build-worker          # target/release/libhindcast_worker.so (.dylib on macOS)
pixi run build                         # build/hindcast

export MORCA_WAREHOUSE=$PWD/build/wh
export HINDCAST_LIB=$PWD/target/release/libhindcast_worker.so
./build/hindcast run --partitions 2026-11-01..2026-11-12
./build/hindcast report 2026-11-01
```

- `HINDCAST_LIB`: the worker library; by default `hindcast_worker`, found as
  `libhindcast_worker.so` in `$CONDA_PREFIX/lib`, next to the binary or in
  its `../lib`.
- `HINDCAST_SOURCE`: where the files come from, a base URL or a directory
  laid out as the bucket; by default NOAA's bucket. The tests point it at
  fixtures/.
- `HINDCAST_LEADS`: the leads in hours, comma separated, whole days; by
  default `0,24,48,72,96,120,144,168`.

`hindcast report [cycle]` prints the skill per lead over the cycles scored,
the cycles still waiting for analyses, the tables' row counts, and with a
cycle, that cycle's rows and city scorecard.

## Tests

`pixi run -e rust test` (tests/hindcast.sh, CI's `test` job on arm64 and
amd64) runs offline, on fixtures/: the 2 m temperature messages of the
2026-11-12 and 2026-11-13 cycles at f000 and f024, with `.idx` files whose
offsets are those in the cut file. It checks the `.idx` lookup and the
decode against ecCodes (`cargo test`), the library's exports and that it
links no Rust std or OpenSSL, that the two cycles in either order give the
same scores, the skill and the city scorecard against numbers ecCodes and
numpy got (tools/skill_ref.py, run once by hand), a waiting cycle reported
incomplete, a rerun replacing, and a missing cycle deferring or failing by
its age.

`cargo test -- --ignored` fetches f024 of 2026-11-12 from the live bucket
and checks it's the fixture.
