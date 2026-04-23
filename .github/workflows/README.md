# `.github/workflows/` — what lives here

This directory holds every GitHub Actions workflow for the fork. The
project is a **CI release-pipeline experiment** for
[SBFspot/SBFspot](https://github.com/SBFspot/SBFspot): we rebuild the
upstream V3.9.12 Linux release tarballs in GitHub Actions, so the
maintainer can eventually replace hand-built Windows-cross artefacts
with reproducible, auditable CI builds.

Most workflows are **frozen probes** from earlier investigation
phases. Each one is gated behind a `.<name>-trigger` file in this
folder — renaming that file (or touching its timestamp) is the only
way to re-run the probe. That keeps CI minutes free while preserving
every experiment as executable documentation.

## Active workflow

| File | Companion doc | Status |
|---|---|---|
| [`phase5e-probe.yml`](phase5e-probe.yml) | [`phase5e-probe.md`](phase5e-probe.md) | **Current** — modern hardened pipeline, 15-cell matrix. |

## Frozen probes (historical)

Each of these was a milestone that taught us something; they're kept
so anyone can reproduce our findings, and so a later phase can
revive one if needed. Deep rationale for each is in
[`../docs/phase*-baseline.md`](../docs/).

| File | What it did | Evidence |
|---|---|---|
| [`probe.yml`](probe.yml) | Phase 0 feasibility: proved `ubuntu-24.04-arm` runs 32-bit armhf binaries natively; `snapshot.debian.org` works with debootstrap; Raspbian snapshot service doesn't. | `docs/` (CLAUDE.md Phase 0 section) |
| [`fingerprint.yml`](fingerprint.yml) | Phase 1 CI audit: dissects all 15 upstream tarballs (gzip metadata, tar layout, ELF headers, NEEDED, `.comment`, build-id) for third-party reproducibility. | `docs/fingerprint.md` |
| [`release.yml`](release.yml) | Phase 2/3 first successful rebuild: `sqlite × arm × bookworm` matched upstream's binary byte-for-byte after several iterations. | `docs/phase2-baseline.md`, `phase3.1-baseline.md` |
| [`toolchain-probe.yml`](toolchain-probe.yml) | Phase 3.2: rebuilt gcc-12 from Raspbian source to prove pipeline self-reproducibility. | `docs/phase3.2-baseline.md` |
| [`bullseye-probe.yml`](bullseye-probe.yml) | Phase 3.4: first successful full byte-match on bullseye via `-fmacro-prefix-map` wrapper. | `docs/phase3.4-baseline.md` |
| [`phase35-probe.yml`](phase35-probe.yml) | Phase 3.5: generalisation across DB / codename / arch axes. | `docs/phase3.5-baseline.md` |
| [`phase4-probe.yml`](phase4-probe.yml) | Phase 4: back-tested V3.9.10 and V3.9.11 on the same pipeline. Same residual across versions → confirms arm-bookworm irreducible is a toolchain property. | `docs/phase4-baseline.md` |
| [`phase4b-probe.yml`](phase4b-probe.yml) | Phase 4b: narrowed the arm-bookworm residual to specific ELF sections. | `docs/phase4-baseline.md` (addendum) |
| [`phase4c-probe.yml`](phase4c-probe.yml) | Phase 4c: probed three libstdc++ rebuild flags (none closed the residual). | `docs/phase4-baseline.md` (addendum) |
| [`phase5-matrix.yml`](phase5-matrix.yml) | Phase 5: 15-cell matrix run measuring byte-match on every combo. 7 of 25 binaries matched; 18 fell in documented residual categories. | `docs/phase5-baseline.md` |
| [`phase5d-probe.yml`](phase5d-probe.yml) | Phase 5d: tested and retired the SysGCC-toolchain hypothesis via Wine/xdotool automation. SysGCC r2 = Raspbian package verbatim. | `docs/phase5d-baseline.md` |

## How to re-run a frozen probe

1. Touch the corresponding `.<name>-trigger` file:
   `date -u +"%Y-%m-%dT%H:%M:%SZ" > .github/workflows/.<name>-trigger`
2. Commit + push.
3. The `on.push.paths` filter in the workflow fires on that specific
   trigger file changing.

Alternatively, use the GitHub UI: **Actions → <workflow name> → Run
workflow** (every workflow also has `workflow_dispatch:` enabled).

## Conventions

- **Frozen-probe gate.** One-shot investigations use
  `on.push.paths: [.github/workflows/.<name>-trigger]` plus
  `workflow_dispatch:`. The workflow is inert until the trigger
  file changes.
- **Matrix per combo.** The 15 upstream tarballs are
  `{sqlite, nosql, mariadb} × {buster, bullseye, bookworm} × {arm, arm64}`
  minus `buster × arm64` (upstream never published it). Any workflow
  building more than one combo uses a `strategy.matrix` block keyed
  by a single `cell:` list where each entry fully describes the combo.
- **Documentation.** One-line YAML comments for what-a-step-does.
  Extended rationale (why a flag, what we tried first, what broke)
  goes in the companion `.md` — e.g. `phase5e-probe.md`.
- **Action pins.** `actions/checkout@v6`, `actions/upload-artifact@v7`,
  `actions/download-artifact@v8` (all Node.js 24). Bump together.
- **ARM runner.** `runs-on: ubuntu-24.04-arm` — aarch64 silicon that
  runs 32-bit armhf binaries natively (no qemu). See Phase 0 findings.

## Out of scope

The fork deliberately does **no C++ source changes**. Everything in
this folder is CI plumbing + documentation. Behavioural fixes go as
separate upstream PRs (see `CLAUDE.md → Non-goals`).
