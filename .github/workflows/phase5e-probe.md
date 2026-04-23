# `phase5e-probe.yml` — how we rebuild SBFspot's Linux tarballs

Companion doc for
[`phase5e-probe.yml`](phase5e-probe.yml). Written so someone who has
never touched GitHub Actions, never cross-compiled, and doesn't know
what PIE or RELRO stand for can still follow along. If you already
know those terms, skip ahead — headings are dense enough to navigate.

**Status:** current pipeline as of April 2026. All 15 cells produce
modern, hardened, reproducible binaries that are drop-in replacements
for upstream V3.9.12's hand-built tarballs.

**Companion forensic docs** in [`../docs/`](../docs/) record every
experiment that led here. This doc is forward-looking: "why does the
workflow look like it does today." The phase baselines are backward-
looking: "what did we learn on the way here."

---

## Reading order

Each section is self-contained, but they build on each other.

1. [What we're actually trying to do](#1-what-were-actually-trying-to-do)
2. [The shape of a run: runners, chroots, the 15 cells](#2-the-shape-of-a-run)
3. [Step-by-step walkthrough](#3-step-by-step-walkthrough)
4. [Why each compile flag](#4-why-each-compile-flag)
5. [Why each link flag](#5-why-each-link-flag)
6. [The PIC libbluetooth rebuild](#6-the-pic-libbluetooth-rebuild)
7. [Per-cell matrix table](#7-per-cell-matrix-table)
8. [Things we tried and dropped](#8-things-we-tried-and-dropped)
9. [Glossary](#9-glossary)

---

## 1. What we're actually trying to do

### The upstream problem

SBFspot is a small C++ program that reads power-production data from
SMA solar inverters. Every release, the upstream maintainer publishes
**15 Linux release tarballs**: one for each combination of

- 3 database variants — `sqlite`, `nosql`, `mariadb`
- 3 Debian/Raspbian codenames — `buster`, `bullseye`, `bookworm`
- 2 architectures — `arm` (32-bit armhf, for Raspberry Pi) and
  `arm64` (aarch64, for modern Pi / generic ARM servers)

…minus `buster × arm64`, which upstream never published. So
3 × 3 × 2 − (1 × 1 × 3) = **15 tarballs**.

Each tarball is built by hand on the maintainer's Windows machine,
cross-compiled against a sysroot for the relevant Debian version.
That's a legitimate way to ship releases, but it has two downsides:

1. **Not auditable.** Nobody outside the maintainer's head knows
   exactly which compiler, which flags, which source patches produced
   any given tarball. The SHA256 on the release page only tells you
   that the file hasn't changed — not that it was built the way you
   expect.
2. **Not reproducible.** If the maintainer gets hit by a bus
   tomorrow, there is no recipe for rebuilding V3.9.13.

### What this workflow is

`phase5e-probe.yml` is our **reproducible replacement** for that
Windows build process. On every run:

- GitHub Actions spins up 15 parallel Linux VMs (one per cell).
- Each VM creates a clean Debian/Raspbian chroot matching the cell's
  codename + arch (e.g. `bookworm × armhf`).
- Inside the chroot, we `git checkout` upstream's source at tag
  `V3.9.12` and build the binary with carefully chosen compile + link
  flags.
- We then fetch the *official* upstream tarball for that same combo,
  strip both binaries of metadata that's known to differ for trivial
  reasons (build timestamps, build-ID hashes), and compare byte sizes.

### What we're **not** doing anymore

We spent phases 2–5d trying to reproduce upstream's binaries
byte-for-byte. We got 7 of 25 binaries byte-identical and
characterised every residual. The remaining ~16 KB gap on
`arm × bookworm` traces to upstream's private Windows cross-toolchain
(see [`../docs/phase5d-baseline.md`](../docs/phase5d-baseline.md)) —
unreachable without the maintainer handing us their cross-gcc.

Phase 5e **stopped chasing byte-match** and pivoted to a better
goal: produce binaries that are *functionally indistinguishable* from
upstream's (same behaviour, same NEEDED libraries, no new runtime
deps) but built with **modern hardening flags** that upstream's
circa-2020 Windows cross-compiler doesn't emit by default.

The workflow still reports the byte-delta vs upstream as a warning
annotation per cell — but that's now informational, not a pass/fail
gate.

## 2. The shape of a run

### Runners

A **runner** in GitHub Actions is just a Linux VM that GitHub starts
up for the duration of a job. When we write:

```yaml
runs-on: ubuntu-24.04-arm
```

we're saying "give me a freshly-booted Ubuntu 24.04 VM whose CPU is
**aarch64** (64-bit ARM)." This is a real hardware VM on Azure's
Ampere Altra silicon, not an emulated one. That matters because:

- 64-bit ARM chips can execute 32-bit ARM (armhf) binaries
  **natively** via the AArch32 execution mode built into the CPU.
  So a single runner can build and test *both* our arm64 and our
  armhf cells, with no QEMU emulation slowing things down.
- We verified this experimentally in Phase 0
  ([`../docs/` via CLAUDE.md](../CLAUDE.md)); a 32-bit armhf `gcc`
  runs on this silicon and produces binaries that execute there too.

### Chroots via `debootstrap`

Debian/Raspbian ship new library versions every release. Upstream's
`buster` tarball is linked against buster's libboost, buster's
libsqlite3, buster's glibc. We can't build a buster tarball on a
bookworm host and expect the library versions to match.

So for each cell, we create a **chroot** — short for "change root" —
a self-contained tree that looks like a fresh minimal Debian install
of the cell's target codename. The command that builds it is
[`debootstrap`](https://wiki.debian.org/Debootstrap):

```
sudo debootstrap --arch=armhf --variant=minbase \
    --keyring=$KEYRING --include=ca-certificates \
    bookworm rootfs/ http://archive.raspbian.org/raspbian
```

That downloads all of bookworm-armhf's base packages into `./rootfs/`.
After that, `sudo chroot rootfs /bin/sh` drops you into a shell
where `/usr/lib`, `/usr/bin`, and `gcc` are **bookworm-armhf's**, not
the host's. It's isolation without the overhead of a full container.

### Mirrors

`debootstrap` pulls packages from a mirror URL. We use three:

| Mirror | For which cells | Why |
|---|---|---|
| `http://archive.raspbian.org/raspbian` | arm × {bullseye, bookworm} | Current Raspbian archive. Upstream's armhf binaries are Raspbian-flavoured (`.comment` says `GCC: (Raspbian X+rpi1)`), so we use Raspbian too. |
| `http://legacy.raspbian.org/raspbian` | arm × buster | Raspbian dropped buster from the live archive; legacy mirror still serves it. |
| `http://snapshot.debian.org/archive/debian/20250222T000000Z` | arm64 × all | Debian's snapshot service, pinned to the date upstream released V3.9.12. Arm64 binaries are Debian, not Raspbian. |

Raspbian doesn't have a snapshot service that works (Phase 0 finding),
so we accept the live archive's natural drift over time. If bookworm
library versions shift, our binaries shift with them — same as
upstream's would if they rebuilt today.

### The 15-cell matrix

A GitHub Actions **matrix** is an instruction that says "run this job
N times with different inputs." Our matrix block has one row per
cell:

```yaml
strategy:
  fail-fast: false      # don't cancel sibling cells if one fails
  matrix:
    cell:
      - id: sqlite-arm-buster
        codename: buster
        debarch: armhf
        ...
      - id: nosql-arm-buster
        ...
      # 13 more
```

GitHub takes that list, creates 15 independent jobs, and schedules
them in parallel. Each job gets a fresh `ubuntu-24.04-arm` runner
and runs the same set of steps — but with different matrix values
filled in via `${{ matrix.cell.codename }}`, etc.

Wall-clock per full matrix run: 3–5 minutes. Every cell does its own
debootstrap + compile + link + smoke-test independently.

### Triggering the workflow

The workflow has two triggers:

```yaml
on:
  workflow_dispatch:           # "Run workflow" button in the UI
  push:
    branches: [claude/add-ci-release-pipeline-QKIcZ]
    paths: [.github/workflows/.phase5e-probe-trigger]
```

That `paths:` filter is the "frozen probe" gate explained in
[`README.md`](README.md): the workflow only runs on push if the
trigger file itself changed. Touching its timestamp
(`date -u +"%Y-%m-%dT%H:%M:%SZ" > .phase5e-probe-trigger`) is enough
to fire it.

We do this so day-to-day commits don't consume CI minutes on probes
that don't need re-running.

## 3. Step-by-step walkthrough

Every cell runs the same 11 steps. Here's what each one does, in
order, in plain language. Flag rationale is in sections 4–6; this
section is just "what happens when."

### Step 1: `Install host-side tooling`

Runs on the outer runner (not the chroot). Installs `debootstrap`,
`binutils`, `file`, `curl`, `jq`, and the Debian archive keyring.
Everything we need to *build* a chroot and later *compare* binaries
lives here.

### Step 2: `Fetch Raspbian keyring` (conditional)

Runs only for arm cells (which use Raspbian mirrors). Debootstrap
won't trust a Raspbian mirror unless we give it Raspbian's signing
key. Downloads `raspbian.public.key`, dearmors it into a binary
keyring file that debootstrap accepts.

### Step 3: `Debootstrap rootfs`

This is where the chroot gets built. Picks the right mirror URL +
keyring based on `matrix.cell.mirror` (`raspbian` / `raspbian-legacy`
/ `debian`) and runs debootstrap. When it finishes, `./rootfs/` is a
minimal bookworm-armhf (or whatever the cell is) install.

This step takes ~60–90 s — the bulk of each cell's runtime.

### Step 4: `Check out upstream V3.9.12`

Uses `actions/checkout@v6` to clone the *upstream* SBFspot repo
(`SBFspot/SBFspot`) at tag `V3.9.12` into `./upstream/`. Not our
fork — we want the exact upstream source we're trying to reproduce.

### Step 5: `Stage source tree inside rootfs`

`cp -a upstream/SBFspot rootfs/src/` and the same for
`SBFspotUploadCommon` (and `SBFspotUploadDaemon` if the cell has
a daemon — `nosql` doesn't). After this, the source is inside the
chroot, ready to compile with the chroot's g++.

### Step 6: `Install hardened compiler wrapper`

This one's sneaky. We `sudo tee rootfs/usr/local/bin/g++` to create
a small shell script that *intercepts* calls to `g++` inside the
chroot. The real g++ lives at `/usr/bin/g++`; our wrapper at
`/usr/local/bin/g++` comes first in `$PATH` and forwards every call
to the real g++ with extra flags appended.

```sh
#!/bin/sh
exec /usr/bin/g++ \
  '-fmacro-prefix-map=/usr/include=<upstream-prefix>' \
  -D_FORTIFY_SOURCE=2 \
  -fstack-protector-strong \
  -D_GLIBCXX_ASSERTIONS \
  -fPIE \
  "$@"
```

Why a wrapper instead of editing the Makefile? Because we have a
hard rule in this fork: **no C++ source changes, no Makefile
changes**. The wrapper is a clean side-channel: the source tree is
byte-identical to upstream, but every `g++` invocation gets our
hardening flags.

Rationale for each flag is in [§4](#4-why-each-compile-flag).

### Step 7: `Install build deps in chroot`

Enters the chroot with `sudo chroot rootfs /bin/sh`, runs
`apt-get install` for:

- `g++`, `make` — the compiler, which ends up at `/usr/bin/g++`
  (then shadowed by our wrapper).
- `libbluetooth-dev`, `libboost-date-time-dev`, `libboost-system-dev`
  — libraries SBFspot links against.
- `libsqlite3-dev` or `libmariadb-dev{,-compat}` per cell (nosql
  gets none).
- `libcurl4-openssl-dev` or `libcurl4-gnutls-dev` per cell (for the
  upload daemon only).
- `binutils`, `file` — for inspecting the built binary.

### Step 8: `Build PIC libbluetooth.a from distro bluez source`

Deep-dive in [§6](#6-the-pic-libbluetooth-rebuild). One sentence:
Debian ships `libbluetooth.a` but compiles it without `-fPIC`, which
breaks our PIE binaries on 32-bit ARM. We rebuild the same 3 source
files Debian used, with `-fPIC`, and install over the distro one.

### Step 9: `Build SBFspot (+ daemon if applicable)`

The actual build. `cd /src/SBFspot && make $target LDFLAGS=…` inside
the chroot, with our constructed LDFLAGS string passed explicitly.
`make` runs g++ (which is actually our wrapper → real g++ with extra
compile flags) on every `.cpp`, then links the `.o` files into the
final binary with the hardening link flags we pass.

`readelf -d` is run on the binary immediately, showing the dynamic
section's `NEEDED` entries so you can see in the log which shared
libraries the binary requires.

For cells where `has_daemon: true`, the same dance is repeated for
`SBFspotUploadDaemon` from `/src/SBFspotUploadDaemon`.

Rationale for the link flags in [§5](#5-why-each-link-flag).

### Step 10: `Smoke test — binary runs and reports version`

Actually executes the freshly-built binary in the chroot:

```
./sqlite/bin/SBFspot --version
```

We want the log to contain "SBFspot V3.9.12" + the architecture
banner. For armhf cells this is running 32-bit ARM code on
aarch64 silicon — the CPU's AArch32 mode. If anything is wrong
(missing library, broken linker script, hardening flag that aborts
startup), this step catches it before we waste time on the compare.

Also runs `readelf -l | grep GNU_RELRO|GNU_STACK|PIE|INTERP` and
`readelf -d | grep BIND_NOW|FLAGS_1` to confirm the hardening flags
took effect.

### Step 11: `Download upstream asset` + `Normalised-ELF compare + report`

Fetches the official tarball for this cell from
`https://api.github.com/repos/SBFspot/SBFspot/releases/tags/V3.9.12`,
unpacks it, and compares sizes between our just-built binary and
upstream's.

**Normalisation** means: we copy both binaries, then run
`strip --strip-all` and `objcopy --remove-section=.comment
--remove-section=.note.gnu.build-id` on both copies. That removes
metadata that's guaranteed to differ (build timestamps, debug symbols,
SHA1 build-IDs, compiler-version strings) so the compare reflects
the *code* difference, not formatting.

If sizes/hashes match: `::notice title=<cell>::<bin> MATCH size=…`
shows up as a green annotation.
If they differ: `::warning title=<cell>::<bin> residual=<N> B`
shows the byte-delta. Both warnings and notices are informational in
Phase 5e — we're not failing the run on them anymore.

### Step 12: `Upload per-cell artefacts`

Tars up `./out/` (which has `out/ours/`, `out/upstream/`,
`out/summary.txt`) as an artefact named `phase5e-<cell-id>`.
`actions/upload-artifact@v7` handles the upload. 90-day retention.

That's the whole pipeline. The next three sections explain *why*
each compile/link/libbluetooth choice is what it is.

## 4. Why each compile flag

*TBD*

## 5. Why each link flag

*TBD*

## 6. The PIC libbluetooth rebuild

*TBD*

## 7. Per-cell matrix table

*TBD*

## 8. Things we tried and dropped

*TBD*

## 9. Glossary

*TBD*
