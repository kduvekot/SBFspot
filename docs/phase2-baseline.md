# Phase 2 baseline — sqlite × arm × bookworm (V3.9.12)

First green end-to-end run of the release pipeline. Artefact lives
in GitHub Actions run `24742260942`
(`release-mvp-sqlite-arm-bookworm`, 90-day retention) and contains:

- `upstream.tar.gz` — the upstream asset.
- `sbfspot-sqlite-arm-linux-bookworm.tar.gz` — our build.
- `out/rootfs-packages.txt` — Raspbian Bookworm armhf package
  manifest from the rootfs the build ran against.
- `out/diffoscope.{html,txt}` — full diff vs upstream (~694k text
  lines).
- `out/diffoscope-{SBFspot,SBFspotUploadDaemon}.{html,txt}` —
  normalised-ELF diff per binary.

## Sha256s

| Artefact | Sha256 |
|---|---|
| upstream `sbfspot-sqlite-arm-linux-bookworm.tar.gz` | `887a393a64dc6d0924c9afa92047002b95a42395c1a2a20ecc09ca71acacabb0` |
| ours (run 24742260942) | `bb8f5defec5e08b72aed9a0317ba7c08588c1d033b75dbf0f8955fd478cbc704` |

Compressed sizes: upstream 1,601,125 B, ours 1,851,143 B. The gap
is bigger than it looks on the normalised-ELF bar because our
build is still carrying the reducible diffs listed below.

## Reducible diffs (targets for Phase 3)

1. **Text files shipped as LF, upstream is CRLF.** Sizes confirm it:
   `TagList*.txt` differ by +8,503 / +8,505 bytes (one byte per
   line for the `\r`); `CreateSQLiteDB.sql` +261; `SBFspotUpload.default.cfg`
   +57. `date_time_zonespec.csv` and `SBFspot.default.cfg` are
   already identical size — those are already CRLF-in-the-repo.
   Fix: `unix2dos` the 8 files that need it in the staging step.
2. **Tar format: ours `POSIX tar archive (GNU)`, upstream
   `POSIX tar archive` (plain ustar).** Fix: `tar --format=ustar`.
3. **Per-file mtimes truncated to whole minutes.** Upstream carries
   filesystem seconds (`13:36:02`, `13:36:03`, `17:28:09` etc.).
   Fix: bake the exact observed seconds into the mtime lookup.

## Accepted irreducible (per Decision 1a / fingerprint.md)

1. **Gzip wrapper.** Upstream: `os=00`, `xfl=04`, populated
   wall-clock `mtime` (from a Windows-side gzip writer — 7-Zip or
   similar). Ours: `os=03` (Unix), `xfl=04`, `mtime=0` (from
   `gzip -n --fast`).
2. **Wall-clock mtimes on freshly-built binaries inside the tar.**
   Upstream recorded 2025-02-22 09:49/09:50; we record build-time.
3. **Normalised-ELF compiler-version strings zeroed** via the
   normalised-ELF compare (`.comment` and `.note.gnu.build-id`
   removed).

## Raspbian toolchain drift observation (Decision 2A data point)

Upstream build used `g++ (Raspbian 12.2.0-14+rpi1) 12.2.0`. The
rootfs this run pulled from live `archive.raspbian.org` shipped
`g++ (Raspbian 12.2.0-14+rpi1+deb12u1) 12.2.0` — a post-upstream-build
security update. The visible downstream effects in normalised-ELF
sizes:

| Binary | Upstream (stripped, normalised) | Ours (run 24742260942) | Delta |
|---|---|---|---|
| `SBFspot` | 1,241,948 B | 1,131,336 B | **−110,612 B (−8.9%)** |
| `SBFspotUploadDaemon` | 885,384 B | 881,288 B | **−4,096 B (−0.5%)** |

The 110 KB SBFspot delta is the big-ticket irreducible item at the
"raw bytes" level. Almost certainly Raspbian libstdc++ (and
possibly boost) object-level differences between the two package
versions — both are statically linked into the SBFspot binary on
arm × bookworm (per fingerprint.md). `SBFspotUploadDaemon` does
not static-link libstdc++ (it uses dynamic `libstdc++.so.6` — see
fingerprint.md NEEDED table), so its delta is much smaller.

**This is exactly the drift Decision 2A exists to measure.** Phase 3
will decide whether to:
- leave the drift as reported (accept normalised-ELF diff on
  identical-input-normalised-output tests — pragmatic), or
- trigger Decision 2B fallback — bake a rootfs with the pinned
  package versions and cache it on GHCR — to collapse the drift.

## Phase 3 entry list

In priority order:

1. Fix the three reducible diffs (CRLF, tar format, mtime seconds).
   Expect collapse of most of the ~694k-line top-level diffoscope
   report.
2. Re-run, diff against upstream, record the new baseline.
3. Decide on the 110 KB binary drift: pin package versions or
   accept as irreducible with clear documentation.
