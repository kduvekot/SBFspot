# SBFspot (kduvekot fork) — CI release-pipeline experiment

## What this repo is

A working fork of [SBFspot/SBFspot](https://github.com/SBFspot/SBFspot) used to develop a reproducible CI release pipeline that matches upstream's hand-built tarball set. The eventual goal is to contribute the pipeline back to upstream; this fork is the staging ground.

We produce **no** behavioural changes to SBFspot. The only files added here are `.github/workflows/*.yml`, pipeline-adjacent documentation, and `.gitignore` entries. C++ source is left alone so the fork can track upstream cleanly on tracked files.

## Non-goals

- **No C++ source changes.** Related C++ fixes live in separate upstream PRs: #692 (`__USE_TIME_BITS64` guard), #723 (boost `from_string`, already in V3.9.12), #740 (`-lboost_system` link). Do not mix them in here.
- **No installer work.** The [SBFspot/sbfspot-config](https://github.com/SBFspot/sbfspot-config) codename whitelist rejects Trixie; that is a separate fork, handled after the tarballs exist.
- **No ARMv6 / Pi 1 / Zero support.** Upstream does not publish ARMv6. That work lives in [kduvekot/sbfspot-rpi1-build](https://github.com/kduvekot/sbfspot-rpi1-build) and stays there.
- **No upstream master rebuilds.** Everything targets the V3.9.x release tags.

## Upstream tarball shape (what we're reproducing)

Upstream V3.9.12 (released 2025-02-22) ships **15 Linux tarballs** hand-built by the maintainer (the full release asset set is 17: 15 tarballs + 2 Windows zips we don't reproduce):

- Codenames: `buster`, `bullseye`, `bookworm`
- Architectures: `arm` (= Debian armhf under the hood), `arm64`
- DB variants: `sqlite`, `mariadb`, `nosql`
- Minus `buster × arm64` (upstream has no arm64 for buster): 3 × 2 × 3 − (1 × 1 × 3) = **15** tarballs

Naming (exact): `sbfspot-<db>-<arch>-linux-<codename>.tar.gz` — e.g. `sbfspot-sqlite-arm-linux-bookworm.tar.gz`. Note the filename uses `arm`, not `armhf`.

Makefile targets (`SBFspot/makefile`):
- `nosql` — base sources, no `-DUSE_*`, libs `pthread bluetooth boost_date_time boost_system`
- `sqlite` — adds `db_SQLite*.cpp`, `-DUSE_SQLITE`, `-lsqlite3`
- `mariadb` — adds `db_MySQL*.cpp`, `-DUSE_MYSQL`, `-lmariadbclient` (needs `libmariadbclient-dev` / `libmariadb-dev` in the rootfs)
- `LDFLAGS = -s` → binary is stripped at link time
- `CFLAGS = -c -Wall -O2 -Wno-unused-local-typedefs -Wno-psabi`

## Ground rules

- Work on a feature branch (currently `claude/add-ci-release-pipeline-QKIcZ`). Never push to `master`.
- Only add files outside `SBFspot/` proper; do not touch tracked C++ or the Windows solution.
- Expand the matrix **one dimension at a time** with a review gate between phases. Never add two dimensions in one phase.
- Reproducibility toggles from the start: `SOURCE_DATE_EPOCH`, fixed `--build-id` policy, `tar --sort=name` with fixed owner/group/mtime, `gzip -n`.
- Pin Debian archives to [snapshot.debian.org](http://snapshot.debian.org). Raspbian snapshot is non-functional (Phase 0); **arm builds need a Raspbian rootfs (Phase 1 finding) so the Raspbian pin is unavoidable** — strategy TBD per Decision 2 below.
- Runner strategy is TBD until Phase 0 closes. Working hypothesis: `ubuntu-24.04-arm` for native ARM, `ubuntu-latest` + `debootstrap` + `qemu-user-static` as fallback.

## Decisions (signed off before Phase 2, 2026-04-21)

1. **Reproducibility bar: (a) normalised-ELF match** — strip + zero build-id / `.comment`, compare byte-for-byte. Raw tarball `sha256` match (option b) is unreachable without reproducing upstream's Windows-side gzip write path (`os=00`, `xfl=04`, populated wall-clock `mtime`); revisit in Phase 3 only if time permits.
2. **Snapshot pinning:** **Debian = (i) per-release-tag date on `snapshot.debian.org`** (V3.9.12 → `20250222T000000Z`). **Raspbian = (A) live `archive.raspbian.org`**, measure drift from actual build runs; re-evaluate after Phase 2 (fallback plan is option B — bake rootfs at known-good date and cache on GHCR).
3. **DB variant order for Phase 5: sqlite (MVP) → nosql (5d) → mariadb (5e).** `nosql` is the simpler delta (no DB lib, no `SBFspotUploadDaemon`); `mariadb` pulls in `libmariadbclient-dev` and openssl/gnutls transitively.

## Phased plan

Stop for review between phases. One artefact per phase.

- **Phase 0 — Setup + probes.** This CLAUDE.md; throwaway probe workflow confirming (i) `ubuntu-24.04-arm` is aarch64, (ii) a 32-bit ARM binary built in an armhf debootstrap chroot executes natively, (iii) `snapshot.debian.org` is reachable and usable by debootstrap, (iv) Raspbian snapshot status.
- **Phase 1 — Fingerprint V3.9.12.** Read upstream makefile at the tag, download all 15 Linux release tarballs, tabulate tar / gzip / mtime / uid-gid / perms metadata, per-ELF `readelf -h/-a/-d`, `objdump -p`, build-id, `.comment`, `ldd`. Produce `fingerprint.md`.
- **Phase 2 — MVP for one combo.** Build `sqlite × arm × bookworm` against V3.9.12. `diffoscope` vs upstream asset; publish diff as artefact.
- **Phase 3 — Minimise V3.9.12 diffs** down to the irreducible set from `fingerprint.md` (at minimum: build-id hash, `.comment` compiler-version string).
- **Phase 4 — Back-test V3.9.11 and V3.9.10** on the same one combo. Gate: all three versions hit the same bar before matrix expansion.
- **Phase 5 — Incremental matrix.** 5a bullseye, 5b buster, 5c arm64, 5d nosql, 5e mariadb. Introduce `actions/cache` on the debootstrap rootfs starting 5a (first repeat codename).
- **Phase 6 — Trixie.** 6 new cells (`trixie × {arm,arm64} × {sqlite,nosql,mariadb}`). No upstream comparison (no tarballs exist); verify binaries run in a fresh trixie rootfs and pass smoke checks.
- **Phase 7 — Upstream contribution.** Separate session: open an issue at `SBFspot/SBFspot` with links to the green workflow + diffoscope report, offer a PR.

## Current phase

Phase 2 complete. First green end-to-end run at `.github/workflows/release.yml`, frozen behind `.release-trigger`. Run `24742260942` (artefact `release-mvp-sqlite-arm-bookworm`, 90-day retention) holds the upstream asset, our build, the rootfs package manifest, and full diffoscope output. Baseline + diff observations recorded in `docs/phase2-baseline.md`. Phase 1 CI audit at `.github/workflows/fingerprint.yml` (frozen behind `.fingerprint-trigger`, last run `24737026571` green). Phase 0 probe workflow kept and frozen behind `.github/workflows/.probe-trigger`. Three decisions signed off 2026-04-21. **Advancing to Phase 3** — minimise diffs down to the irreducible set.

### Phase 0 findings

- **`ubuntu-24.04-arm` is aarch64.** Azure-hosted VM, 4 cores, Ampere Neoverse V1 (`CPU part 0xd49`), ARMv8-A. Kernel 6.14.0-1017-azure.
- **AArch32 native exec works on that silicon.** An armhf `gcc` built hello-world ran inside the chroot without qemu and printed `hello from armhf`. `file` reports: `ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, BuildID[sha1]=…, not stripped`. **Implication: the real pipeline can build armhf natively on `ubuntu-24.04-arm` — no qemu on the common path.**
- **`snapshot.debian.org/archive/debian/20250222T000000Z` works as a `debootstrap` mirror** for armhf bookworm, both on the native-aarch64 path and via `--foreign` + `qemu-arm-static` + `--second-stage` (control job).
- **`snapshot.raspbian.org` is not a functional snapshot service.** Root page (`/`) returns 200 but `/archive/`, `/archive/raspbian/`, and date URLs in either `YYYYMMDDTHHMMSSZ` or `YYYYMMDD` form return 404. Live `archive.raspbian.org` and `archive.raspberrypi.org` are reachable. Since our critical path is Debian (upstream's armhf tarballs target Debian armhf = ARMv7+), this is **documented risk, not a blocker**. If we ever need Raspbian-specific builds we'll pin against the live archive and accept the reproducibility hit.
- **qemu-user-static fallback path works too**, proven in the control job; available as a backup for any arch/codename combo that turns out not to exec natively.

### Implications for the matrix

- `arm` (armhf) builds: `ubuntu-24.04-arm` + `debootstrap --arch=armhf` + native AArch32 exec.
- `arm64` builds: `ubuntu-24.04-arm` native (arch match); still debootstrap per codename for correct glibc/boost.
- qemu stays in the toolbox as a fallback, not the default.

### Phase 1 findings

Full detail in `docs/fingerprint.md`. Headline points that change strategy:

- **Source state is clean.** `SBFspot/makefile` at this fork matches V3.9.12 byte-for-byte; no source divergence blocks the rebuild.
- **Arm builds are Raspbian, arm64 builds are Debian.** Binary `.comment` strings: arm variants all say `GCC: (Raspbian X+rpi1) X`; arm64 variants say `GCC: (Debian X) X`. Bookworm arm64 carries **both** strings — one object (likely a bundled Boost static lib) was compiled on Raspbian and linked into a Debian build. This invalidates a Debian-only pipeline: arm reproducibility needs a Raspbian rootfs.
- **Upstream does not normalise tarball gzip.** OS byte is `00` (Windows / FAT), `xfl=04` (`gzip --fast`), and `mtime` is the wall-clock time of compression, not zeroed — strongly suggests a Windows-side gzip writer (7-Zip or similar). **No `SOURCE_DATE_EPOCH` in upstream's flow.** Per-file mtimes inside the tar are preserved filesystem mtimes from the maintainer's checkout (ranging 2021-01-17 through 2024-06-15), not normalised. To match raw bytes we'd need either a custom Windows-style gzip writer or a per-file mtime lookup table. Both are punted to Phase 3.
- **Tar entry layout is normalisable.** Order is lexicographic (`tar --sort=name`), uid/gid=0 everywhere (`--owner=0 --group=0 --numeric-owner`). We match this trivially.
- **All non-binary payload is byte-identical across all 15 tarballs.** One source checkout, no per-variant post-processing.
- **`arm × bookworm` is 2–3× the size of its siblings** because it statically links libstdc++ (no `libstdc++.so.6` in `NEEDED`) and has no `.note.gnu.build-id`. Both point at Raspbian Bookworm armhf linker/toolchain defaults rather than explicit flags — but the pipeline can also force them via `-static-libstdc++` and `-Wl,--build-id=none`.
- **Second binary: `SBFspotUploadDaemon`** — built from `SBFspotUploadDaemon/makefile` (pulls sources from `../SBFspot` and `../SBFspotUploadCommon`), present only in `sqlite` and `mariadb` tarballs, `NEEDED` adds `libcurl` plus the DB lib.

### Phase 2 MVP target

Build `sqlite × arm × bookworm` against V3.9.12. Upstream asset: `sbfspot-sqlite-arm-linux-bookworm.tar.gz` (sha256 `887a393a64dc6d0924c9afa92047002b95a42395c1a2a20ecc09ca71acacabb0`). First-pass diffoscope goals, in effort order:

1. Match tar layer (sort, uid/gid, per-file mtime lookup).
2. Ship non-binary payload verbatim from the tag checkout with preserved mtimes.
3. Build both binaries on a Raspbian Bookworm armhf rootfs with `gcc 12.2.0-14+rpi1`, `-static-libstdc++`, and `-Wl,--build-id=none` for the arm-bookworm combo.

Accept as irreducible (document, don't fight): gzip `os`/`xfl`/populated `mtime`, individual `.note.gnu.build-id` hashes on variants that still emit them, wall-clock mtimes on freshly-built binary entries inside the tar.

## Files (planned)

- `.github/workflows/probe.yml` — Phase 0 feasibility probe. Kept as reference, frozen behind `.github/workflows/.probe-trigger` so it does not auto-run.
- `.github/workflows/fingerprint.yml` — Phase 1 CI audit (planned next increment). Reproduces the local dissection on GitHub Actions so `docs/fingerprint.md` is auditable by third parties.
- `.github/workflows/release.yml` — the release pipeline (Phase 2 MVP in place, Phase 3 is iterating it). Frozen behind `.release-trigger`.
- `docs/fingerprint.md` — Phase 1 output, target spec for Phase 2.
- `docs/phase2-baseline.md` — Phase 2 end state: first-run artefact pin, reducible-vs-irreducible diff breakdown, Phase 3 entry list.
- `docs/reproducibility.md` — final remaining diffoscope differences + rationale (Phase 3 exit).
- `CLAUDE.md` — this file.

## Related repos

- [SBFspot/SBFspot](https://github.com/SBFspot/SBFspot) — upstream, source + release assets
- [SBFspot/sbfspot-config](https://github.com/SBFspot/sbfspot-config) — upstream installer (codename whitelist lives here)
- [kduvekot/sbfspot-rpi1-build](https://github.com/kduvekot/sbfspot-rpi1-build) — ARMv6 cross-build reference; style model for workflow + docs

## Tooling note

The Claude Code GitHub MCP in this session is scoped to `kduvekot/sbfspot` only. For everything outside that scope (upstream `SBFspot/SBFspot` source + release assets, the `kduvekot/sbfspot-rpi1-build` reference), use the `gh` CLI — the session env has `GH_TOKEN` set to a fine-grained PAT that reads those public repos fine. Install latest from `github.com/cli/cli/releases`. This path is what makes Phase 1 tarball dissection and workflow triggering tractable.
