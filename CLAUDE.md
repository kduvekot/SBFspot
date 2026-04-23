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

Phase 5e complete — modern hardened pipeline, 15 cells, reproducible tarballs. Active workflow: `.github/workflows/phase5e-probe.yml`, full rationale in companion `.github/workflows/phase5e-probe.md`. The pivot off chasing upstream byte-match (documented through Phase 5d) opened up modernising the pipeline: PIE + full RELRO + BIND_NOW + `_FORTIFY_SOURCE=2` + `-fstack-protector-strong` + `-D_GLIBCXX_ASSERTIONS` + `--build-id=sha1`, with PIC libbluetooth.a rebuilt per cell from distro bluez source so static-linking into PIE emits no TEXTREL on armhf. Runtime-dep surface matches upstream exactly (libbluetooth not in NEEDED; libmariadb dynamic). All 15 cells produce reproducible `sbfspot-<db>-<arch>-linux-<codename>.tar.gz` artefacts with deterministic tar (ustar, sorted, uid/gid 0, `--mtime=@$SOURCE_DATE_EPOCH` derived from V3.9.12 tag commit) + deterministic gzip (`gzip -n`). Run-to-run reproducibility proven: runs `24840329834` and `24840705192` produced byte-identical SHA256s on all 15 tarballs. Smoke-test step (binary exec + readelf hardening verification) passes on every cell. Known limitation: arm cells pull from live Raspbian archive (no working snapshot service per Phase 0), so long-term reproducibility holds modulo Raspbian security updates; arm64 cells fully snapshot-pinned. Phase 5 (byte-match matrix run `24789691360`) preserved as frozen probe `phase5-matrix.yml` for reference; its residual analysis lives in `docs/phase5-baseline.md`.

Phase 5 matrix findings for context: 7 of 25 binaries were byte-identical with upstream (all four `*-bullseye × {sqlite,nosql}` cells clean, plus `sqlite-arm64-bookworm` daemon). Remaining 18 binaries fell into documented categories: 6 "size-match, content differs" (buster sysroot drift + arm64-bookworm dual-`.comment`); 3 small residuals < 100 B (NEEDED ordering); 5 medium 4–8 KB (mariadb `libmariadbclient.a` byte drift); 4 arm-bookworm bookworm-specific residuals (−12–16 KB static-libstdc++ characterised in Phase 4). Anomaly: `nosql-arm64-bookworm` showed −65 KB file-layout padding (LOAD-segment alignment), not content drift. Phase 5 surfaced 2 real pipeline bugs since fixed (sqlite-arm-buster missing libgnutls28-dev; mariadb-arm-buster needs gnutls not openssl). Per-combo LDFLAGS/pkgs/prefix/mirror consolidated in `docs/phase5-baseline.md`.

Phase 4 summary for context: version back-test gate PASSED. Run `24766226453` (3-cell matrix `.github/workflows/phase4-probe.yml`, frozen behind `.phase4-probe-trigger`) built SBFspot V3.9.10, V3.9.11, V3.9.12 against the current Phase 3.3 + 3.4 pipeline on `arm × bookworm × sqlite`. **Residual vs upstream is identical across all three tags**: SBFspot `−16,384 B`, SBFspotUploadDaemon `−4,096 B`. Confirms the arm-bookworm irreducible is a property of upstream's toolchain setup, not a V3.9.12-specific quirk. Bar 1a' (pipeline-internal reproducibility across source tags) met: our daemon byte-matches across V3.9.10↔V3.9.11 where the effective source didn't change; upstream's daemon byte-matches across V3.9.11↔V3.9.12 (their different change-point, but same determinism property). Full matrix + per-binary sha256 table in `docs/phase4-baseline.md`, including a post-Phase-4 addendum that narrows the arm-bookworm ~16 KB residual to a specific mechanism: same function set as upstream (2,278 vs 2,279 functions), same `.ARM.attributes` (armv6/VFPv2), same `.comment` label (`Raspbian 12.2.0-14+rpi1`) — but **different code bytes** in the libstdc++ region (byte-level compare on a "fat" function at upstream `0x3794c` shows only 20% match in the first 256 bytes; prologue matches, then instruction ordering and register allocation diverge). Combined with binary mtimes across V3.9.10 (2024-06-12), V3.9.11 (2024-06-17), V3.9.12 (2025-02-22) that all carry the identical residual, this points to: upstream's gcc is a Canadian-cross build (x86_64-w64-mingw32 host → armhf target) installed once pre-June 2024 and reused across the 8-month span, whose bundled libstdc++.a has different object bytes than any Debian/Raspbian Linux-native libstdc++.a would. Phase 4c probed three plausible libstdc++ rebuild flags (~5h total CI): `-fasynchronous-unwind-tables` closes only 4 KB and grows the wrong section (`.ARM.exidx` not `.ARM.extab`); `-D_GLIBCXX_ASSERTIONS` overshoots by +24 KB growing `.text`/`.rodata`; `-fnon-call-exceptions` overshoots by +61 KB growing `.ARM.extab` 4× too much. None produces upstream's exact shape — residual is not reachable with a single well-known gcc flag. **Accepted as irreducible** for `arm × bookworm`: 1.3% residual on one combo, 9 combos that byte-match on the current pipeline. Closure path requires asking upstream maintainer for their specific Canadian-cross gcc configure (Phase 7 topic). Phase 4's version-axis outcome is orthogonal to the DB/codename/arch axes from Phase 3.5 — Phase 5 matrix expansion proceeded keyed on V3.9.12 as the release target. **Next gate: Phase 5b (per-variant tarball assembly) or Phase 6 (Trixie).**

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

- `.github/workflows/README.md` — folder index: active workflow + frozen probes, with pointers to companion docs. Conventions (frozen-probe trigger gate, 15-cell matrix shape, action pins, ARM runner).
- `.github/workflows/phase5e-probe.yml` — **active** workflow: modern hardened 15-cell pipeline producing reproducible tarballs. Frozen behind `.phase5e-probe-trigger`.
- `.github/workflows/phase5e-probe.md` — companion rationale doc (10 sections: intro, shape, step-by-step, compile flags, link flags, libbluetooth rebuild, per-cell matrix, dropped options, reproducibility guarantees, glossary). "Jip-en-Janneke" style: accessible to someone who's never cross-compiled or touched GitHub Actions, but goes deep enough to teach.
- `.github/workflows/probe.yml` — Phase 0 feasibility probe. Kept as reference, frozen behind `.github/workflows/.probe-trigger` so it does not auto-run.
- `.github/workflows/fingerprint.yml` — Phase 1 CI audit (planned next increment). Reproduces the local dissection on GitHub Actions so `docs/fingerprint.md` is auditable by third parties.
- `.github/workflows/release.yml` — the release pipeline (Phase 2 MVP in place, Phase 3 iterated it). Frozen behind `.release-trigger`.
- `.github/workflows/toolchain-probe.yml` — Phase 3.2 one-off: source-rebuild of `gcc-12_12.2.0-14+rpi1` and SBFspot build against the pinned toolchain. Frozen behind `.toolchain-probe-trigger`. Last green `24746308004`.
- `docs/fingerprint.md` — Phase 1 output, target spec for Phase 2.
- `docs/phase2-baseline.md` — Phase 2 end state: first-run artefact pin, reducible-vs-irreducible diff breakdown, Phase 3 entry list. Drift attribution corrected in-place after Phase 3.2.
- `docs/phase3.1-baseline.md` — Phase 3.1 end state: CRLF + tar ustar + second-precise mtimes applied, tar-entry parity achieved. Drift attribution corrected in-place after Phase 3.2.
- `docs/phase3.2-baseline.md` — Phase 3.2 end state: source-rebuild probe, pipeline self-reproducibility proven. Mechanism attribution ("build-farm state") retracted by Phase 3.3; see in-place correction.
- `docs/phase3.3-baseline.md` — Phase 3.3 end state: libbluetooth static-link closes 85% of the 110 KB delta; per-section diff + embedded `d:\rpi\cross\...` strings identify upstream as a Windows cross-compile. Applies to `arm × bookworm` only.
- `docs/phase3.4-baseline.md` — Phase 3.4 end state: `-fmacro-prefix-map` wrapper achieves full Bar 1a'' byte-match on `arm × bullseye × sqlite`. Identifies upstream's per-combo prefix scheme; scopes Windows-CI out-of-scope.
- `docs/phase3.5-baseline.md` — Phase 3.5 end state: cross-combo generalisation probe across DB/codename/arch axes. Revises Phase 3.4's 14-of-15 claim: actually 9 of 15 are clean-match achievable; all 6 bookworm combos carry an additional cross-toolchain irreducible (~10-16 KB) from upstream's private pre-built static libs. Per-combo LDFLAGS table for Phase 5.
- `docs/phase4-baseline.md` — Phase 4 end state: version back-test V3.9.10/11/12 on arm-bookworm-sqlite. Residual identical across all three tags (−16,384 B SBFspot, −4,096 B daemon). Gate passed; version axis orthogonal to DB/codename/arch. Post-Phase-4 addendum with three libstdc++ rebuild probes (Phase 4c) characterising the irreducible mechanism.
- `docs/phase5-baseline.md` — Phase 5 end state: 15-combo matrix residual measurement. 7 of 25 binaries byte-identical; 18 with documented residual categories (sysroot drift, NEEDED ordering, bookworm-specific irreducibles, file-layout padding).
- `docs/phase5d-baseline.md` — Phase 5d end state: SysGCC-toolchain hypothesis tested end-to-end and retired. SysGCC r2's libstdc++.a is byte-identical to Raspbian's package (no-op substitution). SysGCC r1 not publicly reachable. 16 KB arm-bookworm residual unreachable from public data; Phase 7 (upstream-contact) topic.
- `docs/reproducibility.md` — final remaining diffoscope differences + rationale (Phase 3 exit).
- `CLAUDE.md` — this file.

## Related repos

- [SBFspot/SBFspot](https://github.com/SBFspot/SBFspot) — upstream, source + release assets
- [SBFspot/sbfspot-config](https://github.com/SBFspot/sbfspot-config) — upstream installer (codename whitelist lives here)
- [kduvekot/sbfspot-rpi1-build](https://github.com/kduvekot/sbfspot-rpi1-build) — ARMv6 cross-build reference; style model for workflow + docs

## Tooling note

The Claude Code GitHub MCP in this session is scoped to `kduvekot/sbfspot` only. For everything outside that scope (upstream `SBFspot/SBFspot` source + release assets, the `kduvekot/sbfspot-rpi1-build` reference), use the `gh` CLI — the session env has `GH_TOKEN` set to a fine-grained PAT that reads those public repos fine. Install latest from `github.com/cli/cli/releases`. This path is what makes Phase 1 tarball dissection and workflow triggering tractable.
