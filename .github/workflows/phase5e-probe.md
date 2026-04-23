# `phase5e-probe.yml` — how we rebuild SBFspot's Linux tarballs

Companion doc for
[`phase5e-probe.yml`](phase5e-probe.yml). Written so someone who has
never touched GitHub Actions, never cross-compiled, and doesn't know
what PIE or RELRO stand for can still follow along. If you already
know those terms, skip ahead — headings are dense enough to navigate.

**Status:** current pipeline as of April 2026. All 15 cells produce
modern, hardened, reproducible binaries that are drop-in replacements
for upstream V3.9.12's hand-built tarballs.

**Companion forensic docs** in [`../docs/`](../docs/) record every
experiment that led here. This doc is forward-looking: "why does the
workflow look like it does today." The phase baselines are backward-
looking: "what did we learn on the way here."

---

## Reading order

Each section is self-contained, but they build on each other.

1. [What we're actually trying to do](#1-what-were-actually-trying-to-do)
2. [The shape of a run: runners, chroots, the 15 cells](#2-the-shape-of-a-run)
3. [Step-by-step walkthrough](#3-step-by-step-walkthrough)
4. [Why each compile flag](#4-why-each-compile-flag)
5. [Why each link flag](#5-why-each-link-flag)
6. [The PIC libbluetooth rebuild](#6-the-pic-libbluetooth-rebuild)
7. [Per-cell matrix table](#7-per-cell-matrix-table)
8. [Things we tried and dropped](#8-things-we-tried-and-dropped)
9. [Glossary](#9-glossary)

---

## 1. What we're actually trying to do

*TBD*

## 2. The shape of a run

*TBD*

## 3. Step-by-step walkthrough

*TBD*

## 4. Why each compile flag

*TBD*

## 5. Why each link flag

*TBD*

## 6. The PIC libbluetooth rebuild

*TBD*

## 7. Per-cell matrix table

*TBD*

## 8. Things we tried and dropped

*TBD*

## 9. Glossary

*TBD*
