# Repository layout, and what must never be committed

## One repo per plugin, none nested

The container is one repository; **each plugin is its own, and none of them
nests inside another.** Plugin repos sit side by side in a directory the
container mounts at `/work`.

```
PLUGINS_DIR/               ->  /work
  atak-plugin-alpha/
  atak-plugin-beta/
  atak-plugin-alpha.worktrees/some-branch/
```

Pointing the mount at the parent rather than at one project is what lets
several people — or several agents — work on different plugins, or different
`git worktree`s of one plugin, at once. Nothing is shared but the Gradle cache,
which locks correctly for concurrent builds.

Only one thing is genuinely exclusive: **the device.** Installing, running
instrumented tests and driving the UI all contend for it. Serialise device work
between agents; parallelise everything else.

The container's helper scripts (`doctor`, `deploy`, `instrument`, `adb-bridge`)
belong to the container, are mounted read-only at `/opt/tak-bin`, and are on
`PATH`.

If a plugin did end up nested inside another repo, `git subtree split
--prefix=<path>` extracts it with its history intact.

## The licence boundary — settle it before the first commit

The TAK licence permits deriving applications from the SDK and **forbids
copying, publishing or distributing the SDK itself**. Scaffolding a plugin from
`samples/plugintemplate` copies SDK files into your working tree, and if you
commit them, your repository now contains SDK material.

This is easy to miss and expensive to undo. A plugin scaffolded from the
template can carry dozens of byte-identical SDK files — the gradle scripts,
sample resources, the espresso archives, the typst user-manual scaffolding.
One real plugin, checked after the fact, had **45**.

History is what gets published, so a private repo does not protect you.
Removing the files in a later commit achieves nothing; it takes a history
rewrite.

Check what you are about to commit:

```bash
for f in $(git ls-files); do
  m=$(find "$ATAK_SDK" -name "$(basename "$f")" -type f | head -1)
  [ -n "$m" ] && cmp -s "$f" "$m" && echo "SDK material: $f"
done
```

Gitignore anything that matches, and copy it from `$ATAK_SDK` at build time
instead. Do this at scaffold time — that is the moment the mistake is made, and
the only moment it is cheap to avoid.
