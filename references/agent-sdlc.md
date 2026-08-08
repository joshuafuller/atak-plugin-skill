# Agent-driven SDLC, and engineering the loop

An agent that codes well still fails on a real project, because the hard part
is not writing code — it is knowing whether the code is right, and noticing
when it is not. That is a loop problem, and loops are designed, not hoped for.

This is the practice these repositories run on. Every rule here was paid for.

## Contents

- [Plan before building](#plan-before-building)
- [Goals must be able to end](#goals-must-be-able-to-end)
- [Iron Law: the failing test comes first](#iron-law-the-failing-test-comes-first)
- [Evidence beats claims](#evidence-beats-claims)
- [Loop engineering](#loop-engineering)
- [Git hooks](#git-hooks)
- [Agent hooks](#agent-hooks)
- [Feed findings back](#feed-findings-back)
- [Several agents at once](#several-agents-at-once)

## Plan before building

The most expensive defects are decisions, not bugs. A wrong function is a
morning; a wrong architecture is a fortnight, and it is usually discovered
after the UI is built on top of it.

The sequence that works:

1. **A PRD** — what it does, for whom, and what is explicitly out of scope.
   Short. Requirements numbered so issues can cite them.
2. **ADRs** for decisions that are expensive to reverse. Each records what was
   *measured*, the options, and the consequences. Status is `proposed` until
   the evidence exists.
3. **Issues** with acceptance criteria and a definition of done, each citing
   the requirement it serves.
4. **Spikes before commitments.** When a decision rests on an unknown, the
   unknown gets its own issue with a **kill criterion** — the number at which
   the idea is dead. Write the criterion before running it, or the result will
   be interpreted generously.

A worked example of why: the plan here assumed vector tiles could be
registered locally with ATAK. A spike measured it: registered locally the same
archive is classified `momap` — ATAK's raster type — and registered over
loopback it is `tak-cdn`/`tiles`. Everything downstream depended on that, and
no amount of careful UI work would have surfaced it.

## Goals must be able to end

If an agent is driven by a goal with a stopping condition, the condition must
be **observable**. "Polished, exquisite UI" cannot be satisfied by any
artifact, so a checker will always, correctly, find it unmet — and the agent
will work forever.

Bad: *finish the plugin to a polished, exquisite standard with perfect
workflows.*

Good: *commit `docs/evidence/map.png` showing roads drawn over the region.
Done when that file is in git.*

One observable per goal. Chain them; do not bundle them. Judgement words like
"polished" belong in a human review of a demo, not in a machine-checked
condition.

## Iron Law: the failing test comes first

Write the test, **run it, watch it fail**, then implement. A test that has
never been red proves nothing — it may assert something already true.

Make it checkable rather than claimed. Here that is a commit convention: the
red test lands in its own `test(red):` commit before the implementation, so the
history shows the order and CI can enforce it. Otherwise "I did TDD" is a
statement of character, not a fact about the repository.

Watch *how* it fails. "Cannot find symbol" is the expected first red. An
assertion failing with a number you did not predict means your model is
already wrong, and that is worth more attention than the implementation.

## Evidence beats claims

A definition of done in tiers, because "tests pass" and "it works" are
different claims:

1. **Unit tests green** — in CI, on every push.
2. **Integration/instrumented green** — on the real target.
3. **Observed** — a screenshot or a measured number, committed to the
   repository.

Tier 3 exists because the first two can pass while the feature does nothing. A
plugin here shipped resumable downloads with 72 green tests and no one had
watched it resume. Later, a map registered cleanly, reported success at every
layer, and drew nothing.

Numbers in commit messages beat adjectives: *"8.2 s, 1479 tiles, peak heap 180
MB"* can be compared next month; *"fast enough"* cannot.

## Loop engineering

The loop is the product. An agent's effectiveness is bounded by how quickly
and reliably reality gets back to it.

```
    plan  ->  specify  ->  build  ->  verify  ->  record  ->  correct
      ^                                                          |
      +----------------------------------------------------------+
```

Three properties worth designing for:

**Short.** A check that takes a minute gets skipped. Put the seconds-long
checks where they run every time, the slow ones where they run once.

**Automatic.** Anything requiring someone to remember it will be forgotten
under pressure, which is exactly when it matters. Hooks and CI, not
discipline.

**Honest.** A check that reports success when it did not run is worse than no
check. Two examples from this work: a scanner whose bad flag was read as "leak
found", and a reflective call that bound a *notifier* named
`dispatchStyleRegistered` and reported that the style had been applied. Both
looked like signal. Prefer a check that says "could not run".

## Git hooks

Deterministic, run for humans and agents alike, and survive a context reset.

| Hook | Put here | Because |
| --- | --- | --- |
| `pre-commit` | secret scan, SDK-material check, formatter | Seconds. Stops the mistake entering history, where removing it is a rewrite |
| `commit-msg` | message conventions (`test(red):`) | Makes the Iron Law machine-checkable |
| `pre-push` | unit tests, the linter | Slower, but before anything is shared |
| `post-merge` | dependency install, migrations | Stops "works on my branch" |

Keep them fast and make them bypassable in a genuine emergency — an
unbypassable slow hook is bypassed permanently by deleting it. If a hook is
routinely skipped, that is data: it is in the wrong stage.

## Agent hooks

Harness hooks are the loop's other half: they carry information *into* the
agent's context at the moment it is needed, rather than relying on it to
remember. In Claude Code these are configured in settings and fire on events:

| Event | Use it for |
| --- | --- |
| `SessionStart` | Prime context — current branch, failing tests, open issues, the goal |
| `PostToolUse` | React to an edit: run the formatter, run the affected test, run `scan` |
| `PreToolUse` | Refuse a dangerous action before it happens |
| `Stop` | Assert a goal condition and keep working until it holds |

The pattern that pays: **make the result of an action arrive on its own.** An
agent that must decide to check something will sometimes decide not to. An
agent that is told, automatically, that a test just went red, cannot miss it.

Two cautions. A `Stop` condition that is unobservable never releases the agent
(see above). And hook output is context: a hook that dumps a thousand lines on
every tool call costs more than it returns.

## Feed findings back

The loop is not closed until what was learned is written where the next run
will see it — a skill, a reference, an ADR. Otherwise the same hour is paid
repeatedly.

Trigger it on cost, not on novelty: **anything that took more than about half
an hour to diagnose is worth writing down**, and the note should say what the
symptom looked like, not just what the cause was. Symptoms are what the next
reader will have.

## Several agents at once

Parallelism is nearly free for code and expensive for anything exclusive.

- Give each agent its own directory — a separate repository, or a `git
  worktree`. Shared caches with proper locking are fine.
- Identify the **exclusive resources** and serialise them explicitly. Here it
  is the device: installing, instrumenting and driving the UI all contend, and
  one agent force-stopping the app under test looks, to another, like a crash
  it must debug. That happened, and cost both sides time.
- Say who owns what, in writing, before starting.
