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
 9. [Reproducibility guarantees](#9-reproducibility-guarantees)
10. [Glossary](#10-glossary)

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

Every cell runs the same 13 steps. Here's what each one does, in
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

### Step 5: `Derive SOURCE_DATE_EPOCH from release tag`

Reads the commit timestamp of the V3.9.12 tag
(`git -C upstream log -1 --pretty=%ct V3.9.12`) and exports it to
`$GITHUB_ENV` so every subsequent step sees it in the environment.

`SOURCE_DATE_EPOCH` is a
[reproducible-builds.org convention](https://reproducible-builds.org/docs/source-date-epoch/):
a single Unix timestamp that build tools substitute for "now"
wherever they'd otherwise embed a build-time date. `gcc` respects it
for `__DATE__`/`__TIME__` macros, `tar` for `--mtime=@$SOURCE_DATE_EPOCH`,
`gzip` via `-n`, `ar` + `objcopy` (binutils ≥ 2.35) for embedded
timestamps in archives and build-IDs.

The value is derived from the tag itself, not configured separately:
same input (tagged commit) → same output (tarball), forever. See
[§9](#9-reproducibility-guarantees) for what we do with it.

### Step 6: `Stage source tree inside rootfs`

`cp -a upstream/SBFspot rootfs/src/` and the same for
`SBFspotUploadCommon` (and `SBFspotUploadDaemon` if the cell has
a daemon — `nosql` doesn't). After this, the source is inside the
chroot, ready to compile with the chroot's g++.

### Step 7: `Install hardened compiler wrapper`

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

### Step 8: `Install build deps in chroot`

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

### Step 9: `Build PIC libbluetooth.a from distro bluez source`

Deep-dive in [§6](#6-the-pic-libbluetooth-rebuild). One sentence:
Debian ships `libbluetooth.a` but compiles it without `-fPIC`, which
breaks our PIE binaries on 32-bit ARM. We rebuild the same 3 source
files Debian used, with `-fPIC`, and install over the distro one.

### Step 10: `Build SBFspot (+ daemon if applicable)`

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

### Step 11: `Smoke test — binary runs and reports version`

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

### Step 12: `Download upstream asset` + `Normalised-ELF compare + report`

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

### Step 13: `Assemble reproducible release tarball`

Takes the fresh binary (+ daemon where applicable) + the
non-binary data files from the tagged source tree, stages them in
a temporary directory with sensible modes (0755 binaries / 0644
data), and `tar`s the lot up deterministically.

Tar flags chosen for reproducibility:
- `--sort=name` — member order is fixed (lexicographic).
- `--owner=0 --group=0 --numeric-owner` — no host-specific uid/gid.
- `--mtime=@${SOURCE_DATE_EPOCH}` — every member has the tag's commit time.
- `--format=ustar` — the old POSIX tar format, no GNU extensions
  that could shift across tar versions.

Piped into `gzip -n`, which skips the gzip-header "original filename"
and "modification time" fields (otherwise gzip embeds its own
wall-clock `mtime`).

Result: `out/tarballs/sbfspot-<db>-<arch>-linux-<codename>.tar.gz` —
byte-identical across runs given identical inputs. See
[§9](#9-reproducibility-guarantees) for measurement.

Per-variant member list:
- **nosql** (9 members): `SBFspot` + `SBFspot.default.cfg` +
  `date_time_zonespec.csv` + 6× `TagListXX-XX.txt`.
- **sqlite** (12): nosql set + `SBFspotUploadDaemon` +
  `SBFspotUpload.default.cfg` + `CreateSQLiteDB.sql`.
- **mariadb** (13): nosql set + `SBFspotUploadDaemon` +
  `SBFspotUpload.default.cfg` + `CreateMySQLDB.sql` +
  `CreateMySQLUser.sql`.

Two files get renamed during staging: `SBFspot.cfg` →
`SBFspot.default.cfg` and `SBFspotUpload.cfg` →
`SBFspotUpload.default.cfg`. Upstream convention: "default" in the
filename so `sbfspot-config`'s install flow doesn't overwrite a
user's edited copy on upgrade.

### Step 14: `Upload per-cell artefacts`

Tars up `./out/` (which now has `out/ours/`, `out/upstream/`,
`out/tarballs/`, `out/summary.txt`) as an artefact named
`phase5e-<cell-id>`. `actions/upload-artifact@v7` handles the upload.
90-day retention.

That's the whole pipeline. The next three sections explain *why*
each compile/link/libbluetooth choice is what it is.

## 4. Why each compile flag

These are the flags applied by the `/usr/local/bin/g++` wrapper
(step 6). They're added to *every* C++ compilation in the build.

### `-fmacro-prefix-map=/usr/include=<upstream-prefix>`

**What it does:** tells the preprocessor "whenever you bake the
current source file's path into the binary via the `__FILE__` macro,
substitute `/usr/include` with this other string."

**Why we need it:** C and C++ code can embed its own file paths via
things like `assert()` macros (which expand to
`__assert_fail("x != NULL", "/usr/include/…/bits/stl_vector.h", …)`).
Those paths end up as string literals inside the `.rodata` section
of the compiled binary. If our paths differ from upstream's — say
we have `/usr/include/c++/12/vector` and upstream has
`d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include\c++\12\vector`
— the binaries differ byte-for-byte just because of those paths.

We discovered this mechanism in Phase 3.4. Each cell's matrix entry
has a `prefix:` field with the exact Windows-style path upstream
uses; the wrapper substitutes it in. The binary ends up with
upstream's paths even though we're on Linux. Weird, but it's the
cheapest way to match a known reproducibility quirk.

**Why we still do it even though we've given up byte-match:**
low cost, keeps the binary self-consistent with upstream for anyone
doing `strings SBFspot | grep rpi` for forensic reasons, and leaves
the door open to future byte-match work if upstream publishes their
toolchain.

### `-D_FORTIFY_SOURCE=2`

**What it does:** tells glibc to replace certain standard-library
functions (`memcpy`, `strcpy`, `sprintf`, `read`, …) with
bounds-checking versions when the compiler can statically prove a
buffer size. At runtime, if a check fails, the program calls
`__chk_fail()` and aborts.

**Why:** catches buffer overflows that would otherwise corrupt memory
silently. This is the Debian hardening-wrapper default for every
Debian-built binary since buster; we match it.

**Behaviour change risk:** if SBFspot has a latent buffer overflow
(it's mature C++, we don't expect any), hardened builds abort where
upstream's would silently misbehave. That's strictly an improvement,
but worth knowing.

**Why `=2` and not `=3`:** `=3` (glibc 2.34+) adds more aggressive
checks that rely on `__builtin_dynamic_object_size`, which only
works well on gcc ≥ 12. Our buster cells use gcc 8.3, which doesn't
support `=3`. `=2` works everywhere.

### `-fstack-protector-strong`

**What it does:** emits a "canary" — a random value written to the
stack between local variables and the saved return address — in
every function that has stack buffers, arrays, or uses `alloca()`.
The canary is checked at function return; if corrupted, the program
aborts with `*** stack smashing detected ***`.

**Why:** classic defense against stack-based buffer overflows
exploited to overwrite return addresses (the mechanism behind many
classic CVEs). Debian default since stretch.

**Alternatives considered:**
- `-fstack-protector` (only functions with ≥8-byte char buffers) —
  too narrow.
- `-fstack-protector-all` — every function — unnecessarily slow;
  nobody uses it in production.
- `-fstack-protector-strong` is the Debian/Ubuntu default.

### `-D_GLIBCXX_ASSERTIONS`

**What it does:** turns on runtime debug-assertions inside libstdc++
(the GNU C++ standard library). Things like `std::vector::operator[]`
gain bounds checks; `std::list::front()` aborts on empty list;
iterator comparisons sanity-check.

**Why:** catches C++ standard-library misuse at runtime instead of
letting it corrupt memory. Again, Debian default.

**Cost:** some code paths get slightly slower. For SBFspot's
workload (a few dozen solar readings per minute), imperceptible.

### `-fPIE`

**What it does:** emits *position-independent code* suitable for
linking into an executable that will be loaded at a randomized
address (ASLR). Paired with `-pie` at link time to actually produce
a PIE executable.

**Why we need the `-f` part and the `-` part:**
- `-fPIE` (compile flag): tells gcc to generate relocation-table
  references instead of absolute addresses for globals and functions,
  so the code works at any load address.
- `-pie` (link flag, in LDFLAGS): tells ld to produce an ELF of type
  `ET_DYN` (shared object, executable) instead of `ET_EXEC` (fixed-
  address executable). Without it, PIE codegen just adds overhead
  without enabling ASLR.

**Why PIE at all:** address-space layout randomisation. Each time
the binary starts, it loads at a different base address. An attacker
who finds a code-execution vulnerability then has to also leak the
random offset to do anything useful. Debian default for all new
binaries since buster.

**Upstream doesn't do this on armhf.** Upstream's cross-toolchain
predates Debian's PIE default for armhf, so upstream's armhf
binaries are non-PIE (`ELF … executable`, not `pie executable`).
Upstream's arm64 binaries are PIE.

Going PIE everywhere is one of our deliberate divergences from
upstream's output. It doesn't change functional behaviour — the
binary still reads the same configs, talks the same protocols — but
an auditor running `checksec` will see PIE on every cell instead
of only arm64. See [§8](#8-things-we-tried-and-dropped) for the
option where we considered matching upstream's non-PIE on armhf.

### Flags we *don't* apply

- **`-O3`:** Upstream's Makefile uses `-O2`. We leave that alone.
  `-O3` trades code size + compile time for marginal runtime gains
  that don't matter for SBFspot's polling workload.
- **`-flto`:** Link-time optimisation. Gains negligible on a program
  this size, and would complicate the link step.
- **`-march=native`:** We're cross-building for generic armhf /
  aarch64, not the build host. `-march=native` on the runner would
  produce binaries that crash on older Pis.
- **`-fsanitize=address`** / `-fsanitize=undefined`: debugging tools,
  not production-shippable (they require runtime library support).

### Flags we only apply via the upstream Makefile

The Makefile adds `-c -Wall -O2 -Wno-unused-local-typedefs
-Wno-psabi`. Those stay. `-Wno-psabi` silences a noisy warning about
C++ ABI changes between gcc versions for armhf; it's cosmetic.

## 5. Why each link flag

LDFLAGS for each cell is built in step 9 of the workflow from three
pieces:

```
SBFSPOT_LDFLAGS = $common_ldflags $HARDEN $sbfspot_extra
```

where:

- `common_ldflags` = `-s -Wl,--as-needed` (same on every cell)
- `HARDEN` = `-pie -Wl,-z,relro -Wl,-z,now -Wl,-z,noexecstack -Wl,--build-id=sha1`
- `sbfspot_extra` = `-Wl,-Bstatic -lbluetooth -Wl,-Bdynamic` (same on every cell)
- `daemon_extra` = `''` (empty, same on every cell — daemon doesn't need bluez)

### A note on the `-Wl,…` syntax

gcc passes most `-X` options to its own internal machinery. To pass
a flag through to the **linker** (`ld`), you prefix it with `-Wl,`.
E.g. `-Wl,-z,now` becomes `-z now` when ld sees it. That's why all
the linker-specific flags below look weird.

### `-s`

**What:** strip all symbol and debug info from the output binary.

**Why:** matches upstream's Makefile default, shrinks the binary.
Debug info is useless to end users and bloats the tarball. We could
instead produce a separate `.debug` sidecar (planned Phase 5e.3) —
until then, symbols are discarded.

### `-Wl,--as-needed`

**What:** for each `-lX` the linker encounters, only emit a
DT_NEEDED entry for `libX.so` if at least one symbol from that
library is actually used by the code.

**Why:** SBFspot's Makefile ends its link line with
`-Wl,-Bdynamic $(addprefix -l,$(LIBS))`, which lists every library
the program *might* need. Without `--as-needed`, every `-l` adds a
NEEDED entry regardless of whether it's used — which would re-add
`libbluetooth.so.3` to NEEDED after our static-libbluetooth dance,
since the Makefile's trailing `-lbluetooth` comes *after* our
static-link block.

**Why not rely on the default:** Debian bullseye and bookworm
default to `--as-needed`, but **buster defaults to `--no-as-needed`**.
Without this flag explicit, our buster cells had `libbluetooth.so.3`
in NEEDED despite the static-link. Adding `-Wl,--as-needed`
explicitly makes all 15 cells behave identically regardless of
distro default.

### `-pie`

**What:** produce an ELF of type `ET_DYN` with an `INTERP` segment,
i.e. a position-independent executable.

**Why:** see `-fPIE` rationale in §4. `-fPIE` (compile) + `-pie`
(link) together enable ASLR at runtime.

### `-Wl,-z,relro`

**What:** GNU_RELRO — tells the dynamic linker "mark certain
segments (the GOT, initialised data that shouldn't change after
startup) read-only after all relocations are done at program
startup."

**Why:** mitigates attacks that overwrite the Global Offset Table
to hijack function pointer resolution. Classic technique in
exploit chains.

### `-Wl,-z,now`

**What:** BIND_NOW — resolve *all* dynamic symbols at program
startup, not lazily on first call.

**Why:** enables "full RELRO" — together with `-z,relro`, means the
GOT is fully populated and read-only from startup onwards. Lazy
binding (the default) requires the GOT to stay writable for the
program's lifetime, which weakens RELRO. Debian hardening default.

Slight startup-time cost (all symbols resolved up front). For a
long-lived program like SBFspot, this is a one-time cost dwarfed by
normal operation.

### `-Wl,-z,noexecstack`

**What:** marks the stack as non-executable via a PT_GNU_STACK ELF
segment with flags `RW` (not `RWE`).

**Why:** prevents attackers from jumping to injected shellcode on
the stack. Modern kernels already enforce NX stacks by default on
x86_64/aarch64, but the flag must be on the binary to be honoured.

**Fun history:** C code that nests functions (a GCC extension) or
uses `__builtin_trampoline` can force the stack executable. SBFspot
doesn't use those, so the flag is safe.

### `-Wl,--build-id=sha1`

**What:** embed a SHA1 hash of the binary's contents into a
`.note.gnu.build-id` section. Shows up in `file` output as
`BuildID[sha1]=abcdef…`.

**Why:** lets core dumps, `perf`, `gdb`, and `debuginfod` uniquely
identify the binary even after stripping. Deterministic: same bytes
in → same build-id out. Useful if we ever publish `.debug` sidecars
(they'll match by build-id).

**Alternative:** `--build-id=none` strips the section entirely.
Upstream's arm-bookworm binaries use that — probably a Windows
cross-linker default. We diverge; having a build-id is strictly
better for forensics.

### `-Wl,-Bstatic -lbluetooth -Wl,-Bdynamic` (in `sbfspot_extra`)

**What:**
- `-Wl,-Bstatic` flips the linker's search mode to "only look for
  `.a` archives" (static libraries).
- `-lbluetooth` causes the linker to search for `libbluetooth.a`
  (or `libbluetooth.so` if static mode weren't active) in its library
  path, find it at `/usr/lib/<triplet>/libbluetooth.a`, and pull in
  all object files from it whose symbols are needed.
- `-Wl,-Bdynamic` flips back to default mode (prefer `.so`) for
  everything that comes after.

**Why static libbluetooth:** matches upstream's output. All 15
upstream binaries have libbluetooth statically embedded (no
`libbluetooth.so.3` in NEEDED). This is a *portability* property
— it means users don't need the `bluez` package installed on their
target system. Rasperry Pi OS Lite doesn't include bluez by default,
and `sbfspot-config` (upstream's installer) doesn't install it
either. If our binary dynamically linked libbluetooth, users on
Pi OS Lite couldn't even `exec` the binary.

**Why it's in `sbfspot_extra` and not `common_ldflags`:** the
daemon doesn't use libbluetooth, only SBFspot itself does. Keeping
the bluetooth-specific flag out of the daemon's LDFLAGS keeps the
daemon's NEEDED list clean.

**Why the static `.a` has to be PIC-rebuilt first:** see [§6](#6-the-pic-libbluetooth-rebuild).

### Flags we could add but don't

- **`-Wl,-z,separate-code`:** separates code and read-only data into
  distinct PT_LOAD segments for finer-grained page permissions. Some
  performance benefit, better security. Minor compatibility issues
  on older kernels; skipped to keep the pipeline simple.
- **`-Wl,-z,pack-relative-relocs`:** binary-size win on PIE via
  compressed R_*_RELATIVE sequences. Requires glibc ≥ 2.36;
  bookworm has 2.36 but bullseye and buster don't. Would complicate
  per-cell conditionals.
- **`-flto`:** addressed in §4.

### A note on `-lpthread`, `-lm`, `-lc`

These come in automatically via the compiler driver / Makefile's
LIBS list. We don't need to manage them. `-lm` is pulled in by
libstdc++'s math usage; `-lc` is implicit; `-lpthread` is in
`LIBS := pthread …` in the Makefile.

## 6. The PIC libbluetooth rebuild

This is the trickiest part of the workflow. Understanding why we
need it requires understanding two concepts: PIC vs PIE, and
text relocations.

### PIC vs PIE — one-paragraph primer

**PIC** stands for Position-Independent **Code**. Compiler setting
(`-fPIC`). Code that doesn't hard-code absolute addresses for
functions or globals — instead it looks them up through a table
(the Global Offset Table, GOT) that the loader fills in at runtime.
Required for shared libraries (`.so`), because a `.so` can be loaded
at any address.

**PIE** stands for Position-Independent **Executable**. Linker
output type (`-pie`). An executable that can be loaded at a
randomized address (ASLR). Requires all its code to be PIC.

So: **every piece of code that goes into a PIE binary must be PIC.**

### What breaks

Debian ships `libbluetooth.a` (static archive) in the
`libbluetooth-dev` package. But Debian builds that `.a` **without
`-fPIC`** on armhf — the object files inside use absolute 32-bit
addresses for their globals.

When our linker tries to static-link that non-PIC `.a` into a PIE
binary:

- On **arm64**: the aarch64 instruction set uses PC-relative
  addressing natively, so the linker can transparently convert
  absolute references into PC-relative ones during link. No drama.
- On **armhf**: the linker has no choice but to emit *text
  relocations* — entries in the dynamic section that tell the
  loader "at startup, patch this specific byte in the code segment
  with the absolute address of symbol X." That sets the ELF
  `DT_TEXTREL` flag and makes the code segment writable at load
  time, which breaks RELRO and is universally flagged by security
  auditors.

We hit this exact bug in Phase 5e.1: `readelf -d` on every armhf
cell showed `FLAGS: TEXTREL BIND_NOW`. Arm64 cells showed clean
`FLAGS: BIND_NOW`.

### What upstream does (that we don't)

Upstream's armhf binaries are non-PIE (`ELF … executable`, not
`pie executable`) — see Phase 1 fingerprint. A non-PIE executable
is loaded at a fixed address, so absolute addresses in non-PIC
`.a` files work fine. No TEXTREL, no warnings.

We chose not to follow that route because PIE is a meaningful
security win and we want modern-best-practice output. So we need
`libbluetooth.a` to be **PIC**, not non-PIC.

### What we tried first (5e.1b — rejected)

Simplest workaround: drop the static-link entirely, let
`libbluetooth.so.3` be a dynamic dep.

Result: TEXTREL gone, but `libbluetooth.so.3` appeared in NEEDED.
Users on Pi OS Lite (no `bluez` installed) couldn't run the binary
at all. Real portability regression; rejected.

### What we do instead (5e.1c — this workflow)

**Rebuild `libbluetooth.a` from the cell's own distro source, with
`-fPIC`, and install it over the distro's non-PIC one.**

Concretely, step 8:

```sh
# Enable deb-src so apt-get source works (debootstrap's sources.list
# only has binary-package entries by default).
sed -n 's/^deb /deb-src /p' /etc/apt/sources.list >> /etc/apt/sources.list
apt-get update -qq
apt-get install -y --no-install-recommends dpkg-dev

# Fetch the bluez source for *this codename* (matches the binary
# package's source exactly).
mkdir -p /tmp/bluez && cd /tmp/bluez
apt-get source bluez

# Build the same 3 source files Debian's Makefile.am says go into
# libbluetooth.la, with -fPIC. The sources are stable across
# bluez 5.50 / 5.55 / 5.66 (buster / bullseye / bookworm).
cd bluez-*/lib
gcc -fPIC -O2 -Wall -I. -c bluetooth.c hci.c sdp.c
ar rcs libbluetooth.a bluetooth.o hci.o sdp.o

# Install over the stock .a.
install -m 0644 libbluetooth.a /usr/lib/$TRIPLET/libbluetooth.a
```

The subsequent SBFspot build's `-Wl,-Bstatic -lbluetooth` picks up
our PIC `.a`, which static-links cleanly into the PIE executable
with no TEXTREL. `libbluetooth.so.3` stays out of NEEDED, matching
upstream's runtime-dep surface.

### Why we bypass autoconf

bluez is an autotools project — it normally builds via
`./configure && make`. `./configure` has a long list of checks and
disable flags; running it correctly across bluez 5.50 / 5.55 / 5.66
with all the right `--disable-*` options is finicky.

Bypassing it works because `lib/bluetooth.c` + `lib/hci.c` +
`lib/sdp.c` have a single `#ifdef HAVE_CONFIG_H` guard that skips
the generated `config.h` if we don't define it. No other autotools
magic is needed for just these three files.

### Why exactly these 3 files

From bluez's root `Makefile.am`:

```
lib_sources = lib/bluetooth.c lib/hci.c lib/sdp.c
lib_libbluetooth_la_SOURCES = $(lib_headers) $(lib_sources)
```

`libbluetooth.la` (the installed public library) is built from
`lib_sources` only — not `lib/uuid.c` or the other files under
`lib/`. We verified that the 3-file list is stable across bluez
5.50 (buster), 5.55 (bullseye), and 5.66 (bookworm).

SBFspot uses exactly one symbol from libbluetooth — `str2ba` — and
it's defined in `bluetooth.c`. So even our 3-file rebuild is
overkill for SBFspot specifically, but we want to match Debian's
library content exactly so any ABI-compatible caller sees the
same symbols.

### How we verify it worked

The step ends with:

```sh
readelf -r /usr/lib/$TRIPLET/libbluetooth.a | grep -E \
  'R_ARM_GOT|R_AARCH64_.*GOT|R_ARM_GOTOFF' | head -3
```

If the `.a` is PIC, relocations of those types (indirect-through-GOT)
will be present. If non-PIC, they won't be. We check in step 8 so
a regression surfaces immediately, not 3 steps later when the
SBFspot link produces TEXTREL.

### Cost

~30 s per cell for `apt-get update` + source fetch + the 3-file
compile + install. Tiny compared to the ~90 s debootstrap.

## 7. Per-cell matrix table

All 15 cells. Columns that are identical on every cell are omitted
(see "shared across all cells" below the table).

| cell id | codename | debarch | triplet | mirror | db_pkgs | curl_pkg | daemon |
|---|---|---|---|---|---|---|---|
| `sqlite-arm-buster` | buster | armhf | arm-linux-gnueabihf | raspbian-legacy | `libsqlite3-dev` | `libcurl4-gnutls-dev` | ✓ |
| `nosql-arm-buster` | buster | armhf | arm-linux-gnueabihf | raspbian-legacy | — | — | ✗ |
| `mariadb-arm-buster` | buster | armhf | arm-linux-gnueabihf | raspbian-legacy | `libmariadb-dev libmariadb-dev-compat` | `libcurl4-gnutls-dev` | ✓ |
| `sqlite-arm-bullseye` | bullseye | armhf | arm-linux-gnueabihf | raspbian | `libsqlite3-dev` | `libcurl4-openssl-dev` | ✓ |
| `nosql-arm-bullseye` | bullseye | armhf | arm-linux-gnueabihf | raspbian | — | — | ✗ |
| `mariadb-arm-bullseye` | bullseye | armhf | arm-linux-gnueabihf | raspbian | `libmariadb-dev libmariadb-dev-compat` | `libcurl4-openssl-dev` | ✓ |
| `sqlite-arm-bookworm` | bookworm | armhf | arm-linux-gnueabihf | raspbian | `libsqlite3-dev` | `libcurl4-openssl-dev` | ✓ |
| `nosql-arm-bookworm` | bookworm | armhf | arm-linux-gnueabihf | raspbian | — | — | ✗ |
| `mariadb-arm-bookworm` | bookworm | armhf | arm-linux-gnueabihf | raspbian | `libmariadb-dev libmariadb-dev-compat` | `libcurl4-openssl-dev` | ✓ |
| `sqlite-arm64-bullseye` | bullseye | arm64 | aarch64-linux-gnu | debian | `libsqlite3-dev` | `libcurl4-openssl-dev` | ✓ |
| `nosql-arm64-bullseye` | bullseye | arm64 | aarch64-linux-gnu | debian | — | — | ✗ |
| `mariadb-arm64-bullseye` | bullseye | arm64 | aarch64-linux-gnu | debian | `libmariadb-dev libmariadb-dev-compat` | `libcurl4-openssl-dev` | ✓ |
| `sqlite-arm64-bookworm` | bookworm | arm64 | aarch64-linux-gnu | debian | `libsqlite3-dev` | `libcurl4-openssl-dev` | ✓ |
| `nosql-arm64-bookworm` | bookworm | arm64 | aarch64-linux-gnu | debian | — | — | ✗ |
| `mariadb-arm64-bookworm` | bookworm | arm64 | aarch64-linux-gnu | debian | `libmariadb-dev libmariadb-dev-compat` | `libcurl4-openssl-dev` | ✓ |

### Shared across all cells

- `common_ldflags: '-s -Wl,--as-needed'`
- `sbfspot_extra: '-Wl,-Bstatic -lbluetooth -Wl,-Bdynamic'`
- `daemon_extra: ''` (empty)
- `HARDEN` (appended in step 9):
  `-pie -Wl,-z,relro -Wl,-z,now -Wl,-z,noexecstack -Wl,--build-id=sha1`

### Per-cell `prefix` for `-fmacro-prefix-map`

The prefix-map (see §4) differs per cell because upstream's cross-
toolchain uses a different sysroot path per codename. These
strings were extracted from the `.rodata` of upstream's binaries
in Phase 3.4 by grep'ing for `d:\rpi\cross\`:

| codename × arch | prefix |
|---|---|
| arm × buster | `d:\rpi\cross\buster\gcc8.3.0\arm-linux-gnueabihf\sysroot\usr\include` |
| arm × bullseye | `d:\rpi\cross\bullseye\gcc10.2.1\arm-linux-gnueabihf\sysroot\usr\include` |
| arm × bookworm | `d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include` |
| arm64 × bullseye | `d:\rpi\cross\bullseye64\gcc10.2.1\aarch64-linux-gnu\sysroot\usr\include` |
| arm64 × bookworm | `d:\rpi\cross\bookworm64\gcc12.2.0\aarch64-linux-gnu\sysroot\usr\include` |

Note the `rpi1` rpi-custom-toolchain suffix is missing on our
chroots (stock Debian/Raspbian `.comment` says `Raspbian 12.2.0-14+rpi1`
but we build via distro gcc directly without the `+rpi1` in our
version string). That's fine for functionality; it's one of the
documented Phase 3.5 residuals.

### `asset_name` + `asset_sha256`

Each cell knows the filename + SHA256 of its upstream tarball so
step 11 can download and verify it before the compare. These are
the exact values from the V3.9.12 release page and shouldn't
change unless upstream re-cuts the release.

Full list in the YAML; not repeated here for brevity.

## 8. Things we tried and dropped

If you find yourself thinking "why don't we just…", check here
first. Each entry links to the run that proved it wrong or the
phase doc with the full analysis.

### Dynamic libbluetooth everywhere (5e.1b)

**Idea:** drop `-Wl,-Bstatic -lbluetooth -Wl,-Bdynamic`, let
`libbluetooth.so.3` be a dynamic dep.

**Result:** works, no TEXTREL, no hardening gap. But
`libbluetooth.so.3` appears in NEEDED. Users on Raspberry Pi OS
Lite without `bluez` installed can't run the binary. Sbfspot-config
doesn't install bluez. Functional regression vs upstream. Rejected
in favour of the PIC rebuild.

**Run:** `24806180451` (all 15 cells green, but NEEDED regression).

### Non-PIE on armhf to match upstream (option A of the PIE decision)

**Idea:** drop `-fPIE`/`-pie` on armhf cells, since upstream's armhf
binaries are non-PIE anyway. No TEXTREL, no PIC rebuild needed, true
drop-in.

**Why rejected:** PIE is the biggest single hardening win in modern
Linux; dropping it to avoid a build-time workaround felt backwards.
PIE has zero user-visible behaviour change — only a security-posture
change. We decided modernisation beats byte-for-byte upstream match.

**Reasoning captured in conversation preceding 5e.1c commit.**

### Accept TEXTREL on armhf (option C)

**Idea:** keep PIE everywhere, keep static non-PIC libbluetooth,
accept that armhf binaries have `FLAGS: TEXTREL BIND_NOW`.

**Why rejected:** `checksec` and every other hardening auditor
would flag it. Defeats the point of going PIE.

### Silent install of SysGCC for upstream's toolchain (Phase 5d)

**Idea:** upstream uses some Windows cross-toolchain (probably
SysGCC Raspberry). If we can extract their `libstdc++.a` and use
it, the arm-bookworm residual closes.

**What we did:** ran the SysGCC installer under Wine + Xvfb +
xdotool GUI automation in CI, extracted its `libstdc++.a`.

**Result:** SysGCC r2's `libstdc++.a` is byte-identical to Raspbian's
shipped version. Substituting it was a no-op. SysGCC r1 is not
publicly reachable. Upstream's specific build of gcc remains private.

**Phase retired.** See [`../docs/phase5d-baseline.md`](../docs/phase5d-baseline.md).

### Three libstdc++ rebuild flags (Phase 4c)

**Idea:** maybe the arm-bookworm residual is just a specific gcc
rebuild flag of libstdc++ we haven't tried. Probed three candidates.

**Results:**
- `-fasynchronous-unwind-tables`: closes 4 KB but grows the wrong
  section (`.ARM.exidx` not `.ARM.extab`). Doesn't match.
- `-D_GLIBCXX_ASSERTIONS` on libstdc++: overshoots by +24 KB.
- `-fnon-call-exceptions`: overshoots by +61 KB.

**Conclusion:** no single well-known gcc flag closes the residual.
The difference is baked into upstream's private Canadian-cross gcc
build. See [`../docs/phase4-baseline.md`](../docs/phase4-baseline.md)
addendum.

### Rootfs caching via `actions/cache`

**Idea:** each cell spends ~90 s on debootstrap. If we cache the
rootfs tar across runs, we could cut matrix time ~20 %.

**Why not yet:** we prioritised correctness + hardening over speed.
Caching is deferred. The risk is subtle — a cached rootfs can go
stale against the live Raspbian archive and introduce non-obvious
package-version drift. We'd need a deliberate cache-invalidation
strategy. See [§9](#9-reproducibility-guarantees) for why the
drift matters.

### `-flto`, `-march=native`, `-O3`

See §4 "Flags we don't apply." Short version: no meaningful gain,
risk of breakage, not worth it.

## 9. Reproducibility guarantees

> "Given the same source, the same workflow, and the same input
> libraries, rebuilding produces byte-identical tarballs."

That's the goal. Here's what we do to achieve it, and what's
currently proven vs. not.

### What determinism means here

The word gets overloaded. Three levels:

1. **Run-to-run reproducibility, same month** — running the
   workflow twice in the same day/week produces identical output
   tarballs.
2. **Per-tag reproducibility, long-term** — running the workflow a
   year from now against the same V3.9.x tag produces identical
   output.
3. **Bit-for-bit with upstream's tarballs** — we gave up on this in
   Phase 5 (see [§8](#8-things-we-tried-and-dropped)). Upstream's
   Windows-7-Zip gzip writer uses a different gzip-header encoding
   than Linux gzip; matching requires reimplementing their writer.
   Not worth it for zero user-visible gain.

Level 1 is **proven** (measurement below). Level 2 is partial — one
real risk documented below. Level 3 is not a goal.

### What we do

**Fixed `SOURCE_DATE_EPOCH`.** Step 5 sets it to the tag commit's
Unix timestamp (`1739908502` for V3.9.12 = 2025-02-18 19:55:02 UTC).
Every downstream tool respects it:

| Tool | Effect |
|---|---|
| `gcc` | `__DATE__` / `__TIME__` macros use SDE (SBFspot doesn't currently use them, but if it ever does, reproducible). |
| `ld` | `--build-id=sha1` computes a SHA1 over content, not time, so the build-id is deterministic. |
| `ar` | Archive member timestamps → 0 (binutils ≥ 2.35). |
| `objcopy` | Section mtime fields → 0. |
| `tar` | `--mtime=@${SDE}` writes SDE into every member header. |
| `gzip` | `gzip -n` suppresses its own mtime + filename fields. |

**Deterministic tar format.** `--sort=name` + `--owner=0 --group=0
--numeric-owner` + `--format=ustar`. Member order and per-member
metadata are fully determined by the filename list and SDE.

**Pinned Debian snapshot for arm64.**
`snapshot.debian.org/archive/debian/20250222T000000Z` — that URL
returns exactly the same package bytes forever. Our arm64 binaries
are therefore fully locked down against distro drift.

**No `git`-state noise.** The workflow always checks out the tag,
never "master" or "HEAD." Two runs a year apart both get the same
tag commit → same source tree byte-for-byte.

### Measurement: run 1 vs. run 2

Phase 5e.2 commit produced run `24840329834`. Re-triggering the
workflow produced run `24840705192`, 7 minutes later, no source
changes. All 15 tarball SHA256s:

```
98a92a292f30a0471b25ef38699587192b5979b863dc6edf0f8fb5e182940bd4  sbfspot-mariadb-arm-linux-bookworm.tar.gz
5f49326c0ba4869d8b9a4839e60b12cabe4c5f2e242da801cf8e2ff841ae9862  sbfspot-mariadb-arm-linux-bullseye.tar.gz
16c5d9f225ec1d2e5761141f1b2c956ffc94ff348c1d063ff200258d9c04d502  sbfspot-mariadb-arm-linux-buster.tar.gz
eb1f923e0dede227d8abb8818ccb8c2c88ef164fb48496b4e5f59c2ff81fdf42  sbfspot-mariadb-arm64-linux-bookworm.tar.gz
f2cd2ec7f62389d0ce2b247a33fb817a7bbeb22b1338a2fd6cab063faefb247d  sbfspot-mariadb-arm64-linux-bullseye.tar.gz
f75f946a630d882391272d23d3e1cf6b1ea70c036ac03ac1a513a552a8ddcb22  sbfspot-nosql-arm-linux-bookworm.tar.gz
4ae7b5219b8aca2ef4c101d35290ce44e66e442fa1a42e1275f7d44b4e755953  sbfspot-nosql-arm-linux-bullseye.tar.gz
aa31e7b009b5c8b40c54c852c4236ad4a3318bb423c695e21420e4b298b38de9  sbfspot-nosql-arm-linux-buster.tar.gz
6305e012a79ec6743b318e1be23e0fbf43a285826997df61c387e2d6bac18910  sbfspot-nosql-arm64-linux-bookworm.tar.gz
5760a32cae1b86bce8b3dc6809dee9e820b8192c6722045e116e9b3ee5d0e920  sbfspot-nosql-arm64-linux-bullseye.tar.gz
0bd3b5b28b3bc6ff4538ca87197721a8103112c5473bc4e998c4e1de47d3c62e  sbfspot-sqlite-arm-linux-bookworm.tar.gz
1fb5dcf266ae4504b04372c3143fb7fb873b7a198d1d682b02b22dea8280082b  sbfspot-sqlite-arm-linux-bullseye.tar.gz
bc9c6a52e892819be7d1026ddaa9b9aa598f43070f320589766733fe6f03e2ba  sbfspot-sqlite-arm-linux-buster.tar.gz
7d553deb73eb92b65d07a3792d4053598bc5e31fa61b3fe45c39e9c9bd6d8b3f  sbfspot-sqlite-arm64-linux-bookworm.tar.gz
00731f90173b8a8b0afaa971a6ef3d9686d8452aa27122b851f4032911cbfb02  sbfspot-sqlite-arm64-linux-bullseye.tar.gz
```

**Diff between run 1 and run 2: empty.** All 15 tarballs
byte-identical.

### Known limitation: live Raspbian archive

The 9 arm cells (× {buster, bullseye, bookworm} × {sqlite, nosql,
mariadb}) use the **live** Raspbian archive — not a snapshot,
because Raspbian doesn't run a functional snapshot service (Phase 0
finding). That means:

- If Raspbian pushes a security update to `libc6`, `libgcc-s1`,
  `libstdc++6`, `libboost-date-time1.*`, `libsqlite3-0`,
  `libmariadb3`, or `libcurl4` between run N and run N+1, those
  libraries shift in our chroot and the compiled binaries in the
  affected arm cells change.
- Between runs minutes to days apart (as with our test above),
  this almost never fires — the archive is stable.
- Between runs weeks or months apart, it *will* fire eventually.

Mitigations we *could* apply later:
- Bake each codename's rootfs as a Docker image on GHCR once,
  re-use it per matrix cell. Deterministic across time, but adds
  an image-build workflow + GHCR maintenance.
- Pin specific package versions via `apt-get install pkg=x.y.z`
  per cell. Fragile (apt refuses to downgrade silently sometimes).

For now we document the limitation rather than engineer around it.
Level-1 reproducibility (same day) is the binding guarantee; level-2
(long-term) holds *modulo Raspbian security updates*.

### What we intentionally don't try to reproduce

- **Upstream's tar-layer byte-match.** Their Windows 7-Zip-authored
  gzip header has `os=00` (FAT), `xfl=04` (`--fast`), and a
  populated wall-clock `mtime`. Ours has `os=03` (Unix), `xfl=00`,
  `mtime=0`. Reproducing theirs requires a custom gzip writer; no
  user-visible benefit.
- **Upstream's per-file mtimes.** They preserve filesystem mtimes
  from the maintainer's local checkout (2021–2024 range per file).
  Ours are all `SOURCE_DATE_EPOCH` = 2025-02-18. Equally
  reproducible, more idiomatic for "this is release V3.9.12."
- **Upstream's permission bits.** They use `0777` on everything.
  Ours is 0755 / 0644. Security-sane Unix convention.

### How to verify locally

```sh
gh run download --repo kduvekot/SBFspot <run-id> -p 'phase5e-*'
find phase5e-* -name '*.tar.gz' -exec sha256sum {} +
```

Sort and diff against a previous run's output. Empty diff → clean
reproducibility.

## 10. Glossary

Plain-language definitions of terms used throughout this doc.

**ABI** — Application Binary Interface. The compiled-code contract
between a binary and its libraries: calling conventions, struct
layouts, symbol mangling. Two binaries with compatible ABIs can
share libraries at runtime; incompatible ABIs crash.

**`.a` file (static archive)** — a bundle of `.o` object files
produced by `ar rcs`. At link time, the linker extracts needed
objects from the `.a` and copies their code + data into the output
binary. Once linked, the `.a` is not needed at runtime. Contrast
with `.so`.

**aarch64 / arm64** — 64-bit ARM architecture. Used by modern Pis
(Pi 4/5 with 64-bit OS), most ARM servers, Apple silicon Macs.

**armhf** — "ARM hard-float," the 32-bit ARM EABI with hardware
floating-point calling convention. Used by Pi OS 32-bit. Can be
executed natively on aarch64 silicon via AArch32 mode.

**ASLR** — Address Space Layout Randomization. Kernel feature:
every time a PIE binary starts, its code and data are mapped at a
different random base address, so attackers can't hard-code
addresses to jump to.

**binutils** — GNU tools for manipulating binaries: `ld`, `ar`,
`objcopy`, `readelf`, `strip`, `nm`, etc. We use several.

**BIND_NOW** — ELF dynamic flag requesting "resolve all symbols
at startup, not lazily." Prerequisite for full RELRO.

**bluez** — Linux Bluetooth stack. Ships `libbluetooth.so.3` +
`libbluetooth.a` via the `libbluetooth-dev` package; also the
`bluetoothd` daemon. SBFspot uses libbluetooth for talking to
older Bluetooth-enabled SMA inverters.

**bluez source package** — the Debian source package named `bluez`
that produces `libbluetooth3`, `libbluetooth-dev`, `bluez`, and
several other binary packages. Fetched via `apt-get source bluez`.

**chroot** — "change root." A process's view of the filesystem is
restricted to a subtree starting at a specified directory. Used here
to pretend to be a different Debian version than the host runner is.

**`.comment` section** — an ELF section containing a string like
`GCC: (Debian 12.2.0-14) 12.2.0` identifying which compiler built
the binary. Normalised away during binary compare.

**cross-compile** — compiling on one architecture for execution on
another. Not what we do — we build natively for armhf on aarch64
silicon, which is the same instruction-set family. Upstream
cross-compiles from Windows x86_64 to armhf / aarch64, which is a
different setup.

**debootstrap** — a tool that downloads + unpacks a minimal set of
Debian/Raspbian packages into a target directory, producing a
working chroot for that codename/architecture.

**deb-src** — apt source-package repository entry. Needed for
`apt-get source <pkg>`. Not included in debootstrap's default
`sources.list`; we add it in step 8.

**ELF** — Executable and Linkable Format. The binary file format
Linux uses for executables, shared libraries, and object files.

**ET_DYN vs ET_EXEC** — ELF file types. `ET_EXEC` is a
fixed-address executable. `ET_DYN` is a shared object; when
loadable-as-executable, it's a PIE.

**GOT** — Global Offset Table. A table of pointers inside the
binary; PIC code accesses globals indirectly through the GOT so
the pointers can be patched at load time for a randomized address.
Target of RELRO hardening.

**LIBS (Makefile variable)** — SBFspot's Makefile declares
`LIBS := pthread bluetooth boost_date_time boost_system` plus the
DB-specific ones. At link time the Makefile emits
`$(addprefix -l,$(LIBS))` → `-lpthread -lbluetooth -lboost_date_time …`.

**LDFLAGS** — conventional Make variable for linker flags passed
to `ld` (via the compiler driver). We override it per-cell from
the workflow.

**NEEDED** — ELF dynamic-section entry (`DT_NEEDED`) listing a
shared library the binary requires at runtime. `readelf -d | grep
NEEDED` shows these. If `libX.so.N` is in NEEDED, the loader will
refuse to start the binary without `libX.so.N` installed.

**Non-PIC vs PIC code** — see §6. Non-PIC uses absolute addresses;
PIC goes through the GOT. Non-PIC static archives don't work well
inside PIE executables.

**PIE** — Position-Independent Executable. Links `-pie`, needs
`-fPIE` compile. Enables ASLR at runtime. Debian hardening default.

**relocation** — an instruction in an ELF file that says "at load
time, patch this address using information about symbol X." The
linker decides at link time which relocations to emit; the loader
processes them at startup. Text relocations (inside the `.text`
section) are the problematic kind that RELRO can't handle.

**RELRO** — "RELocation Read-Only." Linker flag `-z,relro`:
segments containing relocations get marked read-only after load-time
patching is done. Paired with BIND_NOW for "full RELRO," which
eliminates writable relocations entirely. Blocks a class of
exploits that overwrite the GOT.

**root / rootfs** — the top-level directory of a Unix filesystem
hierarchy. Inside a chroot, the chroot's directory *is* the root.

**runner** — a machine (VM, in GitHub's case) that executes a
GitHub Actions job.

**sbfspot-config** — upstream's installer script at
`SBFspot/sbfspot-config`. Interactive Bash + whiptail UI that walks
a user through installing a SBFspot tarball on their Raspberry Pi.
Not touched by this workflow, but relevant because it defines what
runtime deps exist on a fresh install.

**.so file (shared object)** — a library meant for runtime dynamic
linking. `libfoo.so.N` is the runtime file; `libfoo.so` is usually
a symlink to it for the linker to find. Contrast with `.a`.

**sources.list** — apt's configuration file listing package
repositories. Inside our chroots, debootstrap writes this for us.

**sysroot** — the directory tree that stands in as `/` for a
cross-compiler. Contains headers and libraries for the target
architecture. Upstream's Windows toolchain has its sysroot at
`d:\rpi\cross\<codename>\gcc<ver>\<triplet>\sysroot\`.

**TEXTREL** — DT_TEXTREL dynamic flag. Indicates the binary has
relocations inside its `.text` (code) segment. Bad for security
(requires writable code segment at load time) and incompatible
with full RELRO. Emitted by the linker when static-linking non-PIC
code into a PIE executable on armhf.

**triplet** — a GNU-style identifier for a CPU + OS + ABI
combination, e.g. `arm-linux-gnueabihf`, `aarch64-linux-gnu`,
`x86_64-w64-mingw32`. Used in library paths like
`/usr/lib/arm-linux-gnueabihf/`.

**Ubuntu-24.04-arm** — GitHub Actions runner image label. Points
to a 4-core Ampere Altra (aarch64) VM running Ubuntu 24.04.

---

*Doc ends. The YAML is the source of truth for exact flag values +
step order; this doc is the source of truth for the reasoning
behind each choice.*
