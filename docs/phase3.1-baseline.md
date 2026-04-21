# Phase 3.1 baseline — reducible tar/mtime/CRLF diffs collapsed

> **Correction (Phase 3.2):** This doc's "Surviving reducible:
> the toolchain drift" section attributes the 591k-line residual
> diff to `+rpi1` → `+rpi1+deb12u1`. That's wrong; the `+rpi1`
> rebuild produces the same bytes we produce here. The real
> cause is Raspbian's original build-farm state. See
> `docs/phase3.2-baseline.md`.

Second green end-to-end run of the release pipeline, after fixing
the three reducible diffs listed in `phase2-baseline.md`. Artefact
lives in GitHub Actions run `24745197356`
(`release-mvp-sqlite-arm-bookworm`, 90-day retention).

## What changed vs Phase 2

1. **CRLF on 8 text files** (`TagList*.txt`, `CreateSQLiteDB.sql`,
   `SBFspotUpload.default.cfg`) via `unix2dos -q`. The other two
   text files ship CRLF-in-repo and were already byte-identical.
2. **`tar --format=ustar`** to match upstream's plain POSIX tar
   (was GNU extended).
3. **Second-precise mtimes** on every non-binary entry, from the
   upstream tar listing (e.g. `2024-02-13 13:36:02` instead of
   `2024-02-13 13:36`).

Implemented in `.github/workflows/release.yml` commit `f3f01eb`.

## Sha256s

| Artefact | Sha256 |
|---|---|
| upstream `sbfspot-sqlite-arm-linux-bookworm.tar.gz` | `887a393a64dc6d0924c9afa92047002b95a42395c1a2a20ecc09ca71acacabb0` |
| Phase 2 (run 24742260942) | `bb8f5defec5e08b72aed9a0317ba7c08588c1d033b75dbf0f8955fd478cbc704` |
| Phase 3.1 (run 24745197356) | `a5c518d9487511099bee0ddd8606b0b9037b68e893479218ff5dd760a8ef01ff` |

Compressed sizes: upstream 1,601,125 B, Phase 2 1,851,143 B,
Phase 3.1 1,858,391 B (+7,248 B vs Phase 2 — CRLF bytes we
deliberately added to catch up with upstream's line endings).

## Diff collapse

| Layer | Phase 2 | Phase 3.1 |
|---|---|---|
| Top-level diffoscope report (lines) | ~694,000 | 591,469 |
| `diffoscope-SBFspot.txt` (normalised-ELF) | ~348,000 | 346,951 |
| `diffoscope-SBFspotUploadDaemon.txt` (normalised-ELF) | ~242,000 | 244,446 |

The ~100k-line collapse at the top level matches theory: Phase 2's
top-level report was "binaries + all 8 CRLF-differing text files +
file-list churn"; Phase 3.1 leaves only the binaries.

**The diffoscope `file list` section now shows only the two
binaries differing**, and only in size + mtime:

```
 -rwxrwxrwx   0  0  0     7133 2024-02-13 13:36:02.000000 CreateSQLiteDB.sql
 -rwxrwxrwx   0  0  0    48487 2021-01-17 18:43:04.000000 date_time_zonespec.csv
--rwxrwxrwx   0  0  0  1242036 2025-02-22 09:49:46.000000 SBFspot
+-rwxrwxrwx   0  0  0  1131432 2026-04-21 20:41:48.000000 SBFspot
 -rwxrwxrwx   0  0  0    10958 2024-02-13 13:36:03.000000 SBFspot.default.cfg
 -rwxrwxrwx   0  0  0     2058 2024-04-29 08:52:56.000000 SBFspotUpload.default.cfg
--rwxrwxrwx   0  0  0   885472 2025-02-22 09:50:15.000000 SBFspotUploadDaemon
+-rwxrwxrwx   0  0  0   881384 2026-04-21 20:41:48.000000 SBFspotUploadDaemon
 -rwxrwxrwx   0  0  0   461754 2024-02-13 13:36:03.000000 TagListDE-DE.txt
 -rwxrwxrwx   0  0  0   447861 2024-06-15 17:28:09.000000 TagListEN-US.txt
 -rwxrwxrwx   0  0  0   481867 2024-02-13 13:36:03.000000 TagListES-ES.txt
 -rwxrwxrwx   0  0  0   491273 2024-02-13 13:36:03.000000 TagListFR-FR.txt
 -rwxrwxrwx   0  0  0   470617 2024-02-13 13:36:03.000000 TagListIT-IT.txt
 -rwxrwxrwx   0  0  0   453607 2024-02-13 13:36:03.000000 TagListNL-NL.txt
```

Ten text entries: matched. Two binary entries: size + mtime only.

## What's left

### Accepted irreducibles (per Decision 1a, unchanged from Phase 2)

1. **Gzip wrapper.** Upstream `os=00 xfl=04 mtime=populated-UTC`
   (Windows-side writer). Ours `os=03 xfl=04 mtime=0`. Visible in
   diffoscope as:
   ```
   -gzip compressed data, last modified: Sat Feb 22 09:50:15 2025, max speed, from FAT filesystem (MS-DOS, OS/2, NT)
   +gzip compressed data, max speed, from Unix
   ```
2. **Wall-clock mtime on freshly-built binaries inside the tar.**
   Upstream 2025-02-22; ours is run time.
3. **Compiler-version strings zeroed** via the per-binary
   normalised-ELF compare.

### Surviving reducible: the toolchain drift

The 591k-line normalised-ELF diff is entirely downstream of one
package-version bump:

```
- GCC: (Raspbian 12.2.0-14+rpi1) 12.2.0           (upstream)
+ GCC: (Raspbian 12.2.0-14+rpi1+deb12u1) 12.2.0   (ours, live archive.raspbian.org)
```

Sizes confirm it propagates through the statically-linked
libstdc++.a:

| Binary | Upstream | Ours | Delta |
|---|---|---|---|
| `SBFspot` | 1,242,036 B | 1,131,432 B | **−110,604 B (−8.9%)** |
| `SBFspotUploadDaemon` | 885,472 B | 881,384 B | **−4,088 B (−0.5%)** |

Identical deltas to Phase 2 (−110,612 / −4,096 B at the
normalised-ELF layer). This is unchanged by Phase 3.1 — Phase 3.1
only touched payload packaging, not the toolchain.

## Phase 3.2 decision

The drift is the Decision 2A data point we gathered live. Now
Decision 2B is on the table:

- **(A) Accept the drift.** Document `+deb12u1` as an irreducible
  delta driven by the upstream security-update cadence we can't
  undo. Normalised-ELF diff stays at ~591k lines but the cause is
  understood. Rebuilds stay reproducible against a fixed snapshot
  date only if Raspbian packages don't move, which they do.
- **(B) Pin package versions via a baked rootfs.** Build the
  Raspbian Bookworm armhf chroot once with
  `libstdc++-12-dev=12.2.0-14+rpi1` and the rest of the upstream
  toolchain pinned, cache the tarball on GHCR, restore it in the
  release workflow. Collapses the 591k diff to whatever is left
  after the common-irreducibles (build-id, binary mtime). Adds
  one-off infrastructure (GHCR image build job) plus a refresh
  strategy.

Recommendation: **(B).** The 110 KB drift is the last knob we can
turn, and the Decision 2A data was collected precisely to
determine whether to pull it. We now have that data. Deferring (B)
leaves the pipeline "reproducible up to external package cadence,"
which defeats the point of a reproducibility bar.
