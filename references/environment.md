# Build environment, and the two things that hang

The tooling this skill assumes — a pinned toolchain, `doctor`, `deploy`,
`instrument`, and an `AGENTS.md` describing how to run the whole loop without a
human — lives in <https://github.com/joshuafuller/atak-plugin-dev>.
**This skill carries the knowledge; that repo carries the tools.**

## Contents

- [What a build needs](#what-a-build-needs)
- [Hang 1: `connectedAndroidTest` sits for ten minutes](#hang-1-connectedandroidtest-sits-for-ten-minutes)
- [Hang 2: ATAK's first-run permission walkthrough](#hang-2-ataks-first-run-permission-walkthrough)
- [Swapping release ATAK for the developer build](#swapping-release-atak-for-the-developer-build)
- [Container portability](#container-portability)
- [Other build-time noise that is not a problem](#other-build-time-noise-that-is-not-a-problem)
- [Emulator states that look like build failures](#emulator-states-that-look-like-build-failures)

## What a build needs

| | |
| --- | --- |
| JDK | 17 |
| Android | platforms 36 and 34, build-tools 36.0.0, platform-tools |
| Gradle | 8.14.3 (the 5.8 template's wrapper) |
| ATAK SDK | mounted **writable** — takdev writes `mapping.txt` into it |

The SDK is licensed; download your own from tak.gov. It contains `atak.apk`
(the developer ATAK), `android_keystore`, `atak-gradle-takdev.jar`, `samples/`
and `espresso/`.

Legacy `aapt` (not `aapt2`) must exist in build-tools — the espresso
`modApkTask.gradle` looks for it. Present in 35.0.0 and 36.0.0.

No NDK. ATAK's own native libraries are built with NDK 12b; match it if you
link against them.

## Hang 1: `connectedAndroidTest` sits for ten minutes

**Symptom:** the task starts and produces nothing. Eventually the log shows
`[DeviceMonitor]: Cannot reach ADB server, attempting to reconnect` and
`'adb start-server' failed -- run manually if necessary`.

**Cause:** `ADB_SERVER_SOCKET` is honoured by the adb *command line* only.
Gradle's Android plugin runs its own device monitor, which shells out to
`adb start-server` and then dials `127.0.0.1:5037` itself. Pointed at a remote
address, the start attempt tries to bind an address the container does not own,
fails, and the task retries.

**Fix:** give the container a genuinely local endpoint and stop setting
`ADB_SERVER_SOCKET`. A ~60-line TCP forwarder from `127.0.0.1:5037` to the
host's adb server is enough (`bin/adb-bridge` in the atak-plugin-dev repo);
`socat TCP-LISTEN:5037,fork,reuseaddr TCP:host.docker.internal:5037` does the
same job.

On the host, adb only listens off-loopback when started explicitly — a client
auto-spawning the server will not do it:

```bash
adb -a -P 5037 server nodaemon    # leave running
```

**If you write your own forwarder, clear the connect timeout.**
`socket.create_connection(..., timeout=10)` leaves that timeout on the socket,
where it then applies to every read. adb connections are long-lived and often
quiet — `am instrument` streams nothing for the length of a test run — so the
connection is torn down mid-command. `adb devices` survives, a twenty-second
test run does not, and the result looks intermittent rather than systematic.
The symptom is an instrumented run with **no output at all** and adb exiting
255.

Even with the bridge, Gradle's **Unified Test Platform** still fails with
`DEVICE_PROVIDER_CONFIG_ERROR` / `Failed to initialize AndroidDebugBridge`.
Drive instrumentation over adb instead — see `testing.md`.

## Hang 2: ATAK's first-run permission walkthrough

A UI script tapping **Allow all the time** opens a settings sub-screen that
re-presents itself indefinitely. Grant directly instead:

```bash
adb shell pm grant com.atakmap.app.civ android.permission.ACCESS_FINE_LOCATION
adb shell pm grant com.atakmap.app.civ android.permission.ACCESS_BACKGROUND_LOCATION
adb shell pm grant com.atakmap.app.civ android.permission.POST_NOTIFICATIONS
adb shell appops  set com.atakmap.app.civ MANAGE_EXTERNAL_STORAGE allow
```

## Swapping release ATAK for the developer build

They share a package name and differ in signing key, so one replaces the other
and **app data is wiped**. Files under `/sdcard/atak` survive (top-level
directory) but registrations and preferences do not — previously imported maps
are on disk and not showing.

If the previous install had encrypted databases, first launch demands a
passphrase you do not have: choose **Remove and Quit**, confirm, relaunch.

```bash
adb uninstall com.atakmap.app.civ
adb install -r "$ATAK_SDK/atak.apk"
```

## Container portability

- `network_mode: host` does **not** share the host's network namespace on
  WSL2/Docker Desktop, which is why the bridge is needed at all.
- `host.docker.internal` is not resolvable under host networking on native
  Linux Docker without `extra_hosts: ["host.docker.internal:host-gateway"]`.
  `127.0.0.1` is the fallback there.
- `local.properties` is per-project and not committed. A fresh clone fails
  confusingly without it.

## Other build-time noise that is not a problem

| Message | Meaning |
| --- | --- |
| `error occured, using revision of 1` | the template's git-revision helper, when the project is not a git repo |
| `fatal: not a git repository` | same |
| `WARNING: Using flatDir should be avoided` | how the espresso AAR is resolved |
| `skipping user manual generation` | typst not installed |

## Emulator states that look like build failures

| Symptom | Cause |
| --- | --- |
| `adb install` fails with an empty message, or `Failure calling service package: Broken pipe (32)` | The package service is wedged after many install cycles. `adb reboot`, wait for `sys.boot_completed`. |
| An instrumented run returns no output, immediately after a previous run | Racing a still-shutting-down ATAK. Force-stop `com.atakmap.app.civ` and pause before instrumenting — `ATAKStarter` launches it itself. |
| A UI script reports successful taps that change nothing | Either an `aosp_atd` system image (black framebuffer, live view tree) or tapping controls with `enabled="false"`. ATAK gates several dialogs until scrolled to the end. |
