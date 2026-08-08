# Java or Kotlin

The SDK's template is three Java files, and it is easy to read that as ATAK
being a Java-only platform. It is not.

## What ATAK actually ships

Measured against ATAK-CIV 5.8.0.1 and its SDK:

| Check | Result |
| --- | --- |
| Kotlin runtime inside `atak.apk` | present — `kotlinx_coroutines_android`, `kotlinx_coroutines_core`, and `kotlin.reflect` service loaders |
| Kotlin in `main.jar`, the API you compile against | **4,384** entries |
| `.kt` files in `samples/plugintemplate` | none — three Java files |
| Kotlin awareness in the template's `build.gradle` | yes — it configures `compile<Flavour>ReleaseKotlin` when that task exists, and carries a `kotlinx-serialization-core` resolution strategy |

How to re-check on a different version:

```bash
unzip -l "$ATAK_SDK/atak.apk" | grep -ci kotlin
unzip -l "$ATAK_SDK/main.jar" | grep -ci kotlin
grep -rn -i kotlin "$ATAK_SDK/samples/plugintemplate"/app/build.gradle
```

So ATAK ships Kotlin, its own API surface is substantially Kotlin, and the
SDK's build already anticipates Kotlin plugins — it simply does not enable one
by default. **Choose the language on its merits.** Kotlin and Java sources mix
freely in one module, so an existing Java plugin can add Kotlin without a
rewrite.

## If you go Kotlin

**`-Xsam-conversions=class` on release builds.** The template forces exactly
this, in an `afterEvaluate` block guarded by a `try`:

```groovy
tasks.named("compile" + getCommandFlavor() + "ReleaseKotlin") {
    kotlinOptions { freeCompilerArgs += "-Xsam-conversions=class" }
}
```

Kotlin 1.5 and later default to `indy`, compiling SAM conversions as
`invokedynamic`. The SDK overriding that **for release only** is a strong
signal that the `indy` form does not survive proguard repackaging and ATAK's
plugin classloader. A plugin missing this flag can work in debug and fail once
released, which is the worst shape a defect can take.

**The plugin bundles `kotlin-stdlib` while ATAK already loads Kotlin.** Version
skew between the host runtime and the plugin's bundled stdlib is plausible.
Check it on a device rather than assuming it.

**Verify a Kotlin unit in the *release* variant, on a device**, before
committing to the language. Debug proves nothing about the repackaged build —
`-repackageclasses` interacts with Kotlin metadata.

## Where each pays

Kotlin earns its place in long-running, cancellable, progress-reporting work —
downloads, tiling, anything with a lifecycle. Structured concurrency prevents
by construction the bug where a superseded task's callbacks keep driving the UI
after a newer task has replaced it.

It earns nothing in primitive-array hot loops, such as a protobuf parser, where
both languages generate the same thing and the Java is often already written
and tested.
