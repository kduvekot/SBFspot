# SBFspot Security Review

Consolidated findings from two independent code-review passes performed on the
same upstream commit. Both reviews independently identified findings #1, #2,
and #3 below; finding #4 was reported by only one of the two passes. This
document supersedes the per-branch write-ups that previously lived under
`docs/review/` on the `claude/explore-codebase-yAmoS` and
`claude/security-review-ioJOe` branches.

## Scope and methodology

| | |
|---|---|
| **Target** | [SBFspot](https://github.com/SBFspot/SBFspot) |
| **Version reviewed** | 3.10.0 |
| **Commit** | `b77f650` (tip of `master` at review time) |
| **Reviews consolidated** | `claude/explore-codebase-yAmoS` + `claude/security-review-ioJOe` |
| **Method** | Static code review. Multi-pass reviews with parallel reviewers per attack surface (network protocol parsing, database layer, file/config handling, MQTT/HTTP clients), followed by an explicit false-positive filtering pass. |
| **Verification** | **Static analysis only.** No PoC frames or payloads were sent against a running build. |
| **Confidence threshold** | Findings reported here scored ≥8/10 in the filter pass. |

Where the two reviews produced independent write-ups of the same underlying
bug, the merged document below keeps the higher-detail treatment from each
side (e.g. CVSS scoring from one branch, SQLite stacked-query/ATTACH analysis
from the other).

## Findings

| # | File | Severity | CVSS | Category |
|---|---|---|---|---|
| [1](001-bluetooth-packet-length-overflow.md) | `SBFspot/SBFspot.cpp:114` | HIGH | 8.8 | Buffer overflow (CWE-120) — Bluetooth `recv()` with attacker-controlled length |
| [2](002-mqtt-command-injection.md) | `SBFspot/mqtt.cpp:285` | HIGH | 8.8 | OS command injection (CWE-78) — inverter strings into `system()` |
| [3](003-sql-injection-type-label.md) | `SBFspot/db_MySQL.cpp:137-153` (+ SQLite mirror) | HIGH | 8.1 | SQL injection (CWE-89) — `type_label()` |
| [4](004-strncat-loadlive-overflow.md) | `SBFspot/SBFspot.cpp:2090` | MEDIUM-HIGH | 3.4 – 7.8 | Out-of-bounds write (CWE-787) — `strncat` size-arg misuse |

Findings #1–#3 share a common root cause: inverter-supplied bytes (the
`pkLength` header field and the `DeviceName`/`DeviceType`/`SWVersion`
strings read at `SBFspot/SBFspot.cpp:2595`) are treated as trusted throughout
the pipeline. A single sanitization pass at the ingest point would not be a
substitute for the per-sink fixes, but it would meaningfully reduce the blast
radius of any future sink that is added and forgotten.

## Disclosure guidance

The repository has no `SECURITY.md`. We recommend:

1. **File a private [GitHub Security Advisory](https://github.com/SBFspot/SBFspot/security/advisories/new)** for findings 1–3 (network-reachable, pre-auth). Do **not** file public issues for these.
2. Finding 4 can ride along in the same advisory or, if disclosed separately, be filed as a regular issue/PR.
3. Suggest the maintainer add a `SECURITY.md` documenting their preferred coordination channel.

## Caveats

- All locations are pinned to commit `b77f650`. Line numbers may drift on later commits — verify with `git blame` before applying patches.
- The patches in each finding are illustrative diffs, not committed code. They have not been built or tested.
- Threat model assumes a LAN-resident attacker, a malicious/compromised inverter, or (for finding #4) a low-privileged actor with write access to `SBFspot.cfg`. None of the findings require pre-existing authentication to the SBFspot process itself.
