# Read the source

ATAK-CIV's source is published under **GPL-3.0** — roughly 4,200 Java files,
including the whole map engine. Almost every question about how ATAK behaves is
answered there in less time than it takes to design an experiment.

Two places, and the difference matters when a claim is version-specific:

| Source | What you get |
| --- | --- |
| <https://tak.gov>, as `atak-civ-client` | The current release, if your account has sufficient permissions |
| <https://github.com/TAK-Product-Center/atak-civ> | A public mirror, released on a delay |

If a behaviour does not match what you read, check which of the two you have
and against which ATAK build — the delayed mirror can lag the binary you are
testing against.

This should be the first move, not the last.

## What it answers that nothing else does

Method semantics. A signature tells you the types; the source tells you the
meaning, and that is usually where the day goes.

A worked example. `MapController.zoomTo(double scale, boolean animate)` looks
like it takes a zoom level or a resolution. It takes **map scale**. Passing
`30.0` — a plausible "30 metres per pixel" — asks for something extremely
zoomed in. Against an archive holding z4–14, every tile request then misses,
and the map renders blank with no error at any layer. Hours went into that,
across several sessions, on the theory that the tiles or the registration were
wrong. Both were fine.

The answer was two greps:

```bash
grep -rn "public void zoomTo" takkernel/engine/src/android/java/com/atakmap/map/AtakMapController.java
# → "Set the map scale (instant) @param scale The new map scale"

grep -rn "mapResolutionAsMapScale" takkernel/engine/src/android/java/com/atakmap/map/AtakMapView.java
# → public double mapResolutionAsMapScale(double resolution)
```

The conversion you need is already a method on the class you already have.

## Where things are

| Looking for | Start at |
| --- | --- |
| Map camera, pan, zoom, scale | `takkernel/engine/src/android/java/com/atakmap/map/AtakMapController.java`, `AtakMapView.java` |
| Layers, imagery selection | `com/atakmap/map/layer/raster/`, `MobileImageryRasterLayer2` |
| Tile readers and formats | `com/atakmap/map/layer/raster/tilereader/`, `mobac/`, `mobileimagery/` |
| Plugin loading and validation | `atak/ATAK/app/src/main/java/com/atak/plugins/impl/` |
| The app's own UI | `atak/ATAK/app/src/main/java/com/atakmap/android/` |

`grep -rn` over the tree is fast enough that narrowing first is rarely worth
it.

## Reading is not copying

GPL-3.0 is a **copyleft** licence. Reading it to understand behaviour is
exactly what publication is for. Pasting its code into a plugin makes that
plugin subject to GPL-3.0, which is a licensing decision with consequences for
how the plugin can be distributed — including through the Third Party
Pipeline.

So: read it, understand it, then write your own. If you find yourself wanting
a non-trivial block verbatim, that is a licensing decision to take
deliberately, not a shortcut to take quietly.

Keep it separate from anything you publish. Do not vendor it into a plugin
repository "for reference"; a clone on disk is enough.

## This replaces guessing at the dex

An earlier instinct here was to infer ATAK's internals from the shipping APK —
obfuscated class names, methods matched by shape. That approach found
`dispatchStyleRegistered`, a listener notifier, and called it as if it were a
registry: it reported success and changed nothing, which is worse than
failing.

It is also the thing the SDK licence forbids. Reverse engineering is both
prohibited and unnecessary when the source is published. If a fact about ATAK
cannot be established from the source or from observed runtime behaviour, it
should not go in this skill.
