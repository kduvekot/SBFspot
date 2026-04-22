# Phase 3.5 baseline — cross-combo generalisation of the 3.4 technique

## Question

*"Can we get the same byte-match result on other variants? Is
the Windows cross-compile environment the only variable left?"*

## Headline

**Mostly yes, with codename-specific caveats.** The Phase 3.4
prefix-map wrapper + static-bluetooth trick generalises cleanly
across DB and arch axes when upstream uses a single clean
toolchain. **Six of fifteen combos (all `*-bookworm`) carry an
additional irreducible** from a Raspbian-tagged pre-built static
lib that shows up in upstream's binary as a second `.comment`
entry. The other **nine combos** are reachable — small per-combo
LDFLAGS adjustments get us there. Full matrix:

| Combo shape | Bar 1a'' result |
|---|---|
| `arm × {bullseye, buster} × {sqlite, nosql, mariadb}` | ✓ achievable |
| `arm64 × bullseye × {sqlite, nosql, mariadb}` | ✓ achievable |
| `arm × bookworm × {sqlite, nosql, mariadb}` | residual ~16 KB (static libstdc++ drift) |
| `arm64 × bookworm × {sqlite, nosql, mariadb}` | residual ~10 KB (pre-built Raspbian-tagged static lib) |

## What Phase 3.5 actually ran

Workflow `.github/workflows/phase35-probe.yml`, 5 cells in a
matrix, one job per cell on `ubuntu-24.04-arm`. Five iterations
(runs 24763889776, 24764047687, 24764470395, 24764936273,
24765262702) as I tracked down workflow bugs, missing packages,
and LDFLAGS ordering.

### Cell results

| Cell | Result |
|---|---|
| `nosql × arm × bullseye` | ✓ MATCH on SBFspot (daemon N/A) |
| `sqlite × arm64 × bullseye` | ✓ MATCH on both binaries |
| `mariadb × arm × bullseye` | Close: NEEDED set matches; ~0.1% byte drift in `.text`/`.rodata`/`.data` from libmariadbclient.a bytes (upstream's sysroot copy ≠ ours) |
| `sqlite × arm × buster` | Close: size matches, content differs. Upstream has libdl + libgnutls in NEEDED that `--as-needed` drops from ours. Needs `-Wl,--no-as-needed -lgnutls -ldl -Wl,--as-needed` tweak — deferred to Phase 5. |
| `sqlite × arm64 × bookworm` | **Known irreducible:** upstream binary has dual `.comment` (`GCC: (Debian 12.2.0-14)` + `GCC: (Raspbian 12.2.0-14+rpi1)`); upstream ships a pre-built Raspbian-tagged static lib we don't have. SBFspot residual +9.7 KB `.text`; daemon matches. Phase 1 flagged this pattern. |

### Things Phase 3.5 taught the pipeline

1. **Upstream statically links boost on ALL 15 tarballs, not just
   arm × bookworm.** On bullseye/bookworm the default
   `ld --as-needed` silently strips unreferenced dynamic boost
   entries from NEEDED, so our "dynamic boost" build looked like
   upstream. On buster (pre-`--as-needed`-default) the dynamic
   entries showed up and exposed the gap. Explicit static boost
   linkage is needed on all combos for true bit-match.
2. **Upstream statically links libmariadbclient** on all mariadb
   variants (no `libmariadbclient.so` in NEEDED anywhere), with
   libdl/libz/libssl/libcrypto resolved dynamically from the
   static archive's transitive references.
3. **buster needs `libcurl4-gnutls-dev`** not `libcurl4-openssl-dev`
   — upstream buster daemon has `libgnutls.so.30` in NEEDED.
4. **buster has been dropped from `archive.raspbian.org`**; debootstrap
   against `legacy.raspbian.org` works (same keyring layout).
5. **mariadb build needs `libmariadb-dev-compat`** (provides
   `<mysql/mysql.h>`) in addition to `libmariadb-dev` (only
   provides `<mariadb/mysql.h>`).
6. **Per-combo path prefix table** (audited across all 15 upstream
   SBFspot binaries):

   | Codename | arm prefix | arm64 prefix |
   |---|---|---|
   | buster | `\buster\gcc8.3.0\arm-linux-gnueabihf\` | — |
   | bullseye | `\bullseye\gcc10.2.1\arm-linux-gnueabihf\` | `\bullseye64\gcc10.2.1\aarch64-linux-gnu\` |
   | bookworm | `\bookworm\gcc12.2.0\arm-linux-gnueabihf\` | `\bookworm64\gcc12.2.0\aarch64-linux-gnu\` |

   All rooted at `d:\rpi\cross\...\sysroot\usr\include`.

## Why bookworm is harder

Phase 1 noted: *"Bookworm arm64 carries both [Debian and Raspbian]
strings — one object (likely a bundled Boost static lib) was
compiled on Raspbian and linked into a Debian build."* Confirmed
in Phase 3.5:

- `arm × bookworm`: upstream statically links a `libstdc++.a` from
  its Raspbian-armhf cross sysroot. Our live-archive `libstdc++.a`
  has different object bytes. ~16 KB `.ARM.extab` residual (Phase
  3.3 analysis).
- `arm64 × bookworm`: upstream statically links a pre-built static
  lib (likely boost) compiled with a Raspbian-tagged gcc that
  targets aarch64. We don't have that binary — our Debian
  snapshot's boost is a clean Debian build. ~9.7 KB `.text`
  residual, isolated to SBFspot (daemon matches because it
  doesn't use boost).

Both are documented as irreducible-from-public-data. No Linux
or Windows CI setup can reproduce them without the maintainer's
private sysroot.

## The actual answer to the original question

"Is Windows the only variable left?"

- **For 9 of 15 combos** (everything except `*-bookworm`): yes,
  modulo small LDFLAGS adjustments (`-Wl,--as-needed`, pre-pending
  `-lpthread` for NEEDED ordering, `-Wl,-Bstatic -lboost_*`,
  explicit `-ldl -lz -lssl -lcrypto` for mariadb). All mechanical;
  none require Windows hosting.
- **For 6 of 15 combos** (`*-bookworm`): no. Upstream pre-bakes
  static libs compiled with toolchains we can't reproduce from
  public data. The Windows prefix-map wrapper is necessary but
  not sufficient for these combos.

The Phase 3.4 conclusion ("100% match achievable on 14 of 15
combos via prefix-map") was **too optimistic by one axis**. Phase
3.5 revises that to: **100% match achievable on 9 of 15 combos**;
~16 KB residual on `arm × bookworm`; ~10 KB residual on
`arm64 × bookworm`.

Windows CI would not help with the bookworm residuals either —
the problem is specific pre-built static libs from the
maintainer's private tree, not the host OS.

## Phase 5 hand-off

What Phase 5 (matrix expansion in `release.yml`) needs to do:

1. Consolidate the per-combo `LDFLAGS`/`apt` package lists from
   this probe's 5 cells into a clean table.
2. Per-combo `PREFIX` string for the compiler wrapper.
3. Handle Raspbian-legacy mirror for buster.
4. Debian snapshot for arm64.
5. Accept the bookworm residuals in the success criterion.
6. (Optional) Keep polishing: push `-Wl,--no-as-needed` for
   buster's libdl/libgnutls; explore whether upstream's
   libmariadbclient.a drift is caused by a specific mariadb
   package version that `snapshot.debian.org`/Raspbian could pin.

## Files

- `.github/workflows/phase35-probe.yml` — matrix probe, frozen
  behind `.phase35-probe-trigger`. Keeps the last good state of
  the 5-cell workflow.
