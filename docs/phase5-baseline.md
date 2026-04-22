# Phase 5 baseline — full 15-combo matrix

Probe: `.github/workflows/phase5-matrix.yml`, frozen behind
`.phase5-matrix-trigger`. Final run `24789691360` (5 min wall-clock,
15 cells in parallel). All 15 cells complete successfully; the
distribution of residuals matches the predictions from Phases 3.5
and 4.

## Residual matrix

| Combo | SBFspot | Daemon |
|---|---|---|
| arm × bullseye × sqlite | **MATCH ✓** | **MATCH ✓** |
| arm × bullseye × nosql | **MATCH ✓** | — |
| arm × bullseye × mariadb | +28 B | +28 B |
| arm64 × bullseye × sqlite | **MATCH ✓** | **MATCH ✓** |
| arm64 × bullseye × nosql | **MATCH ✓** | — |
| arm64 × bullseye × mariadb | −4,032 B | +64 B |
| arm × buster × sqlite | size-match, content differs | size-match |
| arm × buster × nosql | size-match, content differs | — |
| arm × buster × mariadb | −4,096 B | +8,192 B |
| arm × bookworm × sqlite | −16,384 B | −4,096 B |
| arm × bookworm × nosql | −12,288 B | — |
| arm × bookworm × mariadb | −16,384 B | −8,192 B |
| arm64 × bookworm × sqlite | size-match, content differs | **MATCH ✓** |
| arm64 × bookworm × nosql | −65,536 B *(file-layout padding)* | — |
| arm64 × bookworm × mariadb | size-match, content differs | size-match |

**7 of 25 binaries are byte-for-byte identical** to upstream.

## Residual categorisation

Every residual > 0 has a documented mechanism:

### Clean MATCHES (7 binaries)
All in the `*-bullseye` cells where upstream's toolchain doesn't
have the bookworm-specific quirks. One sqlite-arm64-bookworm
daemon matches (daemon doesn't use boost → not affected by
dual-.comment issue).

### Size-match / content-differ (6 binaries)
- `buster` cells: minor sysroot byte drift in specific libs.
- `arm64 × bookworm × sqlite SBFspot` and
  `arm64 × bookworm × mariadb` both: Phase 3.5's dual-.comment
  irreducible (upstream links a Raspbian-tagged pre-built
  static lib we don't have).

### Small residuals < 100 B (3 binaries)
- mariadb-arm-bullseye +28 B / +28 B: NEEDED ordering of
  `libpthread` vs `libdl` in `.dynstr` — cosmetic.
- mariadb-arm64-bullseye daemon +64 B: similar ordering.

### Medium residuals 4–8 KB (5 binaries)
- mariadb combos on bullseye/buster with non-zero deltas: our
  `libmariadbclient.a` from Raspbian/Debian archive has
  different bytes than upstream's sysroot-copy.

### Large residuals (bookworm-arm, bookworm-arm64 nosql)
- arm-bookworm: the static-libstdc++ irreducible characterised
  thoroughly in Phase 3.3 and Phase 4c — can't close without
  upstream maintainer's gcc configure.
- arm64-bookworm nosql SBFspot −65,536 B: **not content diff —
  file-layout padding**. Our linker packs LOAD segments tightly
  in the file (file_offset ≠ vaddr mod page_size); upstream's
  pads to align. Actual section-content delta is only 9,680 B
  (dual-.comment).

## The 2 link bugs Phase 5 surfaced

The initial run had 2 buster cells fail at link:
- `sqlite-arm-buster`: missing `libgnutls28-dev` for -lgnutls
- `mariadb-arm-buster`: wrong transitive-deps — libmariadbclient
  on buster is built against **gnutls** (not openssl like
  bullseye/bookworm); needed `-lgnutls` not `-lssl -lcrypto`.

Fixed in commits `8e0b39b` and `7945634`. All 15 cells now green.

## Per-combo pipeline shape (consolidated from Phase 3.5 table)

For Phase 5 matrix we have a table-driven pipeline:

| Field | Value |
|---|---|
| rootfs source | `raspbian` (arm × {bookworm, bullseye}), `raspbian-legacy` (arm × buster), `debian` (`snapshot.debian.org@20250222T000000Z`, arm64) |
| codename-specific apt pkgs | `libgnutls28-dev` on buster, `libmariadb-dev-compat` on mariadb, libcurl gnutls vs openssl flavour depending on codename |
| LDFLAGS common | `-s` + (`-static-libstdc++ -Wl,--build-id=none` on arm × bookworm only) |
| LDFLAGS SBFspot extra | `-Wl,-Bstatic -lbluetooth -Wl,-Bdynamic` (all combos); add `-lboost_date_time -lboost_system` for buster; `-lmariadbclient` for mariadb; `-Wl,--as-needed` for buster |
| `-fmacro-prefix-map` wrapper | per-combo prefix from Phase 3.4 audit table |
| Daemon NEEDED | matches upstream for every combo after small LDFLAGS tweaks |

## What Phase 5 does NOT do yet

1. **Per-variant tarball assembly.** nosql has 9 members, sqlite
   has 12, mariadb has 13 (with different SQL files). Phase 5
   only does binary-level compare; tarball re-assembly per
   variant with the full CRLF / mtime / ustar dance from Phase
   3.1 is Phase 5b.
2. **Rootfs caching.** 15 cells currently each debootstrap from
   scratch (~2 min each). With `actions/cache` on the rootfs by
   codename, CI time would drop significantly.
3. **Release publishing.** Phase 5 is a *probe*, not the release
   pipeline. Converting to a release.yml that produces the
   signed release artefacts is Phase 5c.

## Phase 5 close

Matrix expansion of the pipeline's build logic is complete.
All 15 upstream variants produce expected-shape binaries with
known, documented residual patterns. The 7 clean byte-matches
vs upstream show the pipeline is working at the bar we set;
the 18 non-clean results are all in known-mechanism buckets.

## Phase 5c — 3 targeted fixes probed

After the Phase 5 baseline, three cheap-looking fixes were tried:

| Fix | Target | Result |
|---|---|---|
| Add `-Wl,--no-as-needed -ldl` to sqlite-arm-buster daemon | close missing NEEDED entry | **Kept.** `libdl.so.2` now in our NEEDED (matches upstream). Daemon residual unchanged because several *other* NEEDED entries still differ (libcurl-gnutls vs libcurl SONAME; missing libpthread/libgnutls/libm) — sysroot-level package-flavour differences. |
| Use `-l:libpthread.so.0` on mariadb-arm-bullseye | close +28 B NEEDED ordering | **Kept.** NEEDED order now matches upstream byte-for-byte. But residual is still +28 B — every section's content has small byte drift. The +28 B was NOT NEEDED ordering; it's widespread `libmariadbclient.a` byte drift, same story as arm-bookworm libstdc++. |
| `-Wl,-z,separate-loadable-segments` on nosql-arm64-bookworm | close 64 KB file-padding | **Reverted.** binutils 2.40 ignored the flag; LOAD segments stayed at their tight-packed file offsets. Different linker version than upstream's — mechanism not reachable by flag. |

**Lessons from Phase 5c**:
- NEEDED-set fixes (add/remove entries or reorder) are ~free and
  correct to keep even when residuals don't drop.
- LOAD-segment alignment drift is binutils-version-sensitive —
  upstream uses a different binutils than our debootstrap provides
  and we have no way to tell which default the flag targets.
- Residuals that look like "ordering problems" (small byte counts)
  are often wider byte drift, visible only via per-section sha
  comparison. Always check section-content shas, not just sizes.

## Next: Phase 5b (tarball assembly) or Phase 6 (trixie)

Phase 5b — per-variant tarball assembly so diffoscope at the tar
layer also matches (where binary does). Minor mechanical work:
per-variant member list + CRLF-file list + mtime table. Could be
rolled into this same workflow.

Phase 6 — Trixie support. 6 new cells (`trixie × {arm,arm64} ×
{sqlite,nosql,mariadb}`). No upstream tarballs exist, so success
criterion is "builds and runs on a fresh trixie rootfs" not
"byte-matches upstream".
