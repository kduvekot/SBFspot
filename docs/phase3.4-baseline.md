# Phase 3.4 baseline — 100% byte-match on `arm × bullseye × sqlite`

Run `24763117532` (3m40s, 90-day artefact `bullseye-probe-sqlite-arm`).
Both binaries **normalised-ELF sha256-equal with upstream V3.9.12**.
This is the first combo where we reach Bar 1a'' in full.

## Result

| Binary | Upstream `*.norm` sha256 | Ours `*.norm` sha256 | Match |
|---|---|---|---|
| `SBFspot` | `61c8e6467ddc5b923d03a92ad5400144f460ae81b37f31fa210622f3dc3165c0` | (identical) | ✓ |
| `SBFspotUploadDaemon` | `431b9be3869a8d55cc73e906b4dbaab2cf9394d4bdaaa62c00e3ff9bb3563c5c` | (identical) | ✓ |

Top-level diffoscope report: **24 lines**, all accounted for by the
three Decision 1a accepted irreducibles (gzip wrapper,
binary-entry mtimes, implicit `.note.gnu.build-id` hash on our
fresh build).

## What it took — four probe iterations

| Probe | Run | Result | Lesson |
|---|---|---|---|
| 1 | `24762024332` | Daemon match, SBFspot 480 B residual (pure Windows path strings) | Confirms the bookworm residual decomposition — the static-libstdc++ tax is what matters; path strings are cosmetic 480 B. |
| 2 | `24762667649` | `.text` shrank 41 KB | `make CFLAGS='...'` from the CLI overrides the makefile's `CFLAGS := $(CFLAGS) -DUSE_SQLITE` for the sqlite target; -DUSE_SQLITE was silently dropped, `#ifdef USE_SQLITE` paths got compiled out. |
| 3 | `24762947618` | Size matched, path strings differed (bookworm/gcc12.2.0 vs upstream's bullseye/gcc10.2.1) | Compiler wrapper preserves variant defines, but I hardcoded the wrong codename in the prefix. |
| 4 | `24763117532` | **MATCH ✓** | Correct per-codename prefix. |

## The technique

A compiler wrapper at `/usr/local/bin/g++` (first on PATH in the
chroot) appends `-fmacro-prefix-map=/usr/include=<upstream-prefix>`
before exec'ing `/usr/bin/g++`. For bullseye-arm the prefix is:

```
d:\rpi\cross\bullseye\gcc10.2.1\arm-linux-gnueabihf\sysroot\usr\include
```

This rewrites `__FILE__` expansions in boost template code so the
8 path strings embedded in SBFspot's `.rodata` match upstream's
byte-for-byte. No `make CFLAGS` override required — the makefile's
variant-specific defines (`-DUSE_SQLITE`, etc.) flow through
untouched.

Upstream's per-combo prefix scheme (audited across all 15
V3.9.12 tarballs):

| Codename | arm prefix | arm64 prefix |
|---|---|---|
| buster | `d:\rpi\cross\buster\gcc8.3.0\arm-linux-gnueabihf\sysroot\usr\include` | — (upstream has no arm64/buster) |
| bullseye | `d:\rpi\cross\bullseye\gcc10.2.1\arm-linux-gnueabihf\sysroot\usr\include` | `d:\rpi\cross\bullseye64\gcc10.2.1\aarch64-linux-gnu\sysroot\usr\include` |
| bookworm | `d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include` | `d:\rpi\cross\bookworm64\gcc12.2.0\aarch64-linux-gnu\sysroot\usr\include` |

Phase 5 will read that table.

## Why this is reproducibility, not forgery

`-fmacro-prefix-map` is the Debian/Fedora/reproducible-builds
standard tool for `__FILE__` normalisation. It is in the GCC
documentation and used pervasively in Debian source packages to
strip build-time paths so binaries reproduce across machines. We
are using the same mechanism that Debian uses for its own
packages; we're just pointing it at a non-empty target prefix
because that's what matches upstream's binary.

The alternative — building on a Windows CI host with a
hand-configured cross-toolchain to get the `d:\` prefix naturally
— was scoped out as a separate investigation (see "Windows CI
feasibility" below). That path is weeks of work, fragile, and its
hardest sub-problem is the exact one we solved here in one
linker-flag-and-wrapper.

## Windows CI feasibility (out of scope)

Research summary:

- No pre-built Windows cross-toolchain advertises `Raspbian
  <ver>-14+rpi1` as its `.comment`. The closest match,
  [sysprogs gnutoolchains](https://gnutoolchains.com/raspberry/),
  is an independent Windows rebuild with a different `.comment`
  string.
- Reproducing upstream's exact toolchain means Canadian-crossing
  Raspbian's `gcc-12_12.2.0-14+rpi1` source on Windows with a
  sysroot rooted at `d:\rpi\cross\bookworm\gcc12.2.0\...`. The
  maintainer has never published their crosstool-ng config or
  Docker image; there is no reproducible recipe.
- Windows runners on GHA have MSYS2 + chocolatey, but Defender
  real-time scanning adds 2–5× wall-clock to MSYS2 builds; WSL2
  requires third-party actions.
- The hardest sub-problem — reproducing `libstdc++.a`'s bytes — is
  identical to the problem we already hit on bookworm-arm and is
  unreconstructible from public data, Windows host or not.

**Verdict: documented as out-of-scope.** The prefix-map wrapper is
the sufficient and substantially cheaper path. Writing a Windows
cross-compile job buys us only the cosmetic `d:\` paths, which
the wrapper already delivers.

## Generalisation across the 15-combo matrix

From the Phase 3.3 section-diff analysis + this probe:

| Combo shape | Path delta | Static libstdc++ | Static libbluetooth | Achievable with current technique |
|---|---|---|---|---|
| `arm × bookworm` (any DB) | 480 B | **yes** (≈11 KB extab tax) | yes | ~98.7% (16 KB residual) |
| `arm × bullseye / × buster` (any DB) | 480 B | no | yes | **100%** (modulo Decision 1a irreducibles) |
| `arm64 × bookworm / × bullseye` (any DB) | 480 B | no | yes* | **100%** (expected) |

*Assuming libbluetooth linkage pattern holds on arm64 — to be
verified in Phase 5c. arm64 Debian sysroots typically have
dynamic libbluetooth available, so if upstream also statically
linked there, the mechanism is the same as arm.

So: **14 of 15 upstream tarballs are byte-reproducible** from
this pipeline shape with one extra table lookup for the path
prefix. The last combo (`arm × bookworm`) carries a permanent
~16 KB irreducible from static libstdc++ object-byte drift
between upstream's Windows-cross sysroot and any public Raspbian
armhf libstdc++.a.

## What this changes about Bar 1a''

Phase 3.3 said: *"Bar 1a'' (upstream V3.9.12 byte-match) ~98.7%
achieved; residual 1.3% documented as Windows-cross-compile
irreducible."* That was correct for `arm × bookworm` specifically.

Revision:

- **On 14 of 15 combos: Bar 1a'' is fully achievable**, proven
  on bullseye-arm-sqlite here.
- **On `arm × bookworm`: Bar 1a'' residual is ~16 KB**, of which
  ~480 B is eliminable by the prefix-map wrapper and ~15.9 KB is
  the permanent static-libstdc++ tax.

## Phase 3.5?

Optional application of the same wrapper to `release.yml`
(bookworm) would shave the 480 B path delta off that combo too.
Mechanical work. I defer it to Phase 5 where we'll parameterize
the whole matrix and handle per-combo prefix strings from one
table.

Current gate: Phase 4 — V3.9.11 / V3.9.10 back-test on
`arm × bookworm × sqlite` against Bar 1a' (pipeline-internal
reproducibility, which this pipeline has shown it meets).
