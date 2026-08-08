# Contributing

Most contributions here will come from an agent that just lost an hour to
something this skill should have warned it about. The process is shaped around
that: cheap to raise, strict about evidence.

## The one rule

**Every claim is measured.** This skill's only value is that its statements
were true of a real device on a stated date. One invented paragraph makes a
reader doubt all of them, and they are right to.

Inferred from documentation is not measured. Reasoned from how Android usually
works is not measured. Observed once, with the command written down, is
measured.

## Raising a PR

```bash
cd ~/.claude/skills/atak-plugin        # the installed skill is a checkout
# ...edit...
bin/propose finding "Coverage does not extend until re-registration"
```

`bin/propose` branches, commits, pushes and opens the PR with the template
filled in. It refuses to run against `main`, refuses an empty diff, and refuses
a title that does not state a claim.

Doing it by hand is four commands:

```bash
git checkout -b finding/<slug>
git commit -am "<claim>

<how it was measured, versions, device>"
git push -u origin finding/<slug>
gh pr create --fill --label finding
```

## The four shapes a PR takes

Each is reviewed against different criteria, so say which one it is.

### `finding` — a new measured fact

Something true that was not written down.

**Acceptance criteria**

- The claim is falsifiable as written. "ATAK is picky about styles" is not a
  claim; "ATAK ignores MapLibre `get` expressions and renders nothing for that
  layer" is.
- The method is reproducible: commands, ATAK version, SDK version, device or
  emulator image.
- It is general to ATAK plugin development, not to one project.
- Its placement matches its cost. `SKILL.md` is for facts that change what you
  do *first*, or that present as a different problem than they are. Everything
  else goes to `references/`.

### `correction` — the skill is wrong

**Acceptance criteria**

- The existing text is quoted, so the reader can see what changed.
- It says whether the old claim was **wrong outright** or **true of an earlier
  version**. Version-qualified claims are kept side by side; wrong ones are
  deleted, not softened.
- Evidence meets the `finding` bar. Correcting a measured claim needs a
  measurement, not an argument.

### `stale` — a claim can no longer be verified

For when something cannot be reproduced and you cannot establish why.

**Acceptance criteria**

- States what was tried and what happened instead.
- Proposes either deletion or a "last verified on ..." qualifier.
- Does not silently delete a claim that may still hold on another version.

### `structure` — moving or trimming, no new claims

Splitting an overgrown `SKILL.md`, moving a fact to a reference, tightening
prose.

**Acceptance criteria**

- The diff adds no new factual claims. If it does, it is a `finding`.
- Nothing is lost — moved, or deliberately deleted and said so.
- `SKILL.md` still reads end to end.

## Definition of Done

A PR is done when all of these hold. They are checked at review, and a
reviewer who cannot confirm one should ask rather than assume.

1. **One finding.** Two findings in a branch means the weaker one either blocks
   the stronger or rides in unexamined.
2. **Measured, with the method in the PR body** — commands, versions, device.
3. **Placed by cost, not by topic.** A fact that costs an afternoon and looks
   like something else earns a line in `SKILL.md`; a fact you look up when you
   need it goes in `references/`.
4. **`SKILL.md` still reads end to end**, and is under roughly 150 lines.
   Progressive disclosure is the design: if it has grown, something in it has
   stopped being load-bearing.
5. **No stale text left behind.** A correction that adds the right answer next
   to the wrong one has made things worse.
6. **No ATAK SDK material.** No code, resources, gradle scripts or binaries
   from the SDK. The TAK licence forbids redistributing it, and this repository
   is meant to be publishable.
7. **No project-specific decisions.** "Map Room stores one MBTiles archive"
   belongs in that project's ADRs. "ATAK reads streaming descriptors from
   `imagery/mobile/mapsources/`" belongs here.
8. **The commit message carries the measurement**, so the git history remains
   the provenance for every claim in the skill.

## Incomplete findings are welcome

A PR saying "this claim looks wrong, here is what I saw, I could not establish
why" is worth more than a lost observation. Label it `unverified`, say so in
the body, and open it anyway. It will not be merged into `SKILL.md` as fact,
but it will be there for whoever hits it next.

## What gets rejected

- Advice that was not tested.
- A general rule extrapolated from one observation.
- Hedged claims — "may sometimes fail" tells nobody what to do.
- Anything that makes `SKILL.md` longer without making the first ten minutes of
  a session cheaper.
