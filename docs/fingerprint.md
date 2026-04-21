# V3.9.12 upstream tarball fingerprint

Target spec for Phase 2. Records what must be reproduced to build
bit-equivalent tarballs, what is irreducible, and what has to be
matched approximately. Produced from local dissection of all 15
Linux release assets on 2026-04-21. Raw data lives under
`/tmp/phase1/` on the build host; this file is the summary.

## Source state

`SBFspot/makefile` in this fork matches the file at tag `V3.9.12`
byte-for-byte (129 lines, identical diff). No source divergence
blocking the rebuild.

Two binaries ship per DB variant:

- `SBFspot` — built from `SBFspot/makefile`.
- `SBFspotUploadDaemon` — built from `SBFspotUploadDaemon/makefile`,
  pulls sources from `../SBFspot` and `../SBFspotUploadCommon`. Only
  present in `sqlite` and `mariadb` tarballs; absent from `nosql`.
  Links `pthread curl` (plus the DB lib).

## Build provenance — Raspbian for arm, Debian for arm64

`.comment` strings in the ELF binaries are the cleanest signal:

| Variant | `.comment` |
|---|---|
| `arm × buster` | `GCC: (Raspbian 8.3.0-6+rpi1) 8.3.0` |
| `arm × bullseye` | `GCC: (Raspbian 10.2.1-6+rpi1) 10.2.1 20210110` |
| `arm × bookworm` | `GCC: (Raspbian 12.2.0-14+rpi1) 12.2.0` |
| `arm64 × bullseye` | `GCC: (Debian 10.2.1-6) 10.2.1 20210110` |
| `arm64 × bookworm` | `GCC: (Debian 12.2.0-14) 12.2.0` **and** `GCC: (Raspbian 12.2.0-14+rpi1) 12.2.0` |

**Implication — strategy pivot.** The original plan of "pin everything
to `snapshot.debian.org`" is wrong for the `arm` variants. Upstream's
maintainer builds `arm` on a Raspbian rootfs (likely a Pi). arm64
builds come from a Debian environment.

This is a real problem, because Phase 0 proved
`snapshot.raspbian.org` is not a functional archive service (root
`/` returns 200, all `/archive/...` paths 404). Options:

1. Pin against `archive.raspbian.org` live — accept the non-trivial
   risk that package updates drift the `.comment` / dynamic-linker
   version strings between reproducibility runs.
2. Bake a Raspbian rootfs at a known-good date and cache it as a
   repo artefact (OCI or tarball on GHCR).
3. Accept the `arm` variants can't hit raw-tarball `sha256` match,
   and target normalised-ELF match only for those.

Phase 2 will use option (1) for the MVP run and measure how large
the live-archive drift is over a few days. That experiment informs
Decision 2 (snapshot pinning strategy).

Bookworm arm64 having **both** Debian and Raspbian `.comment`
strings is curious: some object file was cross-pollinated between
the two rootfs, or a shared dep (e.g. a Boost `.a`) was compiled
in a Raspbian sysroot and linked into a Debian build. Will
re-investigate in Phase 3 when we try to make the arm64 build
match.

## Tarball layer (identical across all 15)

- **gzip header**: `method=08` (deflate), `flags=00`, `xfl=04`
  (`--fast`, ie `gzip -1`), `os=00` (FAT / MS-DOS / Win32).
- **gzip mtime field is populated** with the wall-clock mtime of
  the underlying `.tar` at the time of compression. Range:
  2025-02-22 09:42–10:17 UTC (~35-minute window). **Upstream does
  not use `gzip -n`**. To match raw tarball sha256, we have to
  either reproduce the wall-clock exactly (infeasible) or accept
  that the 4 mtime bytes will always differ.
- **tar entry order**: lexicographic. Matches `tar --sort=name`.
- **uid/gid**: `0/0` on every entry. Matches
  `tar --owner=0 --group=0 --numeric-owner`.
- **OS field `os=00`** strongly suggests a Windows-side gzip
  implementation (7-Zip, GnuWin32, or similar). Matching this on
  Linux requires a custom gzip write path. Punt to Phase 3 — if
  we can't match, document as an irreducible diff.

## Non-binary payload (byte-identical across all 15 tarballs)

Every non-binary file has exactly **one** distinct sha256 across
every tarball that contains it. Maintainer ships a single source
checkout; no per-variant processing.

| File | Present in | mtime preserved from |
|---|---|---|
| `date_time_zonespec.csv` | all 15 | 2021-01-17 18:43 |
| `SBFspot.default.cfg` | all 15 (renamed from `SBFspot.cfg`) | 2024-02-13 13:36 |
| `TagListDE-DE.txt` | all 15 | 2024-02-13 13:36 |
| `TagListEN-US.txt` | all 15 | 2024-06-15 17:28 |
| `TagListES-ES.txt` | all 15 | 2024-02-13 13:36 |
| `TagListFR-FR.txt` | all 15 | 2024-02-13 13:36 |
| `TagListIT-IT.txt` | all 15 | 2024-02-13 13:36 |
| `TagListNL-NL.txt` | all 15 | 2024-02-13 13:36 |
| `SBFspotUpload.default.cfg` | sqlite + mariadb (10 tarballs) | 2024-04-29 08:52 |
| `SBFspotUploadDaemon` | sqlite + mariadb (10 tarballs) | build time |
| `CreateSQLiteDB.sql` | sqlite (5 tarballs) | 2024-02-13 13:36 |
| `CreateMySQLDB.sql` | mariadb (5 tarballs) | 2024-02-13 13:36 |
| `CreateMySQLUser.sql` | mariadb (5 tarballs) | 2023-04-12 19:24 |

Per-file mtimes are preserved filesystem mtimes from the checkout —
not normalised, not touched to a single date. **No
`SOURCE_DATE_EPOCH` in upstream's flow.** To match, we need either
(a) `git archive`-style timestamp preservation from the tag, or
(b) a lookup table that forces each file to the observed mtime.

## Per-variant tarball composition

- **nosql**: 1× `SBFspot`, `SBFspot.default.cfg`,
  `date_time_zonespec.csv`, 6× TagLists → 9 entries.
- **sqlite**: nosql set + `SBFspotUploadDaemon`,
  `SBFspotUpload.default.cfg`, `CreateSQLiteDB.sql` → 12 entries.
- **mariadb**: nosql set + `SBFspotUploadDaemon`,
  `SBFspotUpload.default.cfg`, `CreateMySQLDB.sql`,
  `CreateMySQLUser.sql` → 13 entries.

## Binary fingerprint (SBFspot)

All binaries are **stripped** (makefile `LDFLAGS = -s` honoured).
arm builds are non-PIE executables; arm64 builds are PIE.

| Variant | Class | OSABI | BuildID | PIE |
|---|---|---|---|---|
| `arm × buster` | ELF32 ARM | GNU/Linux (3) | present | no |
| `arm × bullseye` | ELF32 ARM | SYSV (0) | present | no |
| `arm × bookworm` | ELF32 ARM | SYSV (0) | **ABSENT** | no |
| `arm64 × bullseye` | ELF64 aarch64 | SYSV (0) | present | yes |
| `arm64 × bookworm` | ELF64 aarch64 | SYSV (0) | present | yes |

Observations:

- **buster OSABI is `GNU/Linux`** (e_ident[EI_OSABI]=3) while every
  newer build is `SYSV` (0). This is a binutils-8 era default; GCC
  10+ / binutils 2.35+ default to SYSV. Matching means matching
  binutils version.
- **bookworm arm alone has no `.note.gnu.build-id`**. Every other
  variant has one. Either the maintainer invoked
  `-Wl,--build-id=none` on that combo, or the Raspbian Bookworm
  armhf link-time default differs. Investigate in Phase 2.

### Dynamic dependencies (`NEEDED`)

Common denominator: `libc.so.6`, `libgcc_s.so.1`, `libm.so.6`,
plus the platform dynamic linker.

| Variant | Adds |
|---|---|
| `nosql × arm × buster` | `libstdc++.so.6 libpthread.so.0 libgnutls.so.30` |
| `nosql × arm × bullseye` | `libstdc++.so.6 libpthread.so.0` |
| `nosql × arm × bookworm` | *(nothing — statically links libstdc++; libpthread merged into libc)* |
| `nosql × arm64 × bullseye` | `libstdc++.so.6 libpthread.so.0` |
| `nosql × arm64 × bookworm` | `libstdc++.so.6` (libpthread merged into libc) |
| `sqlite × *` | above + `libsqlite3.so.0` |
| `mariadb × buster` | above + no libssl (TLS via gnutls) |
| `mariadb × bullseye` | above + `libssl.so.1.1 libcrypto.so.1.1` |
| `mariadb × bookworm` | above + `libssl.so.3 libcrypto.so.3` |

Two codename-driven behaviours from the toolchain/runtime:

- **glibc 2.34+ merger**: `libpthread.so.0` becomes a stub and is no
  longer `NEEDED` by new binaries. Bookworm (glibc 2.36) drops it.
- **Boost TLS backend**: buster boost uses GnuTLS (`libgnutls.so.30`
  NEEDED); bullseye/bookworm use OpenSSL (libssl).

### The bookworm arm bloat

`arm × bookworm` binaries are 2–3× the size of their
bullseye/buster/arm64 siblings:

| Binary | SBFspot size (bytes) |
|---|---|
| `nosql × arm × bookworm` | 1,188,704 |
| `nosql × arm × bullseye` | 403,808 |
| `nosql × arm × buster` | 387,484 |
| `nosql × arm64 × bookworm` | 528,256 |
| `sqlite × arm × bookworm` | 1,242,036 |
| `sqlite × arm × bullseye` | 448,948 |
| `mariadb × arm × bookworm` | 1,418,368 |
| `mariadb × arm × bullseye` | 624,944 |

Cause: bookworm-arm binaries **statically link libstdc++** (see
NEEDED table — `libstdc++.so.6` is absent). `.text` grows from
~285 KB to ~990 KB for nosql, consistent with an embedded libstdc++.
Almost certainly a consequence of the maintainer's Raspbian
Bookworm armhf environment (package availability / linker
defaults), not a deliberate flag. To reproduce we either replicate
that environment or force `-static-libstdc++` on the
`arm × bookworm` link line.

## Phase 2 target

Build `sqlite × arm × bookworm` against V3.9.12 and diffoscope
vs `sbfspot-sqlite-arm-linux-bookworm.tar.gz`
(sha256 `887a393a64dc6d0924c9afa92047002b95a42395c1a2a20ecc09ca71acacabb0`,
1,601,125 bytes compressed / 5,012,992 bytes `.tar`).

Specific targets for the first MVP run (in order of effort):

1. **Match tar layer** — `tar --sort=name --owner=0 --group=0
   --numeric-owner` with mtimes from the file list table above;
   this should collapse the tar diff to zero.
2. **Match non-binary payload** — ship the files verbatim from the
   tag checkout; preserve the documented mtimes.
3. **Match binary provenance** — build on Raspbian Bookworm armhf,
   GCC `Raspbian 12.2.0-14+rpi1`, with `-static-libstdc++` and
   `-Wl,--build-id=none` on the link line.
4. **Match SBFspotUploadDaemon** — build from its makefile in the
   same environment; same flags.

Expected irreducible diffs (document, don't fight):

- gzip `os`, `xfl`, and populated `mtime` — unless we invest in a
  Windows-style gzip writer. Phase 3 call.
- Individual `.note.gnu.build-id` hashes on variants that have
  them — content-addressed by inputs; matching inputs gives
  matching hashes only if every byte of every object matches.
- Wall-clock mtimes on binaries in tar entries — match within a
  chosen epoch, or accept per-run drift.

## Open questions feeding into Phase 2+

- Does `snapshot.debian.org` have a **Raspbian** mirror under a
  different path? Worth a 10-minute dig before we commit to live
  archive.
- Can we reproduce the `arm × bookworm` static-libstdc++ without
  explicit flags by just matching the Raspbian Bookworm armhf
  rootfs? (Hypothesis: yes; the rootfs shapes the default.)
- For arm64 bookworm's mixed `.comment`, which object carries the
  `Raspbian 12.2.0-14+rpi1` string? If it's a bundled Boost `.a`
  or `.so` pulled from a Raspbian host, the arm64 build isn't
  fully Debian-clean either.
