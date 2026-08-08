# Agent-driven SDLC, and engineering the loop

An agent that writes good code still fails on a real project, because the hard
part is knowing whether the code is right and noticing when it is not. That is
a loop problem, and loops are designed rather than hoped for.

Sources are named inline. Where a claim comes from this project's own
experience rather than a published practice, it says so.

## Contents

- [Specify before building](#specify-before-building)
- [Explore, plan, implement, commit](#explore-plan-implement-commit)
- [Evidence, not assertion](#evidence-not-assertion)
- [Goals must be able to end](#goals-must-be-able-to-end)
- [Iron Law: the failing test comes first](#iron-law-the-failing-test-comes-first)
- [Hooks: the deterministic half of the loop](#hooks-the-deterministic-half-of-the-loop)
- [Git hooks](#git-hooks)
- [Context discipline](#context-discipline)
- [Exclusive resources and naming](#exclusive-resources-and-naming)
- [Feed findings back](#feed-findings-back)

## Specify before building

**Source: GitHub's Spec Kit and its `spec-driven.md`** (github/spec-kit, MIT).

Its claim is that specifications stop being scaffolding and become the
artifact: *"specifications don't serve code — code serves specifications."*
Whether or not you accept the strong form, the workflow it encodes is worth
copying, because each step exists to stop a specific failure:

| Step | Command in Spec Kit | What it prevents |
| --- | --- | --- |
| **Constitution** | `/speckit.constitution` | Principles being re-litigated per feature |
| **Specify** | `/speckit.specify` | Building the wrong thing. Deliberately *what and why*, not *how* |
| **Plan** | `/speckit.plan` | Technology choices with no recorded rationale |
| **Tasks** | `/speckit.tasks` | A plan nobody can execute; emits `tasks.md`, marking independent work `[P]` for safe parallelism |

Two details from its templates transfer even if you never install it:

**Ban implementation detail from the spec.** The template literally instructs
*"Focus on WHAT users need and WHY / Avoid HOW to implement (no tech stack,
APIs, code structure)."* This keeps the spec stable while the stack changes,
and stops the model jumping to a framework before the problem is stated.

**Force uncertainty to be visible.** The template mandates
`[NEEDS CLARIFICATION: specific question]` markers, with the instruction
*"Don't guess: if the prompt doesn't specify something, mark it."* An LLM's
default failure is to resolve ambiguity silently and plausibly. A marker turns
that into a visible question.

**From this project:** add a **kill criterion** to any spike, written before it
runs. The threshold at which the idea is dead, decided in advance, because
afterwards the result will be read generously. A spike here measured that ATAK
classifies the same archive as `momap` (raster) when registered locally and
`tak-cdn`/`tiles` (vector) when served over loopback. Everything downstream
depended on it and no amount of careful UI work would have surfaced it.

## Explore, plan, implement, commit

**Source: Anthropic, *Claude Code best practices*.** Four phases, with
exploration deliberately separated from execution: *"Letting Claude jump
straight to coding can produce code that solves the wrong problem."*

1. **Explore** — read the code, answer questions, change nothing. Plan mode.
2. **Plan** — produce a written plan; edit it directly before proceeding.
3. **Implement** — code against the plan, run the tests, fix failures.
4. **Commit** — descriptive message, PR.

The same source is clear about the cost, and this matters as much as the
practice: *"Plan mode is useful, but also adds overhead… If you could describe
the diff in one sentence, skip the plan."* Planning earns its keep when the
approach is uncertain, several files change, or the code is unfamiliar.

## Evidence, not assertion

**Source: Anthropic, same document:** *"Have Claude show evidence rather than
asserting success: the test output, the command it ran and what it returned, or
a screenshot of the result. Reviewing evidence is faster than re-running the
verification yourself, and it works for sessions you weren't watching."*

**From this project**, a three-tier definition of done, because "tests pass"
and "it works" are different claims:

1. Unit tests green, in CI.
2. Integration or instrumented tests green, on the real target.
3. **Observed** — a screenshot or a measured number, committed.

Tier 3 exists because the first two pass while the feature does nothing. Here,
resumable downloads shipped with 72 green tests and nobody had watched a
download resume; later a map registered cleanly, reported success at every
layer, and drew nothing.

Numbers beat adjectives in commit messages. *"8.2 s, 1479 tiles, peak heap 180
MB"* can be compared next month. *"Fast enough"* cannot.

## Goals must be able to end

**From this project, learned expensively.** If an agent runs against a stopping
condition, that condition must be **observable**. "Polished, exquisite UI"
cannot be satisfied by any artifact, so a checker correctly finds it unmet
forever. One such goal here ran 11 hours and 13 turns without terminating.

Bad: *finish the plugin to a polished, exquisite standard.*
Good: *commit `docs/evidence/map.png` showing roads drawn over the region.*

One observable per goal, chained. Judgement words belong in a human review of a
demo, not in a machine-checked condition.

## Iron Law: the failing test comes first

Write the test, **run it, watch it fail**, then implement. A test that has
never been red proves nothing — it may assert something already true.

Spec Kit reaches the same place from the other direction: *"Acceptance
scenarios become tests… test scenarios aren't written after code, they're part
of the specification that generates both implementation and tests."*

**From this project:** make it checkable rather than claimed. The red test
lands in its own `test(red):` commit before the implementation, so history
shows the order and CI can enforce it. Otherwise "I did TDD" is a statement of
character, not a fact about the repository.

Watch *how* it fails. "Cannot find symbol" is the expected first red. An
assertion failing with a number you did not predict means your model is already
wrong.

## Hooks: the deterministic half of the loop

**Source: Anthropic, *Claude Code best practices* and the hooks reference.**
The distinction that matters: *"Unlike CLAUDE.md instructions which are
advisory, hooks are deterministic and guarantee the action happens."* Use them
*"for actions that must happen every time with zero exceptions."*

Events fire at three cadences — once per session, once per turn, and on every
tool call:

| Event | Fires | Use for |
| --- | --- | --- |
| `SessionStart` | session begins or resumes | Prime context: branch, failing tests, open issues |
| `UserPromptSubmit` | before a prompt is processed | Inject standing constraints |
| `PreToolUse` | before a tool call; **can block it** | Refuse dangerous actions |
| `PostToolUse` | after a tool call succeeds | Format, lint, run the affected test, `scan` |
| `PostToolUseFailure` | after a tool call fails | React to the failure rather than hoping it is noticed |
| `Stop` | when the turn ends | Assert a goal condition and keep working |
| `SubagentStart` / `SubagentStop` | around delegated work | Track parallel work |
| `FileChanged` | a watched file changes on disk | React to something *outside* the agent |
| `PreCompact` / `PostCompact` | around context compaction | Preserve state across a compaction |

Three mechanics worth knowing precisely:

- **Exit code 2 blocks**, and the reason is shown to the agent. Exit 0 with no
  output is "no opinion" — it does not approve anything.
- **`async: true`** runs a hook in the background without blocking the turn.
- **`asyncRewake: true`** runs in the background and **wakes the agent on exit
  code 2**, surfacing the hook's stderr as a system reminder. This is the
  mechanism for exactly the thing that is otherwise unsolvable: a long-running
  check — a full test suite, a device run — whose *failure* needs to reach the
  agent minutes later, without blocking or being polled.
- Hooks also fire inside subagents, carrying `agent_id` and `agent_type`.

Two cautions. A `Stop` condition that is not observable never releases the
agent. And hook output is context: a hook that prints a thousand lines on every
tool call costs more than it returns.

## Git hooks

Deterministic for humans and agents alike, and they survive a context reset.

| Hook | Put here | Because |
| --- | --- | --- |
| `pre-commit` | secret scan, SDK-material check, formatter | Seconds. Stops the mistake entering history, where removal means a rewrite |
| `commit-msg` | message conventions (`test(red):`) | Makes the Iron Law machine-checkable |
| `pre-push` | unit tests, linter | Slower, but before anything is shared |
| `post-merge` | dependency install, migrations | Stops "works on my branch" |

Keep them fast and bypassable in a genuine emergency. An unbypassable slow hook
gets deleted, and then it protects nothing. If a hook is routinely skipped,
that is data: it is in the wrong stage.

## Context discipline

**Source: Anthropic.** CLAUDE.md is loaded every session, so *"only include
things that apply broadly"*, and the test for each line is *"Would removing
this cause Claude to make mistakes? If not, cut it."* The failure mode is
specific and worth quoting: *"Bloated CLAUDE.md files cause Claude to ignore
your actual instructions."* If a rule keeps being violated despite existing,
the file is probably too long.

The division of labour that follows:

- **CLAUDE.md** — always loaded. Broad, short, project-wide.
- **Skills** — loaded on demand. Domain knowledge and workflows that are only
  sometimes relevant. This is why this content is a skill.
- **Subagents** (`.claude/agents/`) — their own context and tool set, for work
  that reads many files without polluting the main conversation.
- **Hooks** — no context cost until they fire.

Also from the same source, and cheap: prefer CLI tools such as `gh`, which are
*"the most context-efficient way to interact with external services."*

## Exclusive resources and naming

**From this project, learned the hard way.** Parallelism is nearly free for
code and expensive for anything exclusive. The exclusive resource here is the
**device**, and it caused a genuine failure: two agents used the same emulator,
one force-stopped ATAK to run instrumented tests, and the other spent time
debugging what looked like a spontaneous crash in its own work.

Nothing about the failure was visible from either side. `adb` reports
`emulator-5554`, which says nothing about who is using it or why.

What prevents it:

**Name the AVD after the work, not the hardware.** `maproom_atak`,
`takfabric_a` — not `Pixel_6_API_34`. The name then appears in
`adb -s <serial> shell getprop ro.boot.qemu.avd_name`, so any agent can ask
what it is looking at rather than assuming.

**Pin the serial, and pin the port when you create the AVD:**

```bash
emulator -avd maproom_atak -port 5554 &     # a serial you chose
export ANDROID_SERIAL=emulator-5554          # every adb call honours this
```

**Discover before acting**, never assume the only device is yours:

```bash
for d in $(adb devices | awk 'NR>1 && $2=="device"{print $1}'); do
  printf '%s %s\n' "$d" "$(adb -s "$d" shell getprop ro.boot.qemu.avd_name | tr -d '\r')"
done
```

**Leave a claim on the device itself**, where the other agent will actually
look:

```bash
adb -s "$ANDROID_SERIAL" shell "echo 'claimed-by=<who> purpose=<what>' \
    > /sdcard/atak/DEVICE_CLAIM.txt"
```

**Never `force-stop` an app on a device you do not own.** That is the specific
action that turns contention into a phantom bug in someone else's session.

Write the assignment down before starting. "Agent A owns 5554, agent B owns
5556" costs one line and saves an afternoon.

## Feed findings back

The loop is not closed until what was learned is written where the next run
will see it — a skill, a reference, an ADR. Spec Kit calls this **bidirectional
feedback**: *"Production reality informs specification evolution."*

Trigger it on cost, not novelty: anything that took more than about half an
hour to diagnose is worth writing down, and the note should record what the
*symptom* looked like, not only the cause. The symptom is what the next reader
will have.
