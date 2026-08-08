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

The template is three Java files; ATAK is not a Java-only platform. Its own
`main.jar` has 4,384 Kotlin entries, `atak.apk` ships `kotlinx-coroutines` and
`kotlin.reflect`, and the template's gradle already configures a Kotlin compile
task if one exists. **Choose the language on its merits**; the two mix freely
in one module.

One trap if you do: the SDK forces `-Xsam-conversions=class` on *release*
Kotlin compilation, which means the default `indy` form does not survive
proguard repackaging and ATAK's classloader — so it works in debug and fails
once released. See `references/language.md`.

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

## Repo layout and the licence boundary

Each plugin is its own repository, and none nests inside another; the container
mounts the parent directory so several agents can work at once. Only the device
is exclusive — serialise device work, parallelise the rest.

**Settle the licence boundary before your first commit.** The TAK licence
permits deriving applications from the SDK and forbids redistributing it, and
scaffolding from the template copies SDK files into your tree — one real plugin
carried 45 of them. History is what gets published, so a private repo does not
protect you, and the fix afterwards is a history rewrite.

Both, with the check to run: `references/project-setup.md`.

## References — read the one you need

| File | Read it when |
| --- | --- |
| `references/environment.md` | Setting up a build environment, or adb/Gradle is misbehaving in a container. **The two hangs live here.** |
| `references/testing.md` | Writing or running tests. Espresso wiring, the `_modApk` trap, getting a plugin context. |
| `references/extension-points.md` | Deciding *how* the plugin hooks into ATAK. Import, tile formats, styles, elevation, file layout. |
| `references/shipping.md` | Preparing a Third Party Pipeline submission. |
| `references/atak-behaviour.md` | Designing around how ATAK treats maps and imports. Explains behaviour that looks like bugs. |
| `references/android-gotchas.md` | Anything that works on the JVM and fails on device. |
| `references/project-setup.md` | Repo layout, and the SDK files that must never be committed. |
| `references/language.md` | Choosing Java or Kotlin, and what a Kotlin plugin must carry. |
| `references/self-improvement.md` | **The working loop, and how to correct this skill when it misleads you.** Read it the first time something here turns out to be wrong. |

## When this skill is wrong, fix it before you finish

Everything here was measured against one ATAK version on one machine on one
day. Some of it is already wrong. **When the device and this skill disagree,
the device wins and this skill is defective.**

The working loop is six steps — orient (`doctor`), specify (red test), build,
verify *on device*, record the evidence, **correct the skill**. The last step
is the one that makes the next session cheaper, and the only one anybody skips.

Correct it whenever a claim here turned out false, a diagnosis cost more than
about half an hour, or you needed a fact that was not here. Do it as a pull
request rather than a push — a wrong claim here is believed by every future
session:

```bash
cd ~/.claude/skills/atak-plugin      # the installed skill is a git checkout
# ...edit...
bin/propose finding "Coverage does not extend until re-registration"
```

One finding per PR, measured with the method in the body, versions stated.
`references/self-improvement.md` has the full protocol; `CONTRIBUTING.md` has
the acceptance criteria and definition of done.

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
