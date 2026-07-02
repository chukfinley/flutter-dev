# Maps: flutter_map + OpenStreetMap, not Google Maps / Mapbox

**Rule:** Show maps with [`flutter_map`](https://pub.dev/packages/flutter_map)
using OpenStreetMap tiles. Do **not** use `google_maps_flutter` (GMS) or a paid
Mapbox SDK.

## Problem

- `google_maps_flutter` pulls **Google Play Services** → IzzyOnDroid `NonFreeDep`
  flag, plus a Google Maps API key + billing account.
- Mapbox needs an account, an access token, and has usage-based pricing — a
  running cost and a proprietary dependency in a FOSS app.

## Fix

`flutter_map` is a pure-Dart Leaflet-style widget. It renders whatever tiles you
point it at; use free OpenStreetMap raster tiles (or a self-hosted / vector
source). This is the stack already used in the Besser-Bahn app.

```yaml
# pubspec.yaml
dependencies:
  flutter_map: ^8.0.0
  latlong2: ^0.9.0                  # LatLng type
  flutter_map_tile_caching: ^10.0.0 # optional: offline / cached tiles
  vector_map_tiles: ^9.0.0          # optional: vector tiles instead of raster
```

```dart
FlutterMap(
  options: const MapOptions(
    initialCenter: LatLng(54.32, 10.13), // Kiel
    initialZoom: 13,
  ),
  children: [
    TileLayer(
      urlTemplate: 'https://tile.openstreetmap.org/{z}/{x}/{y}.png',
      userAgentPackageName: 'dev.chuk.yourapp', // required by OSM policy
    ),
    const RichAttributionWidget(
      attributions: [TextSourceAttribution('© OpenStreetMap')],
    ),
  ],
)
```

## Notes / gotchas

- **Attribution is mandatory** — always show "© OpenStreetMap". OSM's tile usage
  policy also requires a real `userAgentPackageName`; heavy apps should self-host
  tiles or use a tile provider rather than hammering `tile.openstreetmap.org`.
- The **default flutter_map logo** is a package asset that isn't always bundled →
  can throw at runtime; provide your own attribution widget instead of relying on it.
- For offline / cached maps use `flutter_map_tile_caching`; for crisp scalable
  tiles use `vector_map_tiles` (needs a vector tile / style source).
- Pair with `libre_location` for the "my location" dot — see
  `location-libre-location.md`.
