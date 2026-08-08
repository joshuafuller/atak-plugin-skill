# How ATAK treats maps, and why

Behaviour measured on **release** ATAK-CIV 5.8.0.1. Much of it looks like bugs
until you notice the assumption underneath.

## The assumption

**ATAK assumes a provisioning authority exists** — a geospatial cell or unit
tech who decides the basemap, prepares the data, images the devices, and is
reachable when it breaks. ATAK is the consumer end of a supply chain whose other
end is staffed.

That is a reasonable trade for a staffed force and a defect for a volunteer SAR
team or a fire district. It also tells you where a plugin adds the most value:
be the provisioning authority, supplied as software.

## Measured behaviour worth designing around

| Behaviour | Consequence |
| --- | --- |
| Any non-terrain `.mbtiles` is claimed by `ImportGRGSort`, which outranks every other resolver | A handed-over tileset becomes an **overlay**, not a basemap. The default assumes your basemap came from the supply chain |
| A Data Package containing map definitions extracts and registers **nothing**, silently | Mission Package extraction files declared contents into a sandbox the imagery resolvers never inspect. The one sharing mechanism users already understand is the one that fails for maps |
| The resolver prompt appears on file imports, never on URL imports | A URL is assumed to come from someone who knew what they were sending |
| Bundled vector tile styles are compiled into the APK | Only in-process code can register another — which is exactly what a plugin is |
| No progress indication on large imports; failures are silent | Assumes an operator who can read a log, or a tech to call |
| Registration depends on the route taken, not the file | The same file becomes a basemap or an overlay depending on how it arrived |

## The plain-zip pattern

A **plain zip of MOBAC source XML**, imported through ATAK's Import feature, is
the distribution method that works — the sources register automatically.

Adding a `MANIFEST/manifest.xml` breaks it: that presence is what makes
`MissionPackageExtractorFactory` choose the Mission Package extractor over the
plain zip extractor, and the imagery resolvers never see the contents.

[ATAK-Maps](https://github.com/joshuafuller/ATAK-Maps) has shipped exactly this
shape at scale. Two independent routes to the same answer.

## Stream-then-cache is the normal offline workflow

For raster, users pick a streaming source, open Map Manager, choose **Download**,
draw a region and pick zoom levels; tiles persist in SQLite. So "one streaming
source, cache the area you need" is current practice, not a future idea. Whether
the same region download works for a **vector** streaming source is open.

## Where the gaps actually are

Acquisition is solved and popular. These are not:

- **Scope** — nothing helps the user choose region and zoom depth, and depth
  cannot be added later.
- **Verification** — no way to confirm a cache is sufficient before losing comms.
- **Peer sharing** — the cache lives on one device; the person beside you starts
  from scratch.
- **Self-hosting** — a unit with its own imagery has no path that looks like the
  usual one.

## Design consequences

1. Registration should be deterministic, not route-dependent.
2. Failure must be loud. Every silent failure here is unrecoverable for a user
   without a support tech.
3. Scope belongs to the user at acquisition time.
4. Verification must be possible before departure, offline, as an explicit step.
5. Sharing must work for maps, because that is how this community moves
   everything else.
6. No new infrastructure to stand up — it must work from a laptop and a file.

## Vector tile authoring notes

ATAK's bundled OMT style is a specification you can author against: 111 layers,
half of them `transportation`. Attribute values outside its filters are carried
to the device and never drawn.

- `boundary` is filtered to `admin_level` 2 and 4. **County lines never draw**,
  which matters for SAR, fire and law enforcement.
- POI icons are `{class}_11` looked up in the sprite sheet, which uses
  **hyphens** (`fast-food`); OMT emits underscores. Renaming recovers a
  meaningful share of otherwise iconless POIs.
- `building` and `housenumber` are unfiltered; the only control is inclusion.
- Populate `rank` on POIs and places or zoom tiering misbehaves.
