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
- Pin Debian archives to [snapshot.debian.org](http://snapshot.debian.org). Raspbian snapshot availability is unresolved — Phase 0 probes check it; fallback documented on failure.
- Runner strategy is TBD until Phase 0 closes. Working hypothesis: `ubuntu-24.04-arm` for native ARM, `ubuntu-latest` + `debootstrap` + `qemu-user-static` as fallback.

## Open decisions (sign-off required before Phase 2)

1. **Reproducibility bar.** (a) normalised-ELF match after strip + zeroing build-id / `.comment`, or (b) raw tarball `sha256` match. Default (a) unless Phase 1 shows (b) is feasible.
2. **Snapshot pinning strategy.** (i) per-release-tag date, (ii) single date near V3.9.12 (2025-02-22) for everything, (iii) live archive. Default (i).
3. **DB variant order for Phase 5.** Hypothesis: `sqlite → nosql → mariadb` (mariadb needs extra build deps).

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

Phase 0 complete (run `24731665773`, all three jobs green). Awaiting approval to advance to Phase 1.

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

## Files (planned)

- `.github/workflows/probe.yml` — throwaway feasibility probe (Phase 0; deleted once Phase 2 starts)
- `.github/workflows/release.yml` — the real pipeline (Phase 2+)
- `docs/fingerprint.md` — Phase 1 output, target spec for Phase 2
- `docs/reproducibility.md` — documented remaining diffoscope differences + rationale
- `CLAUDE.md` — this file

## Related repos

- [SBFspot/SBFspot](https://github.com/SBFspot/SBFspot) — upstream, source + release assets
- [SBFspot/sbfspot-config](https://github.com/SBFspot/sbfspot-config) — upstream installer (codename whitelist lives here)
- [kduvekot/sbfspot-rpi1-build](https://github.com/kduvekot/sbfspot-rpi1-build) — ARMv6 cross-build reference; style model for workflow + docs

## Tooling note

The Claude Code GitHub MCP in this session is scoped to `kduvekot/sbfspot` only. For everything outside that scope (upstream `SBFspot/SBFspot` source + release assets, the `kduvekot/sbfspot-rpi1-build` reference), use the `gh` CLI — the session env has `GH_TOKEN` set to a fine-grained PAT that reads those public repos fine. Install latest from `github.com/cli/cli/releases`. This path is what makes Phase 1 tarball dissection and workflow triggering tractable.
