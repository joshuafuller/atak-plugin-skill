---
name: atak-plugin
description: Use when building, debugging, testing, or shipping an ATAK (Android Team Awareness Kit) plugin - scaffolding from the SDK template, getting it to actually load on a device, running instrumented tests inside ATAK, and preparing a submission to TAK's third-party signing pipeline. Triggers on ATAK plugin work, "plugin will NOT load", "signature mismatch", plugin manager Incompatible, takdev, plugintemplate, or connectedAndroidTest hanging.
trigger: /atak-plugin
---

# ATAK plugin development

Measured against **ATAK-CIV 5.8.0.1** and its SDK on an Android 14 emulator.
Re-verify version-specific claims on a different ATAK release.

## Four facts that cost the most time

Each produces a failure that looks like something else.

1. **ATAK compares a plugin's signing certificate against its own.** A plugin
   signed with the SDK's `android_keystore` will not load into the release ATAK
   from tak.gov or the Play Store. Install the `atak.apk` at the root of the SDK
   instead — same version, same key, red `DEVELOPER BUILD` watermark.
2. **Installing the APK is not enough.** A sideloaded plugin becomes visible to
   the plugin manager only when a copy is in
   `/sdcard/atak/support/apks/sideloaded/` **and** you run Sync Packages.
3. **Loading is a separate action from installing.** After syncing the row says
   `Not loaded`; tap it and choose **Load**.
4. **The `com.atakmap.app.component` activity in the manifest is what makes the
   plugin discoverable.** Remove it and the plugin is invisible, with no error.

## First move on any "it does not work"

Read the logcat line, not the UI:

```bash
adb logcat | grep -E "AtakPluginRegistry|PluginValidator"
```

| Log line | Meaning | Fix |
| --- | --- | --- |
| nothing for your package | not discovered | missing `com.atakmap.app.component` activity or `plugin-api` meta-data |
| `signature mismatch[pkg]` then `will NOT load` | cert does not match ATAK's | install the SDK's `atak.apk` (fact 1) |
| `api matches` but `will NOT load` / `!should load` | discovered and compatible, not enabled | sync, tap row, **Load** |
| `SDK skipping signature check[pkg]` | on the developer build | expected; the good path |
| `Successfully loaded plugin descriptor` | `plugin.xml` parsed, `impl=` resolved | — |
| `Loaded <class>` + `addPluginIcon` | fully live | — |

The manager's **Incompatible** almost always means signature, not API version,
despite the dialog mentioning software versions.

## Java or Kotlin — the template is not the answer

The SDK's template is three Java files, and it is easy to read that as ATAK
being a Java-only platform. It is not. Measured against 5.8.0.1:

| Check | Result |
| --- | --- |
| Kotlin runtime in `atak.apk` | present — `kotlinx_coroutines_android`, `kotlinx_coroutines_core`, `kotlin.reflect` service loaders |
| Kotlin in `main.jar`, the API you compile against | **4,384** entries |
| `.kt` files in the template | none |
| Kotlin awareness in the template's `build.gradle` | yes — it configures `compile<Flavour>ReleaseKotlin` when that task exists, and carries a `kotlinx-serialization-core` resolution strategy |

So ATAK ships Kotlin, its own API surface is substantially Kotlin, and the
SDK's build already anticipates Kotlin plugins. **Choose the language on its
merits; do not default to Java because the template did.** Kotlin and Java
sources mix freely in one module, so an existing Java plugin can add Kotlin
without a rewrite.

If you go Kotlin, carry this across:

- **`-Xsam-conversions=class` on release builds.** The template forces it, and
  the reason matters: Kotlin 1.5+ defaults to `indy`, compiling SAM conversions
  as `invokedynamic`. The SDK overriding that for release only is a strong
  signal the `indy` form does not survive proguard repackaging and ATAK's
  plugin classloader — so it works in debug and fails once released, which is
  the worst shape a defect can take.
- **The plugin bundles `kotlin-stdlib` while ATAK already loads Kotlin.**
  Version skew between the two is plausible and must be checked on device.
- **Verify a Kotlin unit in the *release* variant on a device** before
  committing to it. Debug proves nothing about the repackaged build.

Where Kotlin pays for itself is long-running, cancellable, progress-reporting
work — downloads, tiling, anything with a lifecycle. Structured concurrency
prevents by construction the bug where a superseded task's callbacks keep
driving the UI. Where it pays for nothing is primitive-array hot loops, such as
a protobuf parser, where the two languages generate the same thing.

## The loop

```bash
cp -r "$ATAK_SDK/samples/plugintemplate" workspace/MyPlugin
cd workspace/MyPlugin && cp template.local.properties local.properties
# local.properties: sdk.dir=/opt/android-sdk
#                   sdk.path=/opt/atak-sdk
#                   takdev.plugin=/opt/atak-sdk/atak-gradle-takdev.jar
```

Build, install, stage for sideload, then on device:
**Tools → Plugins → sync → tap row → Load**.

**Rename the template before writing any code.** Renaming later is strictly
worse. Change together: `namespace` in `app/build.gradle`, the java package
dirs and `IPlugin` class, `impl=` in `assets/plugin.xml`, `app_name`/`app_desc`
in `strings.xml`, and `rootProject.name` in `settings.gradle` (which drives the
APK name and the proguard `-repackageclasses` line). Then `adb uninstall` the
old package **and** delete its APK from the sideload folder, or Sync Packages
lists a phantom product and ATAK reports a signature failure for a package that
no longer exists.

## Repo layout

The container is one repository; **each plugin is its own, and none of them
nests inside another.** Plugin repos sit side by side in a directory the
container mounts at `/work`, so several people — or several agents — can work
on different plugins, or different `git worktree`s of one plugin, at once.

```
PLUGINS_DIR/               ->  /work
  atak-plugin-maproom/
  atak-plugin-weather/
  my-plugin.worktrees/fix-resume/
```

The container's helper scripts (`doctor`, `deploy`, `instrument`, `adb-bridge`)
belong to the container, are mounted read-only at `/opt/tak-bin`, and are on
`PATH`. Run `doctor` first, always: it checks the SDK layout, toolchain, adb
bridge, emulator image, and whether the ATAK on the device will accept plugins
signed with your keystore.

If a plugin did end up nested inside another repo, `git subtree split
--prefix=<path>` extracts it with its history intact.

## Licence boundary — decide this before the first commit, not before the first push

The TAK licence permits deriving applications from the SDK and **forbids
copying, publishing or distributing the SDK itself**. Scaffolding a plugin from
`samples/plugintemplate` copies SDK files into your working tree, and if you
commit them, your repository now contains SDK material.

This is easy to miss and expensive to undo. A plugin scaffolded from the
template can carry dozens of byte-identical SDK files — the gradle scripts,
sample resources, the espresso archives, the typst user-manual scaffolding —
and history is what gets published later, so a private repo does not protect
you. Removing them afterwards means rewriting every commit.

Check what you are about to commit:

```bash
for f in $(git ls-files); do
  m=$(find "$ATAK_SDK" -name "$(basename "$f")" -type f | head -1)
  [ -n "$m" ] && cmp -s "$f" "$m" && echo "SDK material: $f"
done
```

Gitignore anything that matches and copy it from `$ATAK_SDK` at build time.

## References — read the one you need

| File | Read it when |
| --- | --- |
| `references/environment.md` | Setting up a build environment, or adb/Gradle is misbehaving in a container. **The two hangs live here.** |
| `references/testing.md` | Writing or running tests. Espresso wiring, the `_modApk` trap, getting a plugin context. |
| `references/extension-points.md` | Deciding *how* the plugin hooks into ATAK. Import, tile formats, styles, elevation, file layout. |
| `references/shipping.md` | Preparing a Third Party Pipeline submission. |
| `references/atak-behaviour.md` | Designing around how ATAK treats maps and imports. Explains behaviour that looks like bugs. |
| `references/android-gotchas.md` | Anything that works on the JVM and fails on device. |

## Rules of thumb

- **Verify against ATAK's own classes, not your own.** For anything ATAK has to
  accept, assert through the class ATAK uses (e.g. `MobacMapSourceFactory` for
  map sources). A test using only your parser goes green while shipping files
  ATAK refuses.
- **Anything that only runs on device must be tested on device.** Android's
  platform classes differ from the JVM's in ways that fail silently — see
  `references/android-gotchas.md`.
- **Use a `google_apis` emulator image.** `aosp_atd` renders a black
  framebuffer while `uiautomator` still reports a full view tree, so a UI
  script "taps" successfully and changes nothing.
