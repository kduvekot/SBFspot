# Phase 4 baseline — version back-test across V3.9.10/11/12

## Gate

*"Back-test V3.9.11 and V3.9.10 on the same one combo. Gate: all
three versions hit the same bar before matrix expansion."*

Probe: `.github/workflows/phase4-probe.yml`, run `24766226453`,
3-cell matrix (one per tag), all on `arm × bookworm × sqlite`
using the Phase 3.3 pipeline + Phase 3.4 prefix-map wrapper.

## Result — gate PASSED

### Residual identical across all three tags

| Binary | V3.9.10 (upstream → ours) | V3.9.11 | V3.9.12 |
|---|---|---|---|
| `SBFspot.norm` size | 1,241,948 → 1,225,564 | 1,241,948 → 1,225,564 | 1,241,948 → 1,225,564 |
| SBFspot residual | **−16,384 B** | **−16,384 B** | **−16,384 B** |
| `SBFspotUploadDaemon.norm` size | 885,384 → 881,288 | 885,384 → 881,288 | 885,384 → 881,288 |
| Daemon residual | **−4,096 B** | **−4,096 B** | **−4,096 B** |

The exact residual is identical across all three tags. This
confirms what Phase 3.3 hypothesised from V3.9.12 alone: the
~16 KB arm-bookworm irreducible is a property of upstream's
toolchain setup (private Raspbian-compiled-for-armhf
`libstdc++.a`) that's stable across V3.9.10 → V3.9.12, not a
one-off version-specific quirk.

### Cross-tag sha256 matrix

| Side | Binary | V3.9.10 | V3.9.11 | V3.9.12 |
|---|---|---|---|---|
| upstream | `SBFspot.norm` | `3e54edf5…` | `a03a6d06…` | `35b72928…` |
| upstream | `Daemon.norm` | `27190601…` | **`f5eb3b8e…`** | **`f5eb3b8e…`** |
| ours | `SBFspot.norm` | `a9e4828c…` | `c00e7bb0…` | `5e46a607…` |
| ours | `Daemon.norm` | **`8ab567d4…`** | **`8ab567d4…`** | `8e86eb58…` |

Two separate observations from this matrix:

1. **Our pipeline is deterministic at each tag.** Same source tag
   + same pipeline = same bytes. The daemon sha cross-matches
   between V3.9.10 and V3.9.11 (`8ab567d4…`), which is our
   pipeline reproducing byte-identically when the daemon's
   effective source didn't change. Bar 1a' (pipeline-internal
   reproducibility across tags): **met**.

2. **Upstream is also deterministic.** Their V3.9.11 and V3.9.12
   daemon binaries are byte-identical (`f5eb3b8e…`). Whatever the
   maintainer's Windows cross-compile process is, it does reproduce
   bit-for-bit when source is unchanged. This is a Phase 3.2-style
   reproducibility claim for upstream — and the evidence, at least
   for the daemon, looks strong.

3. **Our pairing offset.** Our byte-identical pair is 3.9.10-3.9.11,
   upstream's is 3.9.11-3.9.12. That's a small but real finding:
   the daemon binary's transitive source dependencies (objects
   from `../SBFspot/db_SQLite.cpp` etc.) have their change-points
   at different tag boundaries in our rebuild than in upstream's.
   Could be something as simple as a header-version difference in
   a transitively-included file — don't have a one-line fix or a
   clear cause.

### Wall-clock

~5 min wall-clock, 3 jobs in parallel on `ubuntu-24.04-arm`. Each
cell runs the full debootstrap + build + diffoscope. No caching
yet.

## Post-Phase 4 addendum: where exactly is the 16 KB?

Phase 3.3 characterised the arm-bookworm residual as "static
libstdc++ drift" without pinpointing exactly which library code
or which subsystem contributes. With V3.9.12 binaries from Phase
4 in hand, a deeper probe:

### 1. Section-level breakdown (unchanged from Phase 3.3)

| Section | upstream | ours | delta |
|---|---|---|---|
| `.ARM.extab` | 53,224 | 41,692 | **+11,532** (71% of delta) |
| `.rodata` | 98,660 | 95,848 | **+2,812** (17%) |
| `.text` | 1,033,808 | 1,031,752 | **+2,056** (13%) |

### 2. Function-count check

- upstream: 2,278 functions in `.ARM.exidx` (1,036 with full `.extab` entries, 817 inline-PR0, 15 inline-PR1, 425 cantunwind)
- ours:     2,279 functions in `.ARM.exidx` (1,039 with full `.extab` entries, 813 inline-PR0, 15 inline-PR1, 427 cantunwind)

Function counts are essentially identical (±3). **Not a "more
code" problem** — same functions on both sides.

### 3. Personality-routine distribution

| Model | upstream | ours | delta |
|---|---|---|---|
| PR0 compact | 817 | 813 | +4 |
| PR1 compact | 15 | 15 | 0 |
| PR2 compact | 0 | 0 | 0 |
| Generic model (extab-resident) | 1,021 | 1,024 | −3 |

Same C++ exception model, same personality routines. **Not an
`-fexceptions` vs `-fno-exceptions` / EHABI-vs-sjlj kind of
difference.**

### 4. Per-function extab-size distribution

Average extab bytes per extab-ref function:

- upstream: 53,224 / 1,036 ≈ **51.4 B/function**
- ours:     41,692 / 1,039 ≈ **40.1 B/function**
- delta:    **+11.3 B/function** × ~1,000 functions ≈ +11.5 KB

**Same set of functions; each upstream function carries ~11 more
bytes of unwind metadata on average.**

### 5. Address-range concentration

Bucketed extab bytes by 64 KB `.text` virtual-address range:

| `.text` vaddr range | upstream | ours | delta |
|---|---|---|---|
| `0x10000–0x1ffff` | 4,188 | 2,656 | **+1,532** |
| `0x20000–0x2ffff` | 4,868 | 2,556 | **+2,312** |
| `0x30000–0x3ffff` | 4,856 | 2,500 | **+2,356** |
| `0x40000–0x4ffff` | 5,532 | 2,904 | **+2,628** |
| `0x50000–0x5ffff` | 5,168 | 2,488 | **+2,680** |
| remainder (app code) | ~ equal | ~ equal | ~ 0 |

**The entire 11.5 KB delta concentrates in the first ~320 KB of
`.text`**, which is where the linker places libstdc++ static
content. App code region (from ~0x60000 onward) has ~zero extab
delta. **This is purely a `libstdc++.a` issue.**

### 6. Root cause

Same libstdc++ functions, same personality-routine mix, same
exception model — upstream's per-function unwind tables just
carry more bytes than ours. That points to a gcc-build-time
configure choice for libstdc++ itself, embedded in their `.a`
file. Candidates (all speculative — we don't have upstream's
crosstool-ng config):

- `--enable-libstdcxx-assertions` (adds runtime check wrappers
  with cleanup handlers → more unwind ops per function)
- `--enable-libstdcxx-backtrace` (gcc-12.1+; adds stacktrace
  support with extra cleanup in exception paths)
- Different `--with-default-libstdcxx-abi` / `--enable-clocale`
- Different `-fasynchronous-unwind-tables` vs
  `-funwind-tables` for libstdc++'s own compile

None of these are recoverable from public data — they live in
upstream's Windows-hosted cross-toolchain configure. To close the
11.5 KB, we'd need the maintainer's configure flags or the `.a`
file itself.

### Conclusion

The "~16 KB irreducible on arm × bookworm" now has a specific
root cause: **upstream's libstdc++.a emits ~11 bytes more per
function in its `.ARM.extab` unwind tables than any public
Raspbian-gcc-built libstdc++.a does.** That's a gcc-build-of-
libstdc++ configure-time choice we don't have access to. The
remaining ~4.5 KB (`.rodata` + `.text`) likely follows from the
same configure difference — slightly different LSDA tables
referenced from the extab entries, plus some minor codegen
knock-ons.

Bar 1a'' byte-match stays unreachable on `arm × bookworm`, but
we now know **why** with precision.

## What this tells us going into Phase 5

- The pipeline is **structurally invariant** across source
  versions — shape of output matches shape of upstream, and the
  irreducible is constant-size.
- **Bar 1a'' at its best-achievable level (~16 KB residual on
  bookworm-arm, 0 elsewhere) is a stable property of the pipeline
  and upstream's build setup combined**, not a V3.9.12-specific
  outcome.
- Adding source-version as an axis (Phase 4) doesn't change the
  matrix shape; it's orthogonal to the DB/codename/arch axes
  probed in Phase 3.5.

## Phase 5 hand-off

With Phase 4 cleared:

1. Phase 5a—e can use the same three axes from Phase 3.5 and
   ignore the version axis (pinned to V3.9.12 for the release
   pipeline; back-testing older tags is a one-off QA probe, not
   a release target).
2. The per-combo LDFLAGS / prefix-map / package-list table from
   `docs/phase3.5-baseline.md` is what a Phase 5 matrix
   expansion of `release.yml` needs.
3. The documented residuals on `*-bookworm` combos (~16 KB arm,
   ~10 KB arm64) are acceptable outcomes of Phase 5 and not bugs
   to chase further without access to upstream's private sysroot.

## Files

- `.github/workflows/phase4-probe.yml` — one-off probe, frozen
  behind `.phase4-probe-trigger`.
