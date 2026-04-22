# Phase 5d baseline — SysGCC toolchain hypothesis tested and retired

Forensic analysis (docs/phase4-baseline.md + Gemini report) identified
**SysGCC Raspberry** (sysprogs.com / gnutoolchains.com) as the most
plausible Windows cross-toolchain upstream uses. Phase 5d tested the
hypothesis end-to-end.

## What we did

1. Downloaded the free **raspberry-gcc12.2.0-r2.exe** (400 MB)
   installer from `https://sysprogs.com/getfile/2275/`.
2. Discovered the installer payload is blowfish-encrypted (binwalk
   shows `mcrypt 2.2 encrypted data, algorithm: blowfish-448`) — no
   static extractor (7z, innoextract, cabextract) can read it.
3. Ran the installer under Wine + Xvfb + xdotool GUI automation:
   click "I accept", click "Install", poll for libstdc++.a.
   Install succeeded; libstdc++.a extracted cleanly (5,811,018 B,
   valid `ar archive`, sha256 `ae93d99114b95c02f7f31fa59ec48c3fd8e6a395e67111d5f5e8e4ccbd693cf4`).
4. Substituted SysGCC's libstdc++.a into our bookworm-armhf chroot,
   rebuilt SBFspot V3.9.12, compared to upstream.

Run: `24803241239` (both extract + build jobs green). Workflow:
`.github/workflows/phase5d-probe.yml`, frozen behind
`.phase5d-probe-trigger`.

## Result

**SysGCC r2's libstdc++.a is BYTE-IDENTICAL to Raspbian's
`libstdc++-12-dev` package** (both sha `ae93d99114b9…`). The
"substitution" was effectively a no-op: we replaced Raspbian's
libstdc++.a with the same bytes.

Consequently, the residual was unchanged: **−16,384 B SBFspot,
−4,096 B daemon** vs upstream — identical to Phase 5 baseline.

## Interpretation

**SysGCC r2 repackages Raspbian's libstdc++-12-dev verbatim.** It
does not rebuild gcc; the `.a` is the stock Debian/Raspbian buildd
output.

Therefore:
- Whatever libstdc++.a upstream uses, it is **NOT from SysGCC r2**.
- Our pipeline's libstdc++.a IS the Raspbian package's output (Phase
  3.2 already proved the live chroot's bytes match a from-source
  rebuild). SysGCC r2 is in that same equivalence class.

## Retirement of the SysGCC hypothesis

We probed Sysprogs' `/getfile/` ID space (2050–2300) for the older
r1 variant. Only hit: ID 2222 = `raspberry64-gcc12.2.0.exe` (arm64
variant). **SysGCC r1 is not publicly reachable** via numbered IDs.

Combined with the r2 result (byte-equivalent to Raspbian), there is
**no publicly-accessible SysGCC variant that closes the 16 KB arm-
bookworm residual**.

Remaining live hypotheses (all require info we don't have):
- SysGCC r1 rebuilt gcc from source with specific flags, producing
  different libstdc++.a bytes than r2 does. Unreachable.
- Upstream maintains a private custom cross-gcc. Gemini's Windows-
  cross-compile forensic reasoning is consistent with this but
  doesn't pinpoint configure flags.
- Upstream installed a different Raspbian-derived toolchain (e.g.
  `abhiTronix` Linux-hosted — but `.comment` + `d:\` paths argue
  against that).

## Conclusion

**Bar 1a'' on `arm × bookworm` is unreachable from public data.**
The 16 KB residual is characterised precisely (Phase 4c addendum:
concentrated in libstdc++ `.ARM.extab` / `.text` / `.rodata`; same
function set, same personalities, different code-gen — classic
Canadian-cross reproducibility gap).

Closure path: ask the upstream maintainer for their exact toolchain
setup. Concrete ask: "For arm × bookworm, what cross-toolchain /
sysroot and gcc configure do you use when building V3.9.x?"

Moved to Phase 7 backlog. Phase 5 matrix status unchanged:
7 of 25 binaries byte-match; 18 fall in known-mechanism residual
buckets. Proceeding with Phase 5b (tarball assembly) or Phase 6
(Trixie) as next gate.
