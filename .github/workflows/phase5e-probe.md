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

*TBD*

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
