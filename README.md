# VIctory on an RTX 4090 Laptop GPU

A fork of [avatarmanz/VIctory-7900XTX](https://github.com/avatarmanz/VIctory-7900XTX)
for a laptop with an NVIDIA GeForce RTX 4090 Laptop GPU (16 GB), an Intel
integrated GPU beside it, and 32 GB of RAM. What it adds (2026-11-26):

- **The device is made on the high-performance GPU.** Upstream made it on the
  system's first adapter, which on a laptop with two is often the Intel one
  that drives the screen (no mesh shaders, a fraction of the speed).
  `VICTORY_GPU=default`: the first adapter, as before; `VICTORY_GPU=<n>`: the
  n-th in order of performance.
- **Wide indirect dispatches on NVIDIA, opt-in** (`scripts\play.ps1
  -WideIndirect`, `VICTORY_INDIRECT_WIDE=force`): dispatches past 65,535 groups
  run whole, as on AMD, instead of losing the lowest tiles of the picture.
  The test `indirect::tests::card_runs_wide_indirect_dispatches` passes on the
  RTX 4090 Laptop GPU (up to 2^20 groups, every group once) with
  `VICTORY_INDIRECT_WIDE=force`; not yet tried in a whole session, so it stays
  off unless asked for.
- **Idle command lists let go when the PC is short of memory**: below
  `VICTORY_LOW_MEMORY_MB` free (3072 unless set; 0: never) idle host lists and
  their allocators are released instead of kept, before Windows starts paging.
  With memory to spare nothing changes. `VICTORY_LIST_IDLE` sets how many are
  kept idle (512).
- **`scripts\play.ps1 -Exact`** (`VICTORY_FETCH_CHECK=1 VICTORY_CARD_ALL=1`):
  what the card writes and what the title copies reach the right submission.
  Opt-in until its cost is measured in a session.

### The test suite on the RTX 4090 Laptop GPU (2026-11-26)

`cargo test --workspace --locked --profile diag -- --test-threads=1`: 915
passed, 0 failed (the system's default adapter was the RTX 4090 here). Tests
for the RDNA-only shader forms skip themselves on NVIDIA, as they should.

The ignored relay tests (`host::upcheck`, `host::card_steps`), run one by one
with `VICTORY_RELAY=2 VICTORY_RELAY_COPY=1` as `play.ps1` sets them, 5 runs each:

| Test | As upstream's play.ps1 | With `-Exact` |
|---|---|---|
| `a_result_caught_in_between_is_put_right` | fails (12 of 12 results wrong in title memory) | passes (`FETCH_CHECK`) |
| `each_submission_reads_its_own_copy_in_pc_memory` | fails (2 of 3 submissions read the third's bytes) | passes (`CARD_ALL`) |
| the other 7 | pass | pass |

`writes_through_picked_views_come_back` fails too, but it is about
`VICTORY_PICKED_WRITES`, which `play.ps1` does not set.
`resource::tests::buffer_data_reaches_the_host_texture` fails when the whole
of `play.ps1`'s settings are set together (no single one does it), on an
assertion about who holds the host texture, not on its bytes: the test was
not written for those settings.

Nothing else is changed: what follows is upstream's description.

---

# VIctory on an RX 7900 XTX

A fork of [VIctory](https://github.com/Marvisak/VIctory), a compatibility layer
that runs an Xbox Series title natively on Windows from files the user supplies.
This one is about **speed on AMD RDNA 3**: the world at 30 frames a second on
one PC, without giving up the picture. It is not there yet.

**This repository contains no game files, code or assets.** You need your own
copy of the game. It is not affiliated with or endorsed by the game's developer
or publisher.

## Where it stands (2026-11-25)

Tested on one machine only: Windows 11, Intel i9-12900K, 64 GB RAM, Radeon
RX 7900 XTX, 1440p, the Series X profile.

| | |
|---|---|
| Menus, videos, loading screens | 30 fps |
| Prologue, indoors | 50-75 ms a frame (13-20 fps) |
| First mission with the boat | 46-53 ms (19-22 fps) |
| Street chase with many cars and people | 100-165 ms (6-10 fps), single frames far longer |
| Outdoors on foot, the camp | 60-90 ms |
| Flying | 42-57 ms |
| Longest session so far | 41 minutes, no crash |
| Memory | 40-46 GB of private memory: **64 GB of RAM recommended** |

It started at 280 ms a frame in the same scenes two days earlier.

What is known to be wrong:

- Trees, plants and far buildings flicker outdoors in some places.
- Rooms with little direct light are too dark without `-RayTracing`.
- Textures of what just came into view appear late while the title streams.
- The first run with a new driver or a new build stutters while the driver makes
  pipelines (`victory warm` makes them ahead).
- Recording a frame in the world makes the title's own watchdog end it.

## How to run it

You need Windows 10/11 x64, a CPU with AVX2, a Direct3D 12 GPU with shader
model 6.6 and mesh shaders, Rust (the toolchain is pinned), the MSVC build
tools, and your own dump of the title.

```powershell
git clone https://github.com/avatarmanz/VIctory-7900XTX
cd VIctory-7900XTX
copy victory.example.toml victory.toml      # then set your paths, outside this folder
cargo build --locked --profile diag -p victory --features diagnostics
scripts\play.ps1 -Exe <path of your copy's executable>
```

`scripts\play.ps1 -Safe` leaves out the newest, faster ways; try it first when
something is wrong. `-RayTracing` turns the title's ray-traced indirect light on.

Before the first run, ahead of time (both can be stopped and go on later):

```powershell
target\diag\victory.exe shaders <the shader archives>   # every shader translated, minutes
target\diag\victory.exe warm --used                     # pipelines earlier runs drew with, for a new driver or build
```

Commands, debugging switches and upstream's own description:
[docs/README-fxceless.md](docs/README-fxceless.md). Every `VICTORY_...` switch is
described in a comment where it is read.

## What is different here

In short; the reasons and the measurements are in [docs/findings.md](docs/findings.md).

- **List calls are written down, not carried out, on the title's threads**
  (`VICTORY_THIN`): eight threads of ours translate them.
- **Submissions are prepared by a thread of ours and joined**
  (`VICTORY_ASYNC_SUBMIT`, `VICTORY_JOIN_PARTS`), with what can be spread over
  threads spread: refreshes, compares (AVX2), descriptor records, landings.
- **What shaders wrote comes back beside the write guards**, by the dwords that
  changed, on a thread of its own (`VICTORY_RELAY`, `VICTORY_LAND`).
- **Buffer fills that need bytes still on the card are done on the card**
  (`VICTORY_CARD_STEPS`) instead of waiting for it.
- **The title's writes are caught more cheaply**: one view opened at a fault,
  blocks that rest, places in the title's code that look before they write.
- **Textures go up on a copy queue of their own** and a slot shows one only after
  its upload ran, which also ended a device loss that came every 15 minutes or so.
- **Boats float**: textures that shaders write and the title's CPU reads go back
  to title memory (from fxceless and Geckomacabre, on by default here).
- **Frame record and replay** with per-pass times and shader variants compared
  turn by turn ([docs/replay.md](docs/replay.md)).

## Credits

This is a tree of other people's work with more on top. The history is kept, and
commits taken from elsewhere name the commit and its author.

- **[VIctory](https://github.com/Marvisak/VIctory)** by Marvisak and its
  contributors: the loader, the runtime services, the Direct3D 12 back end and
  the shader recompiler.
- **[fxceless/victory-fork](https://github.com/fxceless/victory-fork)**: the fork
  this one started from, and the people whose pull requests it carries.
- **[DexNom/VIctory](https://github.com/DexNom/VIctory)**: ray tracing fixes,
  cube gradients, sparse textures.
- **[Belobug/VIctory-Belobug-AMD](https://github.com/Belobug/VIctory-Belobug-AMD)**:
  the ambient light idea, pipeline workers, and work by romainhedouin.
- **[FQY55kai/ZJOU-VICTORY](https://github.com/FQY55kai/ZJOU-VICTORY)**.
- **[myokex/victory-amd](https://github.com/myokex/victory-amd)**: the guard over
  every view of a page (with Spexivus, joroyt and tightfall), and Joshua Odom's
  reads into title memory.
- **Geckomacabre** and **cxy0072019-sketch**: the texture read-back.
- **[XCL](https://github.com/ivall/xcl)** (GPL-2.0-only) was read for how the same
  title behaves under another layer; no code of it is here.
- This fork's changes were made with [Claude Code](https://claude.com/claude-code).

## License

GPL-3.0-or-later, as VIctory. See [LICENSE](LICENSE); the one file that is not
covered is named in [docs/asset-policy.md](docs/asset-policy.md).
