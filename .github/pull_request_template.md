<!--
One finding per PR. Delete the sections that do not apply, but do not delete
"How it was measured" — a claim without a method cannot be re-checked when the
next ATAK version lands, and an unverifiable claim is worse than none.
-->

**Type:** finding | correction | stale | structure
<!-- finding: a new measured fact
     correction: something in the skill is wrong
     stale: a claim can no longer be verified
     structure: moving content, no new claims -->

## The claim

<!-- One or two sentences, stated so someone could prove it false. -->

## How it was measured

<!-- Commands, versions, device. Enough that a reader can repeat it.

ATAK-CIV 5.8.0.1 | SDK 5.8.0.1 | Android 14 emulator, google_apis, x86_64

    $ adb shell ...
    <output>
-->

## What it cost

<!-- How long the wrong or missing understanding took to unpick, and what the
symptom looked like. This decides placement: things that cost hours and present
as something else belong in SKILL.md; the rest belongs in references/. -->

## What it replaces

<!-- For corrections: the old text, and whether it was wrong outright or only
true of an earlier version. Version-qualify rather than overwrite when both
were true at some point. -->

---

- [ ] One finding, not several
- [ ] Measured on a device or emulator, not inferred from documentation
- [ ] Versions stated
- [ ] Placed by cost: `SKILL.md` only if it changes what you do first
- [ ] `SKILL.md` still reads end to end and is under ~150 lines
- [ ] No ATAK SDK material added
- [ ] No project-specific decisions added
