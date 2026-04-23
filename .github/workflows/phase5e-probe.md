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

*TBD*

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
