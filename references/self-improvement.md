# The loop, and how it corrects itself

This skill is a **measurement record**, not a manual. Every claim in it was
true of one ATAK version on one machine on one day. Some are already wrong.

That has a practical consequence: when the skill and the device disagree, the
device wins and **the skill is defective until you fix it**. Fixing it is part
of the task you are already doing, not a separate chore for later — later never
comes, and the next agent pays the same cost you just paid.

## The loop

```
    orient  ->  specify  ->  build  ->  verify  ->  record  ->  correct
      |            |           |          |           |           |
   doctor      red test     make it   on device   evidence    fix the skill
                 first        pass                  in repo    if it misled
      ^                                                            |
      +------------------------------------------------------------+
```

Six steps. The last one is the one everybody skips, and it is the only one that
makes the next pass faster than this one.

### 1. Orient

`doctor` before anything. Most "the plugin will not load" sessions are a wrong
ATAK build, a missing sideload copy, or an emulator that renders nothing — all
of which `doctor` reports in seconds, and all of which cost an hour to diagnose
from symptoms.

If `doctor` passes and something is still wrong, that is a gap in `doctor`. Add
the check. A check that would have saved you an hour is worth ten minutes.

### 2. Specify

Write the failing test first, and **run it to watch it fail**. A test that has
never been red proves nothing: it may be asserting something already true, or
asserting nothing at all.

Watch *how* it fails. "Cannot find symbol" is the expected first red. An
assertion failure with a number you did not predict means your model is already
wrong, and that is worth more attention than the implementation is.

### 3. Build

Make it pass. Nothing else.

### 4. Verify — and know what your test cannot see

This is where ATAK punishes confidence. A JVM test proves the JVM. Android's
platform classes differ in ways that fail silently:

- an XML parser feature the JVM accepts and Android rejects, breaking every
  parse on device while unit tests stay green
- `List.of()` compiling happily against a `minSdk` that has no such method
- a UI automation script that "taps" successfully on an emulator rendering a
  black framebuffer

So: anything that only runs on device is only proven on device. And anything a
user sees is only proven by looking at it. Screenshot the working behaviour.

Do not report a pass you did not observe. `am instrument` exits 0 on failure.

### 5. Record

Commit the evidence next to the code: the screenshot, the measured number, the
logcat line that identified the cause. A number in a commit message is worth
more than a paragraph of description, because the next person can compare
against it.

### 6. Correct the skill

Now ask: **did this skill mislead me, or fail to help?**

Triggers — any one of these means edit the skill before you finish:

| Trigger | What to write |
| --- | --- |
| A claim here turned out false | Replace it. Do not hedge it — a hedged claim helps nobody and hides that it was measured wrong |
| Something cost more than ~30 minutes to diagnose | The symptom, the actual cause, and the command that distinguishes them |
| You found a workaround | The workaround **and** why the obvious thing fails, or the next agent will "fix" it back |
| A new ATAK version behaves differently | Version-qualify both behaviours; do not silently overwrite the old one |
| You needed a fact that was not here | Add it, with how you established it |

## How to write a correction

**Measured, not inferred.** "ATAK probably caches the coverage" is worth
nothing. "Coverage did not extend until the dataset was re-registered, ATAK-CIV
5.8.0.1" is worth an hour to somebody.

**Say how you know.** A claim with its method attached can be re-checked when a
version changes. A bare claim can only be trusted or deleted.

**Delete aggressively.** A stale claim is worse than a missing one, because it
is believed. If you cannot tell whether something is still true, say when it
was last verified.

**Respect the budget.** `SKILL.md` earns its place by being short enough to
read entirely. A fact belongs there only if it changes what you do *first*;
everything else goes to `references/`. If `SKILL.md` grows past roughly 150
lines, something in it has stopped being load-bearing — find it and move it
down.

**One commit per finding**, with the measurement in the message. The skill's
git history then becomes the provenance for every claim in it.

## Propose it as a pull request — do not push to main

A wrong claim here does not cost one session, it costs every future session,
silently, because it is believed. So corrections are reviewed.

The installed skill is a git checkout, so an agent can raise the PR from where
it is already standing:

```bash
cd ~/.claude/skills/atak-plugin      # a clone of the skill repo
git checkout -b finding/coverage-needs-reregistration
$EDITOR references/atak-behaviour.md
git commit -am "Coverage does not extend until the dataset is re-registered

Measured on ATAK-CIV 5.8.0.1, Android 14 emulator (google_apis).
Added tiles to a registered MBTiles; getCoverage() returned the
original bounds and the new area did not draw until remove-then-add.
Cost about an hour to find, presenting as 'the tiles are corrupt'."
git push -u origin finding/coverage-needs-reregistration
gh pr create --fill --label finding
```

Repository: <https://github.com/joshuafuller/atak-plugin-skill>

**Branch names say what was found**, not what was touched:
`finding/<slug>` for a new measurement, `correction/<slug>` when the skill
was wrong, `stale/<slug>` when a claim can no longer be verified.

### What the PR must contain

The pull request template asks for these, and a PR without them should not be
merged:

- **The claim**, stated so it can be falsified.
- **How it was measured** — the commands, the versions, the device.
- **What it cost** — how long the wrong understanding took to unpick. This is
  how the reviewer judges whether it belongs in `SKILL.md` or a reference.
- **What it replaces**, if anything, and whether the old claim was wrong or
  merely version-specific.

### Raise the PR even if you cannot finish it

A PR that says "this claim looks wrong, here is what I saw, I did not have time
to establish why" is worth having. The alternative is that the observation dies
with the session. Label it `unverified` and say so in the body.

### One finding per PR

Findings get accepted and rejected individually. Two in one branch means the
weaker one either blocks the stronger or rides in unexamined.

## What not to do

- Do not add advice you have not tested. The value of this skill is that its
  claims were measured; one invented paragraph makes readers doubt all of them.
- Do not record project-specific decisions here. "a plugin uses one MBTiles
  archive" belongs in that plugin's own ADRs. "ATAK reads MBTiles from
  `imagery/mobile/mapsources/`" belongs here.
- Do not turn a specific finding into a general rule on one observation.

## The test of whether this is working

The same failure should never cost two agents an hour. If it does, the loop was
run five steps out of six.
