---
name: atak-plugin
description: Use when building, debugging, testing, or shipping an ATAK (Android Team Awareness Kit) plugin - scaffolding from the SDK template, getting it to actually load on a device, running instrumented tests inside ATAK, and preparing a submission to TAK's third-party signing pipeline. Triggers on ATAK plugin work, "plugin will NOT load", "signature mismatch", plugin manager Incompatible, takdev, plugintemplate, or connectedAndroidTest hanging.
---

# ATAK plugin development

Measured against **ATAK-CIV 5.8.0.1** and its SDK on an Android 14 emulator.
Re-verify version-specific claims on a different ATAK release.

## What you have

Know these before reaching for an experiment; most questions are already
answered by one of them.

| | What it is | Use it for |
| --- | --- | --- |
| **The published source** | ATAK-CIV under GPL-3.0, ~4,200 Java files including the map engine. tak.gov (current, permissioned) or `github.com/TAK-Product-Center/atak-civ` (public, delayed) | **Read this first.** Exact signatures and, more importantly, what they mean |
| **The SDK** | `atak.apk`, keystore, `main.jar`, espresso, samples. Licensed, mounted at `$ATAK_SDK`, never committed | Building, and the developer ATAK that will load your plugin |
| **The dev container** | `github.com/joshuafuller/atak-plugin-dev` — pinned toolchain, `doctor`, `deploy`, `instrument` on `PATH`, and an `AGENTS.md` for unattended runs | Run `doctor` first, always. The tooling lives there, not here |
| **An emulator** | `google_apis` image — never `aosp_atd` | Everything, unattended |
| **This skill** | A measurement record, not a manual | Correct it when it misleads you |

**Read the source before designing an experiment.**
`MapController.zoomTo(double)` takes ATAK's map *scale*, not resolution. A
plausible "30 metres per pixel" asks for something extremely zoomed in: every
tile request misses and the map renders blank, with no error anywhere.
`AtakMapView.mapResolutionAsMapScale()` converts — one grep each, where finding
it by experiment cost hours.

**Permitted:** reading the GPL-3.0 source, observing ATAK at runtime, listing
archive entries, reflection over loaded classes, shipping plugins derived from
the SDK. **Not permitted:** decompiling or disassembling the APK, SDK jars or
AARs; republishing SDK files. `references/licensing.md` quotes both licences
and links every copy.

## Four facts that cost the most time

Each produces a failure that looks like something else.

1. **ATAK compares a plugin's signing certificate against its own.** A plugin
   signed with the SDK's keystore will not load into the release ATAK; install
   `$ATAK_SDK/atak.apk` instead. The manager says "Incompatible", which sounds
   like a version problem and is not.
2. **Installing the APK is not enough.** It is visible to the plugin manager
   only with a copy in `/sdcard/atak/support/apks/sideloaded/` **and** a sync.
3. **Loading is separate from installing.** After syncing, tap the row and
   choose **Load**.
4. **The `com.atakmap.app.component` activity makes the plugin discoverable.**
   Remove it and the plugin is invisible, with no error.

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

The template is three Java files, but ATAK is not Java-only: `main.jar` has
4,384 Kotlin entries and the template's gradle already configures a Kotlin
compile task. Choose on merits; the two mix in one module. One trap — the SDK
forces `-Xsam-conversions=class` on *release* builds, so the default form
fails only after release. See `references/language.md`.

## The loop

```bash
cp -r "$ATAK_SDK/samples/plugintemplate" <plugins-dir>/MyPlugin
cd MyPlugin && cp template.local.properties local.properties
#   sdk.dir=/opt/android-sdk   sdk.path=/opt/atak-sdk
#   takdev.plugin=/opt/atak-sdk/atak-gradle-takdev.jar
deploy MyPlugin      # then on device: Tools -> Plugins -> sync -> row -> Load
```

**Rename the template before writing any code.** Together: `namespace`, the
java package dirs and `IPlugin` class, `impl=` in `assets/plugin.xml`,
`app_name`/`app_desc`, and `rootProject.name` (which drives the APK name and
the proguard `-repackageclasses` line). Then `adb uninstall` the old package
**and** delete its APK from the sideload folder, or Sync Packages lists a
phantom product and reports a signature failure for a package that no longer
exists.

## Repo layout and the licence boundary

Each plugin is its own repository, none nested; the container mounts the parent
so several agents can work at once. Only the device is exclusive.

**Settle the licence boundary before your first commit.** Scaffolding from the
template copies SDK files into your tree — one real plugin carried 45 — and
history is what gets published, so the fix afterwards is a rewrite. Check with
the script in `references/project-setup.md`.

## References — read the one you need

| File | Read it when |
| --- | --- |
| `references/environment.md` | Setting up a build environment, or adb/Gradle is misbehaving in a container. **The two hangs live here.** |
| `references/testing.md` | Writing or running tests. Espresso wiring, the `_modApk` trap, getting a plugin context. |
| `references/extension-points.md` | Deciding *how* the plugin hooks into ATAK. Import, tile formats, styles, elevation, file layout. |
| `references/shipping.md` | Preparing a Third Party Pipeline submission. |
| `references/atak-behaviour.md` | Designing around how ATAK treats maps and imports. Explains behaviour that looks like bugs. |
| `references/android-gotchas.md` | Anything that works on the JVM and fails on device. |
| `references/reading-the-source.md` | **Where the source is and how to search it.** Before any experiment. |
| `references/licensing.md` | What the TAK and GPL-3.0 licences actually permit, quoted. |
| `references/project-setup.md` | Repo layout, and the SDK files that must never be committed. |
| `references/language.md` | Choosing Java or Kotlin, and what a Kotlin plugin must carry. |
| `references/self-improvement.md` | **The working loop, and how to correct this skill when it misleads you.** Read it the first time something here turns out to be wrong. |

## When this skill is wrong, fix it before you finish

Everything here was measured against one ATAK version on one machine on one
day, and some of it is already wrong. **When the device and this skill
disagree, the device wins and this skill is defective.**

Correct it whenever a claim turned out false, a diagnosis cost more than about
half an hour, or you needed a fact that was not here. Raise it as a pull
request, not a push — a wrong claim here is believed by every future session:

```bash
cd ~/.claude/skills/atak-plugin      # the installed skill is a git checkout
scripts/propose finding "Coverage does not extend until re-registration"
```

One finding per PR, measured, method in the body.
`references/self-improvement.md` has the loop; `CONTRIBUTING.md` the criteria.

## Rules of thumb

- **Verify against ATAK's own classes, not your own.** For anything ATAK must
  accept, assert through the class ATAK uses. A test using only your parser
  goes green while shipping files ATAK refuses.
- **Anything that only runs on device is only proven on device**, and anything
  a user sees is only proven by looking at it.
- **`am instrument` exits 0 even when tests fail.** Parse the output.
