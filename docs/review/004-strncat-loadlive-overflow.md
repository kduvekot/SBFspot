# Finding 4: `strncat` size-argument misuse causes adjacent-buffer corruption in `-loadlive` path

## Metadata

| | |
|---|---|
| **Severity** | MEDIUM-HIGH |
| **CWE** | [CWE-787](https://cwe.mitre.org/data/definitions/787.html) — Out-of-bounds Write; secondary [CWE-676](https://cwe.mitre.org/data/definitions/676.html) — Use of Potentially Dangerous Function |
| **CVSS 3.1 Vector** | `AV:L/AC:H/PR:H/UI:N/S:U/C:L/I:L/A:L` — **3.4** for the local-attacker-with-config-write case; raise to `AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` — **7.8** if `SBFspot.cfg` is operator-writable but used in a multi-user context |
| **Affected version** | 3.10.0 (commit `b77f650`, branch `master`) |
| **Affected build** | Any build invoked with the `-loadlive` flag |
| **Confidence** | 8/10 — bug is concrete and deterministic; severity depends on whether the config file is treated as trusted operator input |
| **Verification** | Static analysis only |
| **Notes** | Reported by one of the two review passes only — not an independent confirmation, but the bug is mechanical and unambiguous |

## Summary

`GetConfig()` in `SBFspot/SBFspot.cpp` calls
`strncat(cfg->outputPath, "/LoadLive", sizeof(cfg->outputPath))`. The third
argument to `strncat` is the maximum number of bytes to copy from the
**source**, not the size of the destination buffer. Because `cfg->outputPath`
is a fixed-size `char[MAX_PATH]` populated from the `OutputPath` config
value via `strncpy(..., sizeof(cfg->outputPath) - 1)`, it can already hold
`MAX_PATH − 1` bytes plus NUL — and `strncat` then unconditionally appends
the 9-byte literal `"/LoadLive"` plus a NUL, writing up to 10 bytes past
the end into the adjacent struct member.

`MAX_PATH` is platform-dependent: **256** on Linux (`SBFspot/oslinux.h:60-62`)
and **260** on Windows (from the platform SDK). The overflow therefore
triggers at an `OutputPath` length of 247 chars or more on Linux, and 251 or
more on Windows.

## Affected code

`SBFspot/SBFspot.cpp:2087-2099`:

```cpp
//force settings to prepare for live loading to http://pvoutput.org/loadlive.jsp
if (cfg->loadlive)
{
    strncat(cfg->outputPath, "/LoadLive", sizeof(cfg->outputPath));   // line 2090 — BUG
    strcpy(cfg->DateTimeFormat, "%H:%M");
    cfg->CSV_Export = true;
    ...
}
```

Source population (same file, line 1714):

```cpp
strncpy(cfg->outputPath, value, sizeof(cfg->outputPath) - 1);
```

So `cfg->outputPath` holds up to `MAX_PATH − 1` attacker-influenced bytes
plus a terminator. The misuse of `strncat` then writes past it.

Adjacent struct members (`SBFspot/Types.h:80-81`):

```cpp
char outputPath[MAX_PATH];         // line 80
char outputPath_Events[MAX_PATH];  // line 81
```

`outputPath_Events` is declared immediately after `outputPath` in the same
struct. C guarantees struct members appear in declaration order, and
`char[]` members require no alignment padding, so the two buffers are
contiguous in memory on every mainstream compiler. The overflow corrupts
up to ~10 leading bytes of `outputPath_Events`, which is the path used for
event-CSV writes (see `SBFspot.cpp:2084` for the nearby use).

## Scope

A repository-wide grep for `strncat` returned exactly one match (this site).
The same misuse pattern does not appear elsewhere.

```
$ grep -rn 'strncat' SBFspot/ SBFspotUploadCommon/ SBFspotUploadDaemon/ SBFspotUploadService/
SBFspot/SBFspot.cpp:2090:    strncat(cfg->outputPath, "/LoadLive", sizeof(cfg->outputPath));
```

## Preconditions

1. SBFspot is invoked with the `-loadlive` command-line flag (PVoutput live-upload mode).
2. The `OutputPath` value in `SBFspot.cfg` is long enough to exhaust the buffer: ≥ 247 chars on Linux (`MAX_PATH = 256`), ≥ 251 chars on Windows (`MAX_PATH = 260`).
3. Either the operator is the attacker (low impact — they can already do worse) **or** the config file is writable by a less-privileged actor than the SBFspot process (e.g., shared NFS, misconfigured Docker bind mount, multi-user host where the SBFspot service runs under a more-privileged account).

The privilege-boundary case (precondition #3 second clause) is what justifies
treating this as a real vulnerability rather than a "user-trusted-config" bug.

## Exploit scenario

Threat model: a low-privileged user can write `SBFspot.cfg` (e.g., via a
shared bind-mounted volume in a container deployment), but the SBFspot binary
runs under a more-privileged account.

1. Attacker writes into `SBFspot.cfg`:
   ```
   OutputPath=/var/log/sbfspot/<245 A's>      # Linux  (total length = 255 chars = MAX_PATH-1)
   OutputPath=/var/log/sbfspot/<249 A's>      # Windows (total length = 259 chars = MAX_PATH-1)
   ```
2. The system operator (or systemd timer) starts SBFspot with `-loadlive`.
3. `strncpy` at line 1714 copies `MAX_PATH − 1` bytes of the attacker's path into `cfg->outputPath`.
4. `strncat` at line 2090 appends `"/LoadLive"` plus NUL, writing 10 bytes into `cfg->outputPath_Events`.
5. The attacker now controls the first ~10 bytes of `outputPath_Events`. Combined with knowledge of the format (a path), the attacker can redirect event CSV writes to a directory they read from, exfiltrating event data — **or** combine with a symlink to escalate writes.

Even in the single-user case, this is a deterministic out-of-bounds write
that violates the C standard and can manifest as silent path corruption
(event logs vanish or land in unexpected locations).

## Recommended fix

```diff
--- a/SBFspot/SBFspot.cpp
+++ b/SBFspot/SBFspot.cpp
@@ -2087,7 +2087,11 @@
             //force settings to prepare for live loading to http://pvoutput.org/loadlive.jsp
             if (cfg->loadlive)
             {
-                strncat(cfg->outputPath, "/LoadLive", sizeof(cfg->outputPath));
+                // CWE-787: strncat's size argument is the max source bytes,
+                // not the destination size. Compute remaining space.
+                const size_t used = strnlen(cfg->outputPath, sizeof(cfg->outputPath));
+                const size_t left = sizeof(cfg->outputPath) - used - 1;
+                strncat(cfg->outputPath, "/LoadLive", left);
                 strcpy(cfg->DateTimeFormat, "%H:%M");
```

**Better long-term fix:** migrate `cfg->outputPath` (and the other fixed-size
`char[MAX_PATH]` members in `Types.h`) to `std::string` and use `+=` /
`append`. The Configuration struct in this codebase already mixes
`std::string` and `char[]`; gradual migration is feasible.

**Defense-in-depth:** at config parse time (around `SBFspot.cpp:1714`) reject
`OutputPath` values longer than `sizeof(cfg->outputPath) - sizeof("/LoadLive") - 1`.

## Disclosure guidance

Lower urgency than findings #1–#3 (requires local config write, not
network-reachable). Can probably be filed as a normal issue/PR, but if
combined with the other findings in a coordinated disclosure, include it in
the same private advisory.
