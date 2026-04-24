# `linux-release.yml` — how SBFspot's Linux tarballs are built

Companion doc for
[`linux-release.yml`](linux-release.yml). Written so a reader who has
never touched GitHub Actions, never cross-compiled, and doesn't know
what PIE or RELRO stand for can still follow along. If you already
know those terms, skip ahead — headings are dense enough to navigate.

**What this pipeline is.** An auditable, reproducible CI replacement
for the 15 Linux tarballs that have historically been hand-built on
the maintainer's Windows machine. On every tag push (`V3.9.*`), the
workflow produces 45 artefacts — 15 main tarballs, 15 DWARF-debug
sidecars, and 15 package SBOMs — and attaches them to the matching
GitHub Release. On
master pushes and pull requests it builds the same set for
validation without publishing.

**What it deliberately does not do.** It does not match upstream's
earlier hand-built binaries byte-for-byte. On `arm × bookworm`
cells specifically, upstream's cross-toolchain produced a
`libstdc++` whose object bytes differ from Debian/Raspbian's shipped
`libstdc++` (~16 KB of code-gen drift, characterised by the prior
investigation but not reproducible without access to the
maintainer's toolchain). Separately, the historical tarballs' gzip
headers are Windows-flavoured — `OS=0x00` (FAT filesystem byte)
and `XFL=0x04` (`gzip --fast`) — which Linux `gzip -n` cannot
produce regardless of input. Both deltas are cosmetic: they do
not affect what the binary does when it runs. This pipeline trades
byte-parity-with-the-past for modern hardening flags (PIE + full
RELRO + BIND_NOW + `_FORTIFY_SOURCE=2` + `-fstack-protector-strong`
+ `-D_GLIBCXX_ASSERTIONS` + deterministic `--build-id=sha1`), a
clean runtime-dependency surface, and long-term reproducibility
anchored to the git tag (see [§9](#9-reproducibility-guarantees)
for the exact guarantee and its caveats).

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

## 1. Overview

### Release shape

SBFspot historically publishes **15 Linux release tarballs** per
version:

- 3 database variants — `sqlite`, `nosql`, `mariadb`
- 3 Debian/Raspbian codenames — `buster`, `bullseye`, `bookworm`
- 2 architectures — `arm` (32-bit armhf, Raspberry Pi) and
  `arm64` (aarch64, modern Pi / generic ARM servers)

…minus `buster × arm64` (never published). So
3 × 3 × 2 − (1 × 1 × 3) = **15 tarballs.**

Plus two Windows zips. Those are not in scope for this workflow; a
sibling `windows-release.yml` would be the natural home.

### Why a CI pipeline

Two properties the hand-built process didn't give us:

- **Auditable.** Every flag, every step, every library version
  installed in each chroot is visible in the workflow file, the
  run log, and the per-cell SBOM. A third-party reviewer can
  follow the provenance from git tag → workflow → toolchain +
  libraries → output bytes.
- **Reproducible.** For a given git tag, the same workflow
  produces byte-identical tarballs when re-run minutes or days
  apart (proven by construction and measured; see §9). Long-term
  reproducibility is arch-dependent: arm64 cells pin to
  `snapshot.debian.org` and stay stable indefinitely; arm cells
  pull from a live Raspbian archive and will shift as Raspbian
  pushes security updates, with the per-cell SBOM capturing
  exactly what was installed at build time.

### What it produces per cell

- `sbfspot-<db>-<arch>-linux-<codename>.tar.gz` — the main tarball.
  Same filename and internal structure upstream has always used;
  stripped binaries + unchanged config/SQL/TagList files.
- `sbfspot-<db>-<arch>-linux-<codename>.debug.tar.gz` — a DWARF
  debug sidecar tarball. Lets gdb / `debuginfod` symbolicate crash
  dumps without bloating the main tarball. New in this pipeline;
  not present in historical releases.

## 2. The shape of a run

### Runners

A **runner** in GitHub Actions is just a Linux VM that GitHub starts
up for the duration of a job. When we write:

```yaml
runs-on: ubuntu-24.04-arm
```

we're saying "give me a freshly-booted Ubuntu 24.04 VM whose CPU is
**aarch64** (64-bit ARM)." This is a real aarch64 VM — no emulation
layer. That matters because:

- 64-bit ARM cores that support the AArch32 execution mode can
  run 32-bit ARM (armhf) binaries directly. The runners GitHub
  provides on this label do support AArch32, which the pipeline
  demonstrates empirically on every run: step 12 executes the
  freshly-compiled armhf binary and it prints its version banner
  without QEMU involvement. So a single runner builds *and*
  smoke-tests both arm64 and armhf cells.

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

| Mirror | For which cells | Notes |
|---|---|---|
| `http://archive.raspbian.org/raspbian` | arm × {bullseye, bookworm} | Current Raspbian archive. armhf binaries ship with `.comment` carrying `GCC: (Raspbian X+rpi1)` — verify with `readelf -p .comment <binary>` on any upstream arm tarball. |
| `http://legacy.raspbian.org/raspbian` | arm × buster | Raspbian's legacy archive for `oldoldstable` releases. `archive.raspbian.org` has removed buster's package indexes (its `dists/buster/Release` is 404); `legacy.raspbian.org` continues to serve buster and receives occasional maintenance updates. |
| `http://snapshot.debian.org/archive/debian/<YYYYMMDD>T000000Z` | arm64 × all | Debian's snapshot service, pinned to the UTC-day boundary of the triggering commit's timestamp. Each tagged release builds against a coherent Debian snapshot from its own day. |

The arm64 snapshot pin is **derived at run time** (step 3) and
applied by debootstrap (step 5), not hardcoded. A V3.9.12 build
resolves to `20250218T000000Z`; a V3.9.13 build a few months
later will automatically pick the snapshot for that commit's day.
No per-release maintenance.

Raspbian doesn't run a functional snapshot service (the
`snapshot.raspbian.org/archive/…` paths return 404). All 9 arm
cells therefore use whichever Raspbian mirror is current at build
time, including the legacy buster mirror, which is not a
strictly-frozen snapshot either. The per-cell SBOM (step 14)
records exactly which package versions were installed, so build-
time state remains auditable even if a mirror has moved on.

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

Wall-clock per full matrix run: ~4–6 minutes (observed during the
development reference rebuild — the slowest cells spent ~90 s on
debootstrap + ~60 s on the build). Every cell does its own
debootstrap + compile + link + smoke-test independently, so the
total time is gated by the slowest cell, not the sum.

### Triggering the workflow

Four triggers, each with a clear purpose:

```yaml
on:
  push:
    branches: [master]          # continuous CI on master, artefacts only
    tags: ['V3.9.*']            # release builds, attach to GitHub Release
  pull_request:
    branches: [master]          # PR validation, artefacts only
  workflow_dispatch:            # manual run, artefacts only
```

Release-attachment logic lives in a single conditional step near
the end of the workflow:

```yaml
- name: Attach tarballs + SBOMs to GitHub Release (tag push only)
  if: startsWith(github.ref, 'refs/tags/V3.9.')
  uses: softprops/action-gh-release@v2
  with:
    files: |
      out/tarballs/*.tar.gz
      out/tarballs/*.packages.list
```

So a V3.9.13 tag push causes the workflow to run *and* automatically
upload its 45 artefacts (15 main tarballs + 15 debug sidecars + 15
package SBOMs) to the V3.9.13 Release page. Every other trigger
produces the same artefacts, but they stay as workflow-run artefacts
(90-day retention, not publicly linked).

## 3. Step-by-step walkthrough

Every cell runs the same 17 steps. Here's what each one does, in
order, in plain language. Flag rationale is in sections 4–6; this
section is just "what happens when."

### Step 1: `Install host-side tooling`

Runs on the outer runner (not the chroot). Installs the host-side
tooling needed to build the chroot and inspect output binaries:

- `debootstrap` — creates the chroot.
- `binutils`, `file` — `readelf` / `objcopy` / `strip` for the
  post-build inspection steps, plus `file` for quick type checks.
- `curl`, `jq` — fetching the Debian archive keyring (and,
  historically, downloading release assets for comparison).
- `debian-archive-keyring` — signing keys debootstrap uses to
  verify `snapshot.debian.org` and live Debian mirrors.

### Step 2: `Check out source`

Uses `actions/checkout@v6` to clone the repository at whatever ref
triggered the run (a V3.9.* tag, a master commit, or a PR HEAD)
into `./src/`. For a tag push the tree is the tagged snapshot; for
other triggers it's the current branch state.

### Step 3: `Derive SOURCE_DATE_EPOCH + snapshot date from triggering commit`

Reads the commit timestamp of whatever commit triggered the run
(`git -C src log -1 --pretty=%ct`) and writes two values to
`$GITHUB_ENV`:

- `SOURCE_DATE_EPOCH` — the Unix timestamp verbatim.
- `DEBIAN_SNAPSHOT_DATE` — the same timestamp rounded to the UTC
  day boundary, formatted `YYYYMMDDT000000Z`. Used by the next
  step to pin the arm64 Debian mirror to `snapshot.debian.org`.

`SOURCE_DATE_EPOCH` is a
[reproducible-builds.org convention](https://reproducible-builds.org/docs/source-date-epoch/):
a single Unix timestamp that build tools substitute for "now"
wherever they'd otherwise embed a build-time date. How it's used
in this pipeline:

- **gcc** uses it for the `__DATE__` / `__TIME__` macros
  (SBFspot's source doesn't use those today; safe for future use).
- **tar** reads it via our explicit
  `--mtime=@$SOURCE_DATE_EPOCH`, writing that timestamp into every
  archive member header.
- **ar** and **objcopy** (binutils ≥ 2.35, released 2020-07)
  zero out member timestamps / section mtime fields when
  `SOURCE_DATE_EPOCH` is set in the environment.
- **gzip** is handled separately: `gzip -n` suppresses the
  filename + mtime fields in the gzip header regardless of
  `SOURCE_DATE_EPOCH`. The env var + `-n` together give us a
  gzip-header that's constant per-tag.
- **ld**'s `--build-id=sha1` is content-hashed (SHA1 over the
  loadable segments), not time-based, so it's naturally
  deterministic given deterministic input — `SOURCE_DATE_EPOCH`
  doesn't affect it directly.

### Step 4: `Fetch Raspbian keyring` (conditional)

Runs only for arm cells (which use Raspbian mirrors). Debootstrap
won't trust a Raspbian mirror unless we give it Raspbian's signing
key. Downloads `raspbian.public.key`, dearmors it into a binary
keyring file that debootstrap accepts.

### Step 5: `Debootstrap rootfs`

This is where the chroot gets built. Picks the right mirror URL +
keyring based on `matrix.cell.mirror` (`raspbian` / `raspbian-legacy`
/ `debian`) and runs debootstrap. When it finishes, `./rootfs/` is a
minimal bookworm-armhf (or whatever the cell is) install.

For the `debian` mirror (arm64 cells), the URL substitutes
`$DEBIAN_SNAPSHOT_DATE` from step 3 — so the build pulls from the
`snapshot.debian.org` archive of the triggering commit's UTC day,
not a hardcoded date.

This step takes ~60–90 s — the bulk of each cell's runtime.

### Step 6: `Stage source tree inside rootfs`

`cp -a src/SBFspot rootfs/src/` and the same for
`src/SBFspotUploadCommon` (and `src/SBFspotUploadDaemon` if the
cell has a daemon — `nosql` doesn't). After this, the source is
inside the chroot, ready to compile with the chroot's g++.

### Step 7: `Install hardened compiler wrapper`

This one's sneaky. We `sudo tee rootfs/usr/local/bin/g++` to
create a small shell script that *intercepts* calls to `g++`
inside the chroot. The real compiler lives at `/usr/bin/g++`; our
wrapper at `/usr/local/bin/g++` is earlier on `$PATH` (sudo's
default `secure_path` places `/usr/local/bin` before `/usr/bin`),
so `make` resolves the unqualified `g++` to the wrapper, which
then `exec`s the real compiler with our hardening flags prepended:

```sh
#!/bin/sh
exec /usr/bin/g++ \
  '-fmacro-prefix-map=/src=/build/sbfspot' \
  -D_FORTIFY_SOURCE=2 \
  -fstack-protector-strong \
  -D_GLIBCXX_ASSERTIONS \
  -fPIE \
  -g \
  "$@"
```

Flag-ordering note: our flags come **before** the original
arguments (`"$@"`). gcc's rule for most conflicting flags is that
the *later* occurrence wins, so if the Makefile ever passed a
contradicting flag (e.g. `-fno-stack-protector`) it would
override ours. The V3.9.x Makefile does not, so the hardening
flags stay effective — but this is worth knowing if the Makefile
is ever extended.

The `-g` at the end makes gcc emit DWARF debug info; step 11
splits that into a sidecar. Main binary stays small; auditors
get symbolication data on demand.

Why a wrapper instead of editing the Makefile? The goal is to
add hardening without touching tracked source. The wrapper is a
clean side-channel: the Makefile and `.cpp` files stay
byte-identical to what upstream ships, but every `g++` invocation
transparently picks up our flags. Easier to review, easier to
remove if the project later bakes the flags into the Makefile
directly.

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

### Step 11: `Split debug info into .debug sidecars`

The binary coming out of step 10 is unstripped: `-g` in the wrapper
added DWARF debug info, and the LDFLAGS dropped `-s` (so the linker
didn't strip). This step does the split:

```sh
objcopy --only-keep-debug   SBFspot SBFspot.debug   # extract DWARF
objcopy --strip-unneeded    SBFspot                 # strip main
objcopy --add-gnu-debuglink=SBFspot.debug SBFspot   # add wiring
```

After this:

- `SBFspot` is stripped (~400–550 KB depending on cell — measured
  419 KB/454 KB/461 KB/505 KB/526 KB across a sampling of the 15
  cells during the reference rebuild), runnable, and has a new
  small `.gnu_debuglink` ELF section pointing gdb / `debuginfod`
  at the sidecar.
- `SBFspot.debug` has only the DWARF content (~5–7 MB uncompressed;
  ~2–3 MB after the sidecar tarball is gzipped), not runnable,
  usable for symbolication.

The stripped binary is what goes into the main tarball. The
sidecars go into a companion `.debug.tar.gz` (step 15). If a user
ever core-dumps in production, an auditor can line up the two and
get full source-line backtraces. See
[§9](#9-reproducibility-guarantees) for the reproducibility
measurement that shows both tarballs are deterministic.

Same dance runs for `SBFspotUploadDaemon` on cells where
`has_daemon: true`.

### Step 12: `Smoke test — binary runs and reports version`

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
took effect. Smoke-testing the stripped binary (post step 11)
confirms that the debug-split didn't accidentally break dynamic
loading.

### Step 13: `Copy binaries out of the chroot`

`sudo cp` the built (and now stripped) binary and — where
applicable — the daemon out of `rootfs/src/<target>/bin/` into
`out/bin/`, and `chown` to the runner user. The `.debug` sidecars
from step 11 stay inside the chroot; they're pulled in directly by
the assembly step.

### Step 14: `Capture SBOM (installed packages + .deb SHA256s)`

Writes a per-cell bill of materials to
`out/tarballs/sbfspot-<db>-<arch>-linux-<codename>.packages.list`.
Format is one line per package:

```
<package>\t<version>\t<architecture>\t<sha256-of-.deb>
```

Two sources of package names:

1. **Runtime-linked libraries.** For each `NEEDED` entry in the
   built binaries (`readelf -d`), resolve the soname to a filesystem
   path inside the chroot via `ldconfig -p`, then ask
   `dpkg -S <path>` which package owns it. Union across SBFspot and
   SBFspotUploadDaemon (where applicable).
2. **Explicitly-installed build tooling:** `g++`, `make`,
   `binutils`, `dpkg-dev`, `libbluetooth-dev`,
   `libboost-date-time-dev`, `libboost-system-dev`,
   `bluez` (source for the PIC libbluetooth rebuild), plus the
   per-cell `db_pkgs` (sqlite/mariadb dev) and `curl_pkg`.

Then for each package in the union:
- `dpkg-query` for exact version and architecture.
- Locate the cached `.deb` in `/var/cache/apt/archives/` and
  compute SHA256.

Total ~15 lines per cell — not the ~100 packages `debootstrap`
pulls in wholesale, just the ones that actually affect the output.

**Why the .deb SHA256 matters.** For Debian-originated packages
the version string alone is enough to retrieve the exact `.deb`
(or the source) from `snapshot.debian.org` indefinitely. For
Raspbian-patched (`+rpi*`) packages, the live archive deletes
superseded versions, so the SHA256 is the only way to later
verify a recovered `.deb` (from a third-party mirror, a
user's local cache, a backup) is bit-identical to what the
build used.

### Step 15: `Assemble reproducible release tarball`

Takes the fresh binary (+ daemon where applicable) + the
non-binary data files from the tagged source tree, stages them in
a temporary directory with sensible modes (0755 binaries / 0644
data), and `tar`s the lot up deterministically. The binary is the
stripped one from step 11.

Tar flags chosen for reproducibility:
- `--sort=name` — member order is fixed (lexicographic).
- `--owner=0 --group=0 --numeric-owner` — no host-specific uid/gid.
- `--mtime=@${SOURCE_DATE_EPOCH}` — every member has the triggering commit's time.
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
`SBFspotUpload.default.cfg`. This matches the long-standing
filename convention in the V3.9.x tarballs. The `.default.cfg`
naming means that extracting the tarball directly over an
existing install won't clobber a user-edited `SBFspot.cfg` — the
sample config is carried as a separate, distinctly-named file.
(Note: `sbfspot-config`, the interactive installer, generates
its own `SBFspot.cfg` from shell templates and does not read the
`.default.cfg` from the tarball. The file is effectively a
reference / sample for manual installs.)

**Second tarball: debug sidecars.** The same step also assembles
`sbfspot-<db>-<arch>-linux-<codename>.debug.tar.gz`, containing
just the `.debug` files produced by step 11
(`SBFspot.debug` + optionally `SBFspotUploadDaemon.debug`). Same
reproducible tar+gzip treatment. Consumers who want symbolication
grab this alongside the main tarball; end users never need to.

### Step 16: `Upload per-cell artefacts`

Tars up `./out/` (which has `out/bin/`, `out/tarballs/`) as an
artefact named `sbfspot-<cell-id>`.
`actions/upload-artifact@v7` handles the upload. 90-day retention.

### Step 17: `Attach tarballs + SBOMs to GitHub Release` (tag push only)

Conditional on `github.ref` starting with `refs/tags/V3.9.` —
i.e. this step only runs when the workflow was triggered by a tag
push matching the release-tag pattern. Uses
`softprops/action-gh-release@v2` to attach
`out/tarballs/*.tar.gz` (main tarballs + debug sidecars) plus
`out/tarballs/*.packages.list` (SBOMs from step 14) as assets on
the matching GitHub Release. 45 files total: 15 main + 15 debug +
15 SBOM.

On master pushes, PRs, and manual dispatches, this step is a
no-op. The tarballs and SBOMs still exist as workflow-run
artefacts (step 16); they just don't get promoted to a public
Release.

That's the whole pipeline. The next three sections explain *why*
each compile/link/libbluetooth choice is what it is.

## 4. Why each compile flag

These are the flags applied by the `/usr/local/bin/g++` wrapper
(step 7). They're added to *every* C++ compilation in the build.

### `-fmacro-prefix-map=/src=/build/sbfspot`

**What it does:** tells the preprocessor "whenever you bake the
current source file's path into the binary via the `__FILE__` macro,
substitute the left side of the `=` with the right side."

**Why:** C and C++ code can embed its own file paths via things like
`assert()` macros (which expand to
`__assert_fail("x != NULL", "/src/SBFspot/main.cpp", …)`). Those
paths end up as string literals inside the `.rodata` section of the
compiled binary. Without normalisation, two builders with
different source checkout paths produce binaries that differ
byte-for-byte in their string tables for cosmetic reasons.

We substitute `/src` → `/build/sbfspot` so every cell produces a
binary whose `__FILE__` strings look like `/build/sbfspot/main.cpp`
regardless of where the source actually lived at build time. Clean,
descriptive, stable.

### `-D_FORTIFY_SOURCE=2`

**What it does:** tells glibc to replace certain standard-library
functions (`memcpy`, `strcpy`, `sprintf`, `read`, …) with
bounds-checking versions when the compiler can statically prove a
buffer size. At runtime, if a check fails, the program calls
`__chk_fail()` and aborts.

**Why:** catches buffer overflows that would otherwise corrupt
memory silently. `_FORTIFY_SOURCE=2` has been part of Debian's
default `CPPFLAGS` via `dpkg-buildflags` for many years (verifiable
with `DEB_VENDOR=Debian dpkg-buildflags --get CPPFLAGS`); every
Debian package built with the stock buildflags picks it up. We
match that policy here.

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
- `-fstack-protector` (only functions with ≥8-byte char buffers
  or `alloca()`) — too narrow; misses functions with small char
  buffers + local arrays.
- `-fstack-protector-all` — every function, including trivial
  ones that don't touch the stack in dangerous ways. Higher
  per-call overhead for proportionally-tiny incremental
  coverage vs `-strong`. Used by some security-heavy projects
  but not the mainstream distro default.
- `-fstack-protector-strong` — the sweet spot and what Debian
  (and Ubuntu, Fedora, RHEL) defaults to.

### `-D_GLIBCXX_ASSERTIONS`

**What it does:** turns on runtime debug-assertions inside libstdc++
(the GNU C++ standard library). Things like `std::vector::operator[]`
gain bounds checks; `std::list::front()` aborts on empty list;
iterator comparisons sanity-check.

**Why:** catches C++ standard-library misuse at runtime instead of
letting it corrupt memory.

**Distro policy note:** unlike the other hardening flags in this
section, `_GLIBCXX_ASSERTIONS` is **not** in Debian's default
`dpkg-buildflags` set (verifiable with
`DEB_VENDOR=Debian dpkg-buildflags --get CPPFLAGS` — it's absent).
It *is* the default in Fedora and RHEL's GCC packaging and is
recommended by Red Hat's
[Developer Program](https://developers.redhat.com/blog/2020/02/11/toward-_fortify_source-parity-between-clang-and-gcc)
and by the [OpenSSF compiler hardening
guide](https://best.openssf.org/Compiler-Hardening-Guides/). We
opt into it here as additional defense in depth.

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
- **`-march=native`:** Would query whatever the build machine's
  silicon supports and bake that into the binary. The Azure
  aarch64 runner's cores have ARMv8.2+ features (crypto
  extensions, dot-product, etc.) that older Raspberry Pis lack.
  Binaries tuned to the runner could execute instructions that
  illegal-instruction on a user's Pi. We let gcc use each
  chroot's distro-default `-march` (e.g. Raspbian's `armv6` for
  legacy-Pi compatibility) and stay out of the way.
- **`-fsanitize=address`** / `-fsanitize=undefined`: debugging tools,
  not production-shippable (they require runtime library support).

### Flags we only apply via the upstream Makefile

The Makefile adds `-c -Wall -O2 -Wno-unused-local-typedefs
-Wno-psabi`. Those stay. `-Wno-psabi` silences a noisy warning about
C++ ABI changes between gcc versions for armhf; it's cosmetic.

## 5. Why each link flag

LDFLAGS for each cell is built in step 10 of the workflow from
three pieces:

```
SBFSPOT_LDFLAGS = $common_ldflags $HARDEN $sbfspot_extra
```

where:

- `common_ldflags` = `-Wl,--as-needed` (same on every cell)
- `HARDEN` = `-pie -Wl,-z,relro -Wl,-z,now -Wl,-z,noexecstack -Wl,--build-id=sha1`
- `sbfspot_extra` = `-Wl,-Bstatic -lbluetooth -Wl,-Bdynamic` (same on every cell)
- `daemon_extra` = `''` (empty, same on every cell — daemon doesn't need bluez)

Notably absent: `-s`. Stripping happens explicitly via `objcopy
--strip-unneeded` in step 11 so we can extract DWARF into a
sidecar before stripping. Letting the linker strip at link time
would discard the debug info before we could save it.

### A note on the `-Wl,…` syntax

gcc passes most `-X` options to its own internal machinery. To pass
a flag through to the **linker** (`ld`), you prefix it with `-Wl,`.
E.g. `-Wl,-z,now` becomes `-z now` when ld sees it. That's why all
the linker-specific flags below look weird.

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

**Why:** enables "full RELRO" — together with `-z,relro`, means
the GOT is fully populated and read-only from startup onwards.
Lazy binding (the default) requires the GOT to stay writable for
the program's lifetime, which weakens RELRO.

**Distro policy note:** BIND_NOW is **not** enabled by default in
Debian's `dpkg-buildflags` — the hardening feature `bindnow`
reports `bindnow=no` in `dpkg-buildflags --status`, opt-in only
via `DEB_BUILD_MAINT_OPTIONS=hardening=+bindnow`. Most security
guidance (Debian hardening wiki, OpenSSF, Red Hat) recommends
turning it on explicitly for security-sensitive binaries, which
is what we do here. Binaries built with the stock Debian
`dpkg-buildflags` defaults would have only partial RELRO.

Slight startup-time cost (all symbols resolved up front). For a
long-lived program like SBFspot, this is a one-time cost dwarfed
by normal operation.

### `-Wl,-z,noexecstack`

**What:** marks the stack as non-executable via a PT_GNU_STACK ELF
segment with flags `RW` (not `RWE`).

**Why:** prevents attackers from jumping to injected shellcode on
the stack. Modern kernels already enforce NX stacks by default on
x86_64/aarch64, but the flag must be on the binary to be honoured.

**Fun history:** C code that uses GCC's
[nested-functions extension](https://gcc.gnu.org/onlinedocs/gcc/Nested-Functions.html)
triggers an executable-stack requirement — the compiler emits a
trampoline on the stack for the inner function's closure. Linking
any object compiled with nested functions flips `PT_GNU_STACK` to
`RWE`. SBFspot doesn't use nested functions, so the flag is safe.

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
upstream binaries have libbluetooth statically embedded —
`readelf -d <upstream-binary> | grep libbluetooth` returns nothing
on any of them (verifiable on the V3.9.12 release assets). This
is a *portability* property: it means users don't need the `bluez`
package installed on their target system. Raspberry Pi OS Lite
images are minimal and historically ship without `libbluetooth3`
pre-installed, and `sbfspot-config` (upstream's installer) does
not install it either (grep the installer script for `bluetooth`
or `libbluetooth` — zero matches against `install_pkg`). If our
binary dynamically linked libbluetooth, users on such a system
would hit a missing-library error at `exec` time.

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

So: **for a PIE binary to be clean (no text relocations), every
piece of code linked in must be PIC.** Non-PIC code *can* go in,
but the linker then has to patch absolute addresses at load time
by writing into the `.text` segment — which defeats the read-only-
code invariant that RELRO and PIE together try to enforce. That's
the flag we want to avoid.

### What breaks

Debian ships `libbluetooth.a` (static archive) in the
`libbluetooth-dev` package. On **armhf**, static-linking that stock
`.a` into a PIE binary produces text relocations; on **arm64** it
doesn't. An early iteration of this pipeline hit that asymmetry
empirically: `readelf -d` on every armhf cell showed
`FLAGS: TEXTREL BIND_NOW`, while arm64 cells showed clean
`FLAGS: BIND_NOW`.

What's happening underneath:

- **Text relocations.** When the linker can't resolve a symbol
  reference to a PC-relative or GOT-indirect form at link time, it
  emits a relocation entry in the dynamic section. At load time the
  loader applies it by *writing into the code segment* to patch the
  address. That sets the ELF `DT_TEXTREL` flag and temporarily
  makes `.text` writable, which defeats RELRO's guarantee. Modern
  hardening checkers (`checksec`, OpenSSF compiler-hardening
  guidance) flag it as a regression.
- **Why armhf hits it with stock libbluetooth.a.** The object files
  inside the stock armhf `.a` are compiled without `-fPIC`, so
  they reference globals via `R_ARM_ABS32` relocations. When those
  object files are linked into a PIE executable (which can be
  loaded at any address), the linker has nowhere to resolve those
  absolute references except by emitting TEXTREL. You can see the
  non-PIC relocation types directly:
  `readelf -r /usr/lib/arm-linux-gnueabihf/libbluetooth.a | head`.
- **Why arm64 doesn't hit it.** The aarch64 codegen generally
  produces PC-relative `adrp`+offset sequences for symbol access
  even under non-PIC. That means whatever relocations are in the
  stock aarch64 `.a` tend to resolve cleanly under PIE without
  text patching. (Whether the stock arm64 `.a` is itself compiled
  with `-fPIC` or just benefits from aarch64's PC-relative-by-
  default codegen is a Debian-packaging detail we don't lean on.)

For simplicity the PIC rebuild step runs on **every** cell
(armhf and arm64 alike) — it's required on armhf and harmless on
arm64. Keeping the step unconditional avoids one more per-arch
branch in the YAML; the ~30-second cost on arm64 cells is cheaper
than the maintenance overhead of a conditional.

### Why upstream's historical binaries didn't hit it

The hand-built armhf tarballs are non-PIE (`ELF … executable`,
not `pie executable`) — a default of the older cross-toolchain.
A non-PIE executable loads at a fixed address, so absolute
addresses inside a static non-PIC `.a` link in cleanly. No
TEXTREL, no warnings.

This pipeline emits PIE instead (modern hardening default), so
we need `libbluetooth.a` to be **PIC**.

### First attempt: dynamic-link libbluetooth

Simplest workaround: drop the static-link entirely, let
`libbluetooth.so.3` be a runtime dep.

Result: TEXTREL gone, but `libbluetooth.so.3` appeared in NEEDED.
Users on Pi OS Lite (no `bluez` installed) couldn't run the
binary at all. Real portability regression versus the historical
tarballs; rejected.

### What this pipeline does: PIC rebuild from distro source

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

If the `.a` is PIC, relocations of those types (indirect-through-
GOT) will be present. If non-PIC, they won't be. This check is at
the end of step 9 itself, so a regression in the rebuild surfaces
immediately rather than two steps later when the SBFspot link
would emit TEXTREL.

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

- `common_ldflags: '-Wl,--as-needed'`
- `sbfspot_extra: '-Wl,-Bstatic -lbluetooth -Wl,-Bdynamic'`
- `daemon_extra: ''` (empty)
- `HARDEN` (appended in step 10):
  `-pie -Wl,-z,relro -Wl,-z,now -Wl,-z,noexecstack -Wl,--build-id=sha1`
- `-fmacro-prefix-map=/src=/build/sbfspot` (in the g++ wrapper)

## 8. Things we tried and dropped

If you find yourself thinking "why don't we just…", check here
first. Each entry documents the reasoning so a future reviewer
doesn't redo the experiment cold.

### Dynamic-linked libbluetooth

**Idea:** drop `-Wl,-Bstatic -lbluetooth -Wl,-Bdynamic`, let
`libbluetooth.so.3` be a runtime dependency like the other
libraries.

**Result:** works, no TEXTREL, simpler pipeline. But
`libbluetooth.so.3` then appears in NEEDED. On Raspberry Pi OS Lite
and other minimal setups where `bluez` isn't pre-installed, users
couldn't `exec` the binary. `sbfspot-config` (the existing
installer) doesn't add `libbluetooth3` as a runtime dep. So this
would be a functional portability regression versus the historical
hand-built tarballs, which embed libbluetooth statically. Rejected
in favour of the PIC rebuild.

### Non-PIE on armhf

**Idea:** drop `-fPIE`/`-pie` on armhf cells. Upstream's historical
armhf binaries are non-PIE anyway (a default of their 2020-era
cross-toolchain). Non-PIE binaries tolerate static-linking non-PIC
code without emitting TEXTREL. No PIC rebuild needed.

**Why rejected:** PIE is the biggest single hardening win in
modern Linux — address-space layout randomisation. It's Debian's
`dpkg-buildflags` default (`pie=yes`) for all architectures since
buster; packages can opt out via
`DEB_BUILD_MAINT_OPTIONS=hardening=-pie`, but that's the exception.
Dropping PIE to sidestep a build-time issue felt backwards. PIE
has no user-visible behavioural effect — only a security-posture
improvement.

### Accept TEXTREL on armhf

**Idea:** keep PIE, keep static non-PIC libbluetooth, accept that
armhf binaries end up with `FLAGS: TEXTREL BIND_NOW`.

**Why rejected:** `checksec` and other hardening auditors flag
`TEXTREL` as a serious regression. The direct problem it causes
is that the dynamic linker has to write into the `.text` segment
at load time to patch addresses — either requiring the code
segment to be mapped writable (historically RWX) or a transient
`mprotect` RW→RX dance. That defeats the "code segment is never
writable after load" invariant that modern ELF hardening tries
to maintain. (It's a separate property from RELRO itself —
RELRO targets the GOT and initialised-data segments, not `.text`
— but both are part of the same "write-xor-execute" family of
protections.)

### Rootfs caching (`actions/cache` or GHCR)

**Idea:** each cell spends ~60–90 s on debootstrap. Caching the
unpacked rootfs across runs would cut each cell's wall-clock
time noticeably, with the slowest-cell-gated matrix finishing
faster overall.

**Why not:**
- **Release cadence is low.** Upstream tags a new release a few
  times a year, not daily. The aggregate CI-time saving is small.
- **Cache hits can mask security-update staleness.** A cache
  pinned for weeks or months keeps shipping the Raspbian package
  versions that existed when the cache was populated, even if
  Raspbian has since pushed security updates. End users install
  off the published tarball on release day — a cached build hides
  the fact that users are running against currently-available
  system libraries rather than the ones that were fresh at cache
  time. A deliberate invalidation policy would need to be
  designed and operated.
- **Complexity vs benefit.** Either `actions/cache` (per-branch,
  auto-expiring) or a GHCR-hosted prebaked image (persistent but
  needs a separate image-build workflow). Both add moving parts
  for modest time saving on a low-frequency pipeline.

Not done. Revisit if release cadence picks up or CI minutes
become a constraint.

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

1. **Run-to-run, same day** — triggering the workflow twice on the
   same commit, minutes apart, produces byte-identical tarballs.
2. **Per-tag, long-term** — triggering the workflow a year later
   on the same tag produces byte-identical tarballs.
3. **Bit-for-bit with the historical hand-built tarballs** — not a
   goal (see [§8](#8-things-we-tried-and-dropped)).

Level 1 is **guaranteed by construction** below. Level 2 holds
*modulo distro security updates* on the arm cells — see the known
limitation section.

### What this pipeline does to achieve it

**Fixed `SOURCE_DATE_EPOCH`.** Step 5 reads the triggering commit's
Unix timestamp and exports it to `$GITHUB_ENV`. Every downstream
build tool respects it:

| Tool | Effect |
|---|---|
| `gcc` | `__DATE__` / `__TIME__` macros use SDE (no impact on current SBFspot source, safe for future use). |
| `ld` | `--build-id=sha1` computes a SHA1 over content, not time, so the build-id is deterministic given deterministic input. |
| `ar` | Archive member timestamps → 0 (binutils ≥ 2.35). |
| `objcopy` | Section mtime fields → 0. |
| `tar` | `--mtime=@${SDE}` writes SDE into every member header. |
| `gzip` | `gzip -n` suppresses its own mtime + filename fields. |

**Deterministic tar format.** `--sort=name` + `--owner=0 --group=0
--numeric-owner` + `--format=ustar`. Member order and per-member
metadata are fully determined by the filename list + SDE.

**Pinned Debian snapshot for arm64, derived from the tag.** Step 5
pulls from `snapshot.debian.org/archive/debian/<SNAP>T000000Z`,
where `<SNAP>` is the UTC day of the triggering commit's
timestamp. `snapshot.debian.org` preserves every daily archive
forever, so the URL resolves identically in future. Each tagged
release builds against a coherent Debian snapshot from its own
day — a self-contained per-release pin.

**No `git`-state noise.** `actions/checkout` always produces the
exact commit tree that triggered the run. For a tag push that's
the tagged snapshot; same bytes forever.

**Per-cell SBOM.** Step 14 records name, version, architecture,
and `.deb` SHA256 for every package that actually affects the
build (runtime-linked libraries from the binary's NEEDED entries
plus the explicit build-tool install list — ~15 packages per
cell). Shipped on the Release as
`sbfspot-<cell-id>.packages.list`. Given the version string,
Debian-originated packages are retrievable forever from
`snapshot.debian.org` — both source and binary. For Raspbian-
patched (`+rpi*`) packages, the SHA256 lets an auditor verify a
recovered `.deb` from third-party mirrors or local caches even
after `archive.raspbian.org` has moved on.

### Known limitation: live Raspbian archive

The 9 arm cells use the **live** Raspbian archive
(`archive.raspbian.org` / `legacy.raspbian.org`) — not a snapshot,
because Raspbian doesn't run a functional snapshot service. That
means:

- If Raspbian pushes a security update to `libc6`, `libgcc-s1`,
  `libstdc++6`, `libboost-date-time1.*`, `libsqlite3-0`,
  `libmariadb3`, or `libcurl4` between run N and run N+1, those
  libraries shift in the chroot and the compiled binaries in the
  affected arm cells change.
- For a single tag-release build this doesn't matter: the
  maintainer tags, the workflow runs once, the tarballs get
  uploaded, the release is frozen.
- For a rebuild of an older tag months later, expect the arm
  binaries to differ in library bytes (same source, same flags,
  different libc). Functional behaviour unchanged.

Mitigations available if this becomes important:
- Bake each codename's rootfs as a Docker image on GHCR once,
  re-use it per matrix cell. Deterministic across time, adds an
  image-maintenance workflow.
- Pin specific package versions via `apt-get install pkg=x.y.z`
  per cell. Fragile; apt doesn't always cooperate on downgrades.

Not applied by default — the marginal benefit doesn't justify the
extra machinery for a release cadence of a few tags per year.

### What this pipeline intentionally does not reproduce

- **The historical hand-built tarballs' gzip-header bytes.** Those
  were written by Windows 7-Zip, with `os=00` (FAT), `xfl=04`
  (`--fast`), and a populated wall-clock `mtime`. This pipeline
  writes `os=03` (Unix), `xfl=00`, `mtime=0`. Reproducing the old
  format would need a custom gzip writer — no user-visible gain.
- **Per-file mtimes from the maintainer's local checkout.** The
  historical tarballs preserve filesystem mtimes spanning
  2021–2024 per file. This pipeline stamps every member with a
  single `SOURCE_DATE_EPOCH`. Both are reproducible; the SDE
  approach is more idiomatic for "this tarball represents a
  specific tag."
- **`0777` permission bits.** The historical tarballs use `0777`
  on every file. This pipeline uses Unix-conventional `0755` for
  executables and `0644` for data.

### How to verify reproducibility locally

```sh
# Trigger the workflow twice via the UI or gh CLI, then:
gh run download --repo <owner>/SBFspot <run-id-A> -p 'sbfspot-*'
gh run download --repo <owner>/SBFspot <run-id-B> -p 'sbfspot-*'
find . -name '*.tar.gz' -exec sha256sum {} + | sort
```

Group the output by filename and diff the hashes. An empty diff
means clean reproducibility across the two runs.

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
`sources.list`; added inline by step 9.

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
architecture. In this pipeline each chroot *is* the sysroot; no
separate cross-compiler toolchain is needed because we build
natively in the chroot.

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
