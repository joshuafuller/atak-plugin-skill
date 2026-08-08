# Hooking into ATAK

All public in 5.8.

## Plugin entry point

The 5.8 template implements `gov.tak.api.plugin.IPlugin`, takes an
`IServiceController`, and in `onStart` adds a `ToolbarItem` that opens a `Pane`.

- Get the plugin's own context from `PluginContextProvider`, and call
  `pluginContext.setTheme(R.style.ATAKPluginTheme)`.
- Inflate layouts with **`PluginLayoutInflater`**, not the stock inflater — the
  plugin's resources are not ATAK's.
- Set `ToolbarItem.Builder.setIdentifier(pluginContext.getPackageName())`, or
  the toolbar cannot find the icon again after the user moves it.
- `assets/plugin.xml` names the entry class as a **string** in `impl=`. R8
  cannot see that reference, so verify the **release** variant loads, not just
  debug — a missing keep rule fails only there.

## By goal

| Goal | API |
| --- | --- |
| Import a file type | `ImporterManager.registerImporter`, `MarshalManager.registerMarshal`, `ImportFilesTask.registerExtension(".ext")` |
| Import without copying the file | `ImportExportMapComponent.getInstance().addImporterClasses(ImportInPlaceResolver.fromMarshal(...))` |
| Own the resolver decision | `ImportExportMapComponent.getInstance().addImporterClass(...)` |
| Trigger an import | broadcast `ImportReceiver.ACTION_IMPORT_DATA` with `EXTRA_URI` or `EXTRA_URI_LIST`; optional `EXTRA_CONTENT`, `EXTRA_MIME_TYPE`, `EXTRA_SHOW_NOTIFICATIONS`, `EXTRA_ZOOM_TO_FILE`. Omit content/MIME to let ATAK auto-detect |
| New tile container / renderer | `TileContainerFactory.registerSpi`, `DatasetDescriptorFactory2.register`, `GLMapLayerFactory.registerSpi` |
| Supply a vector tile style | `StyleDocumentRegistry.register(File, name, styleUri)` — registrations take precedence over a tileset's own `styleUrl` |
| Region caching | `StreamingTileClient.cache(CacheRequest, …)`, gated on the descriptor's `downloadable` flag |
| Elevation | `ElevationSourceManager.attach` |
| Custom projection | implement `ProjectionSpi`, register with `ProjectionFactory` |
| Filesystem / DB interception | `IOProviderFactory.registerProvider` in `onCreate`, unregister in `onDestroy` |
| Read a MOBAC map source the way ATAK does | `MobacMapSourceFactory.create(File)` → `MobacMapSource` (`getName`, `getMinZoom`, `getMaxZoom`, `getTileType`, `getSRID`, `getBounds`) |

## Where files go on device

ATAK reads these paths directly, so writing a file is often the whole
integration.

| Path | Contents |
| --- | --- |
| `/atak/imagery` | MOBAC `customMapSource` / WMS XML, MOBAC SQLite tilesets, `.gpkg`, legacy `.zip` caches; also FalconView/JMPS exports (GeoTIFF, NITF, CADRG, MrSID, KMZ…) |
| `/atak/imagery/mobile/mapsources` | **Streaming tile descriptors (`.json`)** — raster, vector or terrain. A MOBAC XML works from either this or `imagery/`; a streaming JSON only works here, and lands silently registered-but-invisible if put one level up |
| `/atak/grg` | GRG overlays **only** — GeoTIFF or KMZ |
| `/atak/overlays` | Vector overlays: DRW, GPX, KML, KMZ, LPT, Shapefile, and 3D models |
| `/atak/DTED/w117/n34.dt2` | DTED levels 0–3; the westing/northing naming is mandatory |
| `/atak/tools/missionpackage` | Data Packages |
| `/atak/export` | Where ATAK writes exported shapes |
| `/atak/prefs` | A file named `defaults` is read once at startup and then deleted — the fleet-provisioning hook |
| `/atak/support/apks/sideloaded` | Sideloaded plugin APKs |

Use `FileSystemUtils.getItem("imagery")` rather than hardcoding a root.

Datasets under `/atak/imagery` with more than 5000 chips are **not** rescanned
at startup; force one through the Import Manager.

## Writing files ATAK will read

- Write to a temporary sibling and rename over the target. ATAK may scan the
  directory at any moment, and a rename means it never sees a partial document.
  Rename-over also makes a repeat install idempotent instead of accumulating.
- Derive filenames with a **whitelist**, not an escape, if any part comes from
  data you did not author. Traversal out of the imagery directory is otherwise
  one `../` away.
- Preserve source documents **verbatim** rather than re-serialising from a
  parsed model, unless the model captures every field. A `customWmsMapSource`
  needs `layers`, `styles` and `additionalparameters`; rebuilding from a partial
  model drops them and yields a source that fetches nothing.

## MOBAC map source XML

Three root elements. Required fields, per ATAK's `MobacMapSourceFactory`:

| Root | Required |
| --- | --- |
| `customMapSource` | `name`, `url`, `maxZoom` |
| `customWmsMapSource` | `name`, `url`, `layers`, `maxZoom`, `tileType` |
| `customMultiLayerMapSource` | `name`, `layers` (nesting whole sources) |

Optional: `minZoom` (defaults 0), `tileType`, `tileUpdate`, `serverParts`,
`invertYCoordinate`, `backgroundColor`, `coordinatesystem`/`coordinateSystem`,
`ignoreErrors`; WMS adds `styles`, `version`,
`aditionalparameters`/`additionalparameters` (both spellings), `north`/`south`/
`east`/`west`.

URL templates use `{$x}`, `{$y}`, `{$z}`. Query separators are XML-escaped as
`&amp;` in the file and must reach ATAK unescaped.

When parsing a multi-layer source, look up **direct children only** — a
document-wide search for `<name>` finds a nested layer's name instead of the
composite's.

## Projections

ATAK displays in EPSG:4326 (flat) and EPSG:4978 ECEF (globe), both WGS84, and
uses Proj.4 to normalise or reproject data on ingest.

## Streaming tile sources (the `.json` descriptors)

Parsed by `com.atakmap.map.formats.cdn.StreamingTiles`. Keys: `schema`,
`name`/`title`, `url` (with `{$z}/{$x}/{$y}`), `attribution`, `eula`,
`downloadable`, `overlay`, `srs` (defaults `EPSG:3857`), `bounds`
(`minX`/`minY`/`maxX`/`maxY`), `invertYAxis`, `additionalParameters`,
`serverParts`, `refreshInterval`, `authorization`, `origin`, `tileMatrix` or
`numLevels`, `isQuadtree`, `content` (defaults `imagery`), `mimeType`,
`metadata`.

`content: "vector"` with `mimeType: "application/vnd.mapbox-vector-tile"`
works, and `metadata.styleSchema: "omt"` selects the OpenMapTiles style.

Three things that are not obvious:

- **The file must be in `imagery/mobile/mapsources/`.** Anywhere else it
  registers and is never selectable.
- **There is no other install route.** The Import Manager does not list
  `.json` (the extension is not registered with `ImportFilesTask`), and ATAK's
  own `linked-user-resources/mapsources` is internal — a file copied in there
  gets a `DELETE_DATA` and is removed. Register through
  `LayersMapComponent.getLayersDatabase().add(File)`.
- **`bounds` is ignored.** Coverage comes back as the global Web Mercator
  extent regardless of what you declare.

Verifying one worked: a streaming source registers as provider `tak-cdn`,
type `tiles`, `isRemote() == true`. A MOBAC XML registers as `mobac`/`mobac`.
Asserting the provider is the difference between "ATAK stored the file" and
"ATAK built a tile source from it" — `contains(file)` is true either way, and
so is the appearance of a new dataset name.

**Region download works for streaming vector**, not just raster: select the
source, then **Select Area** (Rectangle / Free Form / Lasso / Map Select) and
**Download**. `Download` stays disabled until an area exists. The cache also
grows as the user pans, so tiles persist offline without a bulk transfer.
