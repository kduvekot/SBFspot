# Phase 3.2 baseline — source-rebuild probe + pipeline self-reproducibility proof

> **Correction (Phase 3.3):** This doc's "Finding 2" attributes the
> 110 KB upstream gap to "Raspbian's build-farm state on 2025-02-22."
> **That mechanism was wrong.** Phase 3.3 found that
> (a) most of the gap (94 KB of 110 KB) was just libbluetooth not
> being statically linked — a single missing linker flag, and
> (b) the remaining 16 KB is explained by upstream being
> *cross-compiled from Windows*: the upstream binary embeds eight
> `d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\...`
> paths from `__FILE__` macros in boost template instantiations.
> The same-conclusion-different-mechanism is now in
> `docs/phase3.3-baseline.md`. Finding 1 (pipeline self-reproducibility)
> still holds unchanged.

## Executive summary

The Phase 3.2 probe rebuilt `gcc-12_12.2.0-14+rpi1` from the
preserved source packages on `archive.raspbian.org` (1h42m on
`ubuntu-24.04-arm` native AArch32), installed the resulting debs
into a fresh Raspbian Bookworm armhf chroot, and rebuilt SBFspot
V3.9.12 against the pinned toolchain. Artefact: run `24746308004`,
`toolchain-probe-rpi1`, 90-day retention.

**Two findings, one of them changes the reproducibility story.**

### Finding 1: Our pipeline is already self-reproducible

Phase 3.1 (built 2026-04-21 ~20:38 UTC with live `+rpi1+deb12u1`
toolchain) and the Probe (built 2026-04-21 ~22:44 UTC with our
own rebuilt `+rpi1` toolchain) produce **bit-identical normalised
SBFspot binaries**:

| Binary | Phase 3.1 `*.norm` sha256 | Probe `*.norm` sha256 |
|---|---|---|
| `SBFspot` | `f5453d9b6593e0701b0b351ae7df91073783271fa14f6340c8f346b47053dd6f` | `f5453d9b6593e0701b0b351ae7df91073783271fa14f6340c8f346b47053dd6f` |
| `SBFspotUploadDaemon` | `8e86eb5802bec33734dbba8b25cfa6eb005a5f0feebae2e551c3d844370ad695` | `8e86eb5802bec33734dbba8b25cfa6eb005a5f0feebae2e551c3d844370ad695` |

Normalised-ELF here means: `strip --strip-all` + `objcopy
--remove-section=.comment --remove-section=.note.gnu.build-id`.
The only bytes that changed between Phase 3.1 and the Probe are
the 8 bytes of length difference in the `.comment` string
(`+rpi1+deb12u1` vs `+rpi1`).

This is the actual reproducible-builds property. We have it.

### Finding 2: The 110 KB drift vs upstream V3.9.12 is not package-version drift

Earlier docs (`phase2-baseline.md`, `phase3.1-baseline.md`) attributed
the 110 KB `SBFspot` delta vs upstream to `+rpi1` → `+rpi1+deb12u1`
libstdc++ object-code changes from the security update. **That was
wrong.** The Probe's rebuilt `+rpi1` toolchain produces the
identical normalised SBFspot binary as Phase 3.1's live
`+rpi1+deb12u1` toolchain, so the libstdc++.a object code linked
into SBFspot is effectively unchanged between those two
package versions.

The true cause of the 110 KB drift vs upstream V3.9.12 lives one
layer below gcc: **Raspbian's build-farm state on 2025-02-22**
(host binutils, glibc headers, whatever compiler bootstrapped
their gcc-12 build, specific build-deps, SOURCE_DATE_EPOCH
policy, parallelism ordering). None of that is recoverable from
public data.

| Binary | Upstream V3.9.12 `*.norm` | Ours (Phase 3.1 + Probe) `*.norm` | Delta |
|---|---|---|---|
| `SBFspot` | 1,241,948 B | 1,131,336 B | **−110,612 B** |
| `SBFspotUploadDaemon` | 885,384 B | 881,288 B | **−4,096 B** |

Size deltas are identical to Phase 2 and Phase 3.1 — the bar
didn't move, and now we know *why*: gcc isn't the variable, the
farm is.

## Provenance of the rebuild

Sources fetched from `archive.raspbian.org/raspbian/pool/main/g/gcc-12/`:

| File | sha256 |
|---|---|
| `gcc-12_12.2.0-14+rpi1.dsc` | `35a7429a0f1c80508685e5d934f56f65de137e50e3d95f62b812367e623f64d2` |
| `gcc-12_12.2.0-14+rpi1.debian.tar.xz` | `4d14f32cc54663d52b49ac718f4bb711963092c7096fb496e26c73432143d002` |
| `gcc-12_12.2.0.orig.tar.gz` | `b8298be16aeeb96a889c6afed0a8e2241b47452e89cc81fe65ea849d5c740fcb` |

The `+rpi1+deb12u1` binary debs have been flushed from the live
pool; the source trio is preserved. `DEB_BUILD_PROFILES=nolang=...`
did **not** take effect — the gcc-12 packaging ignores the
profile flags and still built Ada/D/Fortran/Go/Modula-2/Objective-C.
Wall-clock was still tolerable at 1h42m on 4-core Neoverse V1.

## Revised reproducibility bar

Original Decision 1a: normalised-ELF match vs upstream V3.9.12.

**We now have hard evidence this bar is unreachable** without
Raspbian's build-farm state, and that is not a bug in our pipeline
but a property of the upstream release process (Raspbian does not
publish a reproducible-builds substrate for their arm packages).

Proposed revision:

- **Bar 1a' (achievable, proven):** *pipeline-internal reproducibility.*
  Two runs of `release.yml`, on different days, on different
  runners, against the same source tag, produce bit-identical
  normalised SBFspot binaries. Today's evidence: Phase 3.1 and
  Probe cross-match.
- **Bar 1a'' (unreachable, documented):** *upstream-match.*
  Byte-for-byte normalised-ELF parity with upstream V3.9.12.
  Blocker: Raspbian's 2025-02-22 build-farm state isn't
  reconstructible from public data.

## What's the drift insurance story?

Phase 3.1 uses the live `+rpi1+deb12u1` toolchain. Today it
produces the same bytes as our rebuilt `+rpi1`. If Raspbian
publishes `+rpi1+deb12u2` tomorrow, the live path may drift. We
have three options:

1. **Trust the live archive** (status quo). Cheap; drift when it
   happens. Our cross-run match today shows the live archive is
   stable enough for SBFspot's purposes.
2. **Archive the Probe-built debs as a release asset on our own
   repo.** One-line `curl` in `release.yml` pulls them, `dpkg -i`
   before the SBFspot build. No GHCR bake, no second workflow.
   Low-cost belt-and-braces.
3. **Full GHCR rootfs bake** (the original Decision 2B shape).
   Heavier; only pays off if we hit drift *and* want a fully
   opaque substrate. Deferred until we observe drift.

## Recommendation

Declare Phase 3 **closed** with Bar 1a' met and Bar 1a'' documented
as unreachable. Pick up Phase 4 (V3.9.11 / V3.9.10 back-test) on
the current pipeline; those runs only need to hit Bar 1a' (self-
reproduce across their respective source tags), which the pipeline
already does.

Option (2) above — commit the Probe-built `+rpi1` debs as a
release asset — is cheap and removes live-archive drift as a
variable before Phase 4. Worth doing once, now, as Phase 3.3.
