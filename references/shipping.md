# Shipping: getting a signed APK

Real users run the release ATAK, which will not load an unsigned plugin. TAK's
**Third Party Pipeline** builds and signs third-party plugins — upload a zip at
<https://tak.gov/user_builds/agreement>. It is an ephemeral build service; the
government states it does not monitor or retain submitted source, and ownership
does not transfer. Plugins it signs are marked in ATAK's UI as
third-party-signed rather than TPC-built.

## Checklist — all verifiable locally

- [ ] Zip contains a **single root folder**; that folder's name becomes the APK
      name. Zip from the parent directory.
- [ ] **`./gradlew assembleCivRelease` succeeds.** The discriminating check —
      run it on day one, not at submission time. It also runs
      `lintVitalCivRelease`, which catches API-level mistakes your unit tests
      cannot.
- [ ] The **release variant loads on a device**, not just builds. R8 minifies
      it, and `plugin.xml` names your entry class as a string, so a missing keep
      rule fails only here. Expect `Loaded <class>` and `addPluginIcon` in
      logcat.
- [ ] SDK referenced only through `atak-gradle-takdev`. Published guidance says
      `2.+` for ATAK 4.2+, but also that a recent plugintemplate clone already
      complies; the 5.8 template ships `takdevVersion = '3.+'` — leave it.
- [ ] `-repackageclasses` names your plugin, not `PluginTemplate`. The template
      derives it from `rootProject.name`, so setting that is enough.
- [ ] `com.atakmap.app.component` activity present in `AndroidManifest.xml`,
      with its `intent-filter`. Present in the template; do not remove it.
- [ ] Gradle scripts included in the archive.

## Pre-submission verification

The documented dry run needs credentials for `artifacts.tak.gov`, which is
restricted to USG federal and military personnel:

```bash
./gradlew -Ptakrepo.force=true -Ptakrepo.url=https://artifacts.tak.gov/artifactory/maven \
          -Ptakrepo.user=<user> -Ptakrepo.password=<pass> assembleCivRelease
```

Without those, a clean local `assembleCivRelease` plus a verified release-variant
load is the strongest signal available. TPC support will ask for the output of
that command first.

## Licensing context

ATAK-CIV is EAR99 and open source. TAK-MIL is restricted to USG military use.
Use outside test and evaluation needs your organisation's ATO paperwork.
