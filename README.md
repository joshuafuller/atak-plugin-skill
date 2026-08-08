# ATAK plugin skill

[![checks](https://github.com/joshuafuller/atak-plugin-skill/actions/workflows/checks.yml/badge.svg)](https://github.com/joshuafuller/atak-plugin-skill/actions/workflows/checks.yml)

A [Claude](https://claude.ai/code) skill for building ATAK plugins — scaffolding
from the SDK template, getting one to actually load on a device, running
instrumented tests inside ATAK, and preparing a Third Party Pipeline
submission.

Everything in it was measured against **ATAK-CIV 5.8.0.1** and its SDK on an
Android 14 emulator, rather than inferred from documentation. Where a claim is
version-specific it says so.

## Why

ATAK plugin development has a handful of failures that each look like a
different problem than they are. A plugin that will not load reports
"Incompatible", which sounds like a version mismatch and is almost always a
signing certificate. `connectedAndroidTest` hangs for ten minutes with no
output rather than reporting that it cannot reach adb. A UI test passes against
an emulator that is rendering nothing at all.

Each of those cost an afternoon once. The skill exists so they cost nothing
again.

## Install

```bash
git clone https://github.com/joshuafuller/atak-plugin-skill \
    ~/.claude/skills/atak-plugin
```

Claude picks it up automatically. Invoke it with `/atak-plugin`, or just start
working on an ATAK plugin — the description triggers on plugin work, signature
mismatches, `takdev`, and hanging instrumented tests.

## What is in it

`SKILL.md` is the entry point and stays short on purpose. It carries the facts
that cost the most time, a logcat table that maps each line to its actual
cause, the language question (ATAK ships Kotlin; the Java-only template is
misleading), and the licence boundary that decides what may be committed.

Detail lives in `references/`, read only when relevant:

| File | Read it when |
| --- | --- |
| `environment.md` | Setting up a build environment, or adb/Gradle is misbehaving in a container |
| `testing.md` | Writing or running tests — Espresso wiring, the `_modApk` trap, getting a plugin context |
| `extension-points.md` | Deciding how the plugin hooks into ATAK — import, tile formats, styles, elevation |
| `atak-behaviour.md` | Designing around how ATAK treats maps and imports; explains behaviour that looks like bugs |
| `android-gotchas.md` | Something works on the JVM and fails on device |
| `shipping.md` | Preparing a Third Party Pipeline submission |

## Related

- [atak-plugin-dev](https://github.com/joshuafuller/atak-plugin-dev) — the
  development container this skill assumes, with `doctor`, `deploy` and
  `instrument`, and an `AGENTS.md` for running the whole loop without a human.

## The SDK is not here

No ATAK SDK material is in this repository, and none should be added. The TAK
licence permits deriving applications from the SDK and forbids redistributing
it. Download your own from [tak.gov](https://tak.gov).

## Licence and attribution

MIT — see [LICENSE](LICENSE).

ATAK and TAK are products of the TAK Product Center and the U.S. Government.
This is an independent, unofficial reference and is **not affiliated with or
endorsed by** them. [NOTICE.md](NOTICE.md) covers that, how the claims here
were established, and the licensing of contributions.
