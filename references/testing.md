# Testing an ATAK plugin

Split by what the code touches.

- **Neither Android nor ATAK API** → local JVM tests,
  `./gradlew testCivDebugUnitTest`. Seconds; gate every build on them. Parsing,
  validation, path handling.
- **Either one** → the SDK's Espresso framework. Plugins run inside ATAK's
  process, so this is the only way to see real behaviour.

The template ships an `ExampleTest` but **no junit dependency**, so add
`testImplementation 'junit:junit:4.13.2'` before the first unit test compiles.

## Contents

- [Wiring the Espresso framework](#wiring-the-espresso-framework)
- [The `_modApk` trap](#the-modapk-trap)
- [Running without Gradle's UTP](#running-without-gradles-utp)
- [Test class shape](#test-class-shape)
- [Two harness quirks](#two-harness-quirks)
- [The harness mutates ATAK's preferences](#the-harness-mutates-ataks-preferences)
- [Espresso against a plugin pane](#espresso-against-a-plugin-pane)
- [Assert through ATAK, not through your own code](#assert-through-atak-not-through-your-own-code)
- [Fail at the real cause](#fail-at-the-real-cause)
- [Flakiness](#flakiness)
- [Debugging](#debugging)

## Wiring the Espresso framework

Copy `espresso/` from the SDK so it sits beside `app/`. Then rename the AAR:
the directory ships `ATAKPluginTests-debug.aar` but `testSetup.gradle` resolves
`atakplugintests-debug` — case-sensitive on Linux, so the flatDir lookup fails
until you rename it.

**Do not add `apply from: espresso/testSetup.gradle`.** The takdev gradle plugin
applies it automatically as soon as the directory exists. A second apply fails
the build with `Cannot add task 'clearScreenshots' as a task with that name
already exists`.

## The `_modApk` trap

`assembleCivDebugAndroidTest` does **not** run `packageCivDebugAndroidTest_modApk`
— only `connectedCivDebugAndroidTest` depends on it.

That task rewrites the instrumentation manifest's `targetPackage` to
`com.atakmap.app.civ` and re-signs. Without it, tests run against the plugin
package instead of ATAK, and every test dies at startup with:

```
java.lang.NoClassDefFoundError: Failed resolution of:
  Lcom/atakmap/android/preference/AtakPreferences;
  at com.atakmap.android.test.helpers.ATAKStarter.testRunStarted
```

The giveaway is the `DexPathList` in that stack trace: it lists the test APK
and the plugin APK, and **not** ATAK's.

## Running without Gradle's UTP

UTP fails with `Failed to initialize AndroidDebugBridge` against a forwarded adb
socket (see `environment.md`). Everything the Gradle task adds is assembling,
`_modApk`, and installing — so do that with Gradle and drive the rest with adb:

```bash
./gradlew assembleCivDebug packageCivDebugAndroidTest_modApk
adb install -r app/build/outputs/apk/civ/debug/*.apk
adb install -r app/build/outputs/apk/androidTest/civ/debug/*.apk
adb shell am instrument -w \
    -e listener com.atakmap.android.test.helpers.ATAKStarter \
    <your.package>.test/com.atakmap.android.test.helpers.NoFinishAndroidJUnitRunner
```

`am instrument` **exits 0 even when tests fail**, so decide from the output:
`OK (n tests)` for success; `FAILURES!!!`, `INSTRUMENTATION_CODE: 0`, or
`Process crashed` for failure. `bin/instrument` in the atak-plugin-dev repo
wraps all of this.

## Test class shape

```java
public class MyTest extends ATAKTestClass {
    @BeforeClass public static void setup() throws Exception {
        dismissPendingDialogs();                       // see below
        helper.installPlugin("My Plugin", "my.package");
        ClassLoaderReplacer.fixClassLoaderForClass(MyTest.class, "my.package");
    }
    @AfterClass public static void teardown() throws Exception {
        ClassLoaderReplacer.restoreLoader(MyTest.class);
    }
}
```

Without `fixClassLoaderForClass` you get `ClassDefNotFoundException` on your own
classes. Test order is not guaranteed, so each test must restore app state.

## Two harness quirks

**`HelperFunctions.getLoadedPlugin(pkg)` can return null** even after ATAK has
logged `addPluginIcon` for your plugin. If all you need is the plugin's context
— for its assets or resources — build it yourself:

```java
pluginContext = InstrumentationRegistry.getInstrumentation().getTargetContext()
        .createPackageContext("my.package",
                Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY);
```

**Reinstalling the plugin APK makes ATAK offer to load it on next start.** That
dialog is modal, so the harness's first tap on the nav menu throws
`NoMatchingViewException` and every test in the class errors during setup.
Clear it first, by resource id — these are framework AlertDialogs whose positive
button is `android:id/button1` whatever it is labelled — and *wait*, because the
dialog arrives after ATAK finishes starting:

```java
UiDevice device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
for (int i = 0; i < 8; i++) {
    UiObject2 positive = device.wait(Until.findObject(By.res("android", "button1")), 4000);
    if (positive == null) return;
    positive.click();
    device.waitForIdle(2000);
}
```

## The harness mutates ATAK's preferences

`ATAKStarter` sets ATAK up for automation, and those changes **persist after
the run**. The visible one is `nav_orientation_right=false`, which moves ATAK's
toolbar to the left side of the screen and stays there. It looks like a plugin
bug or a stray tap; it is neither.

```bash
adb shell am force-stop com.atakmap.app.civ
adb shell 'su 0 sed -i "s|nav_orientation_right\" value=\"false|nav_orientation_right\" value=\"true|" \
    /data/data/com.atakmap.app.civ/shared_prefs/com.atakmap.app.civ_preferences.xml'
```

The general point: a device you run instrumented tests on is no longer in a
user's configuration. Do not measure UI behaviour on it without checking what
the harness changed first.

## Espresso against a plugin pane

Views inflated by `PluginLayoutInflater` land in ATAK's hierarchy, so `onView`
finds them normally. Two things bite:

- **`withText`/`withSubstring` go ambiguous fast.** Once an outcome message
  says "Installed 3", `withSubstring("Install")` matches it *and* the button.
  Match controls by `withId`; the plugin's `R` is available from `androidTest`
  in the same module.
- **Rows repeat.** Every source from one provider may share a zoom range or an
  "on device" marker. Scope to the row: `hasSibling(withText(name))` for a
  direct sibling, or `withParent(hasDescendant(withText(name)))` when the view
  sits beside the block containing the name rather than beside the name itself.

Espresso cannot `scrollTo` inside a `ListView`, so assert on rows that are on
screen — sort order is a design decision you can lean on.

## Assert through ATAK, not through your own code

The single highest-value habit. For anything ATAK must accept, assert with the
class ATAK itself uses. A temp-directory unit test over your own writer and your
own parser goes green while shipping files ATAK refuses.

```java
MobacMapSource accepted = MobacMapSourceFactory.create(writtenFile);
assertEquals(ourName, accepted.getName());
assertEquals(ourMinZoom, accepted.getMinZoom());
```

Comparing ATAK's reading against the values you displayed also catches the case
where the install works but your UI described it wrongly.

Useful for locating things on device: `FileSystemUtils.getItem("imagery")`
resolves a path under the ATAK root.

## Fail at the real cause

Split "did the input load" from "did the operation work". A test asserting only
`installed.size() == 33` reports `expected:<33> but was:<0>` when the actual
fault is that an asset did not parse. A separate test that asserts the input
parsed, and prints the collected problems in its failure message, names the
cause directly.

## Flakiness

```bash
adb shell settings put global window_animation_scale 0
adb shell settings put global transition_animation_scale 0
adb shell settings put global animator_duration_scale 0
```

## Debugging

Plugins have no activity, so attach to the running `com.atakmap.app` process
rather than launching. Breakpoints in plugin source resolve once attached.
Debugging instrumented tests is not known to work in this framework.
