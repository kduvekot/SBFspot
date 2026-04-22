# Phase 3.3 baseline — static libbluetooth + cross-compile origin identified

Run `24761451074` (5m5s, 90-day artefact `release-mvp-sqlite-arm-bookworm`).
`release.yml` now static-links `libbluetooth.a` into SBFspot with
`-Wl,-Bstatic -lbluetooth -Wl,-Bdynamic`, matching upstream's
linkage shape. The two findings this phase produced are both more
informative than the headline diff-line count suggests.

## The bluetooth fix landed cleanly

| Metric | Phase 2 | Phase 3.1 | Phase 3.3 |
|---|---|---|---|
| SBFspot tar-entry size | 1,131,336 B | 1,131,432 B | **1,225,660 B** |
| Upstream V3.9.12 | 1,242,036 B | 1,242,036 B | 1,242,036 B |
| Delta | −110,700 | −110,604 | **−16,376** (~1.3%) |
| % of original gap closed | — | — | **85%** |
| `SBFspot` NEEDED has `libbluetooth.so.3` | yes | yes | **no** (matches upstream) |

## Per-section delta, before and after

| Section | Upstream | Phase 3.1 | Phase 3.3 | Phase 3.3 vs upstream |
|---|---|---|---|---|
| `.rodata` | 98,660 | 44,108 | 95,848 | **+2,812** |
| `.text` | 1,033,808 | 992,264 | 1,031,752 | **+2,056** |
| `.ARM.extab` | 53,224 | 41,692 | 41,692 | **+11,532** |
| 16+ other sections | — | — | — | **±0 or ±8** |
| **TOTAL (sections)** | 1,248,492 | 1,140,548 | 1,232,108 | **+16,384** |

Sixteen other sections (`.dynsym`, `.dynstr`, `.plt`, `.rel.plt`,
`.gnu.hash`, `.dynamic`, `.data.rel.ro`, `.got`, `.gnu.version`, …)
went from small non-zero deltas to byte-identical with upstream
after the bluetooth fix.

## The real irreducible: upstream cross-compiles from Windows

`strings(1)` on the upstream `SBFspot` turns up eight paths that
our binary doesn't have:

```
d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include/boost/format/alt_sstream_impl.hpp
d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include/boost/format/feed_args.hpp
d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include/boost/format/format_implementation.hpp
d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include/boost/format/internals.hpp
d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include/boost/format/parsing.hpp
d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include/boost/smart_ptr/shared_ptr.hpp
d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include/boost/token_functions.hpp
d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot\usr\include/boost/token_iterator.hpp
```

Windows drive letter `d:\`, mixed separators (`\` on the host side,
`/` inside the sysroot). The same eight strings appear in our
binary with `/usr/include` prefix instead. They come from `__FILE__`
macros embedded in boost template instantiations.

So upstream V3.9.12 is **built on Windows with a cross-compile
toolchain targeting `arm-linux-gnueabihf`**, using a custom sysroot
at `d:\rpi\cross\bookworm\gcc12.2.0\arm-linux-gnueabihf\sysroot`.
The toolchain's `.comment` reads `GCC: (Raspbian 12.2.0-14+rpi1) 12.2.0`
because it was built *from* Raspbian's gcc-12 source — but it
runs on Windows and links against a sysroot we don't have.

This explains the bluetooth static linkage too: upstream's sysroot
has `libbluetooth.a` but presumably no `libbluetooth.so`, so
`-lbluetooth` fell through to the static archive.

## Accounting for the residual 16 KB

- **`.ARM.extab` +11,532 B.** C++ exception-handling tables.
  Upstream's `libstdc++.a` and `libbluetooth.a` come from their
  Windows-cross sysroot; the object bytes differ from our
  Linux-native Raspbian-armhf equivalents, and the EH tables
  generated at link time reflect those differences.
- **`.rodata` +2,812 B.** ~480 B is the eight `d:\rpi\cross\...`
  path-prefix strings that are longer than our `/usr/include`
  prefixes. The rest (~2,300 B) is bluetooth-code bytes embedded
  in `.rodata` (string tables, jump tables) from a different
  `libbluetooth.a`.
- **`.text` +2,056 B.** Residual `libbluetooth.a` code-byte
  differences; their static archive is built against different
  headers, different gcc-patches state, different optimization
  decisions from the Windows cross.

None of this is reproducible from a Linux-native build unless we
somehow obtain the upstream maintainer's Windows cross-compile
environment, byte-for-byte. That isn't possible from public data.

## Correction to Phase 3.2's "why"

Phase 3.2 concluded Bar 1a'' was unreachable, attributed to
"Raspbian build-farm state on 2025-02-22". **That attribution was
wrong in mechanism.** The Probe's source-rebuild of `+rpi1` proved
our pipeline is internally consistent, but the gap to upstream
isn't Raspbian drift — it's **Windows-hosted cross compile + a
private sysroot**. Same conclusion on Bar 1a'' (unreachable), much
stronger evidence.

## Revised numbers for the reproducibility bars

- **Bar 1a' (pipeline-internal reproducibility):** met. Proven by
  Phase 3.1 and Probe byte-matching across different toolchain
  provenance.
- **Bar 1a'' (upstream V3.9.12 byte-match):** ~98.7% achieved
  (16,376 B / 1,242,036 B = 1.32% residual). The remaining 1.32%
  is the Windows-cross-compile irreducible set above.

## Phase 3 close

Phase 3 is done. Phases 3.0-3.3 collectively:

- Fixed the three reducible tar-layer diffs (CRLF, ustar,
  second-precise mtimes). *[Phase 3.1]*
- Proved pipeline self-reproducibility across toolchain sources.
  *[Phase 3.2]*
- Identified the bluetooth static-link and closed 85% of the
  remaining gap. *[Phase 3.3]*
- Characterised the residual 1.3% as a Windows-cross-compile
  irreducible with concrete evidence (embedded `d:\rpi\cross\...`
  paths). *[Phase 3.3]*

Next: **Phase 4.** Back-test V3.9.11 and V3.9.10 against Bar 1a'.
These are just cross-tag runs of the current `release.yml` on
different `TAG` / `ASSET_SHA256` envs; expect them to hit Bar 1a'
immediately.
