# Location: use `libre_location`, not `geolocator`

**Rule:** In FOSS Flutter apps that ship on F-Droid / IzzyOnDroid, get GPS with
[`libre_location`](https://pub.dev/packages/libre_location) instead of `geolocator`.

## Problem

`geolocator` (and `google_maps_flutter`, `location`, Firebase, etc.) depend on
**Google Play Services (GMS)** by default. Even when you disable the fused
location provider, the GMS dependency is still linked in. Result:

- IzzyOnDroid's scanner flags the app with the **`NonFreeDep` / `Tracking` anti-feature**.
- On the official F-Droid repo it may be rejected or listed with a warning.
- Users who explicitly want GMS-free builds (GrapheneOS, /e/OS, de-Googled phones)
  get a degraded or non-working app.

This is a *distribution* cost, not a code-quality one — the app "works" but gets
marked non-free, which kills trust in the FOSS store listings.

## Fix

`libre_location` uses only the **Android platform LocationManager** (raw GPS/network
provider), no Play Services, no proprietary deps.

```yaml
# pubspec.yaml
dependencies:
  libre_location: ^<latest>   # check pub.dev for current version
```

```dart
import 'package:libre_location/libre_location.dart';

final libreLocation = LibreLocation();

// one-shot
final pos = await libreLocation.getLocation();
print('${pos.latitude}, ${pos.longitude}');

// stream of updates
libreLocation.onLocationChanged.listen((pos) {
  // handle position
});
```

Permissions still required in `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

Handle the runtime permission request yourself (e.g. `permission_handler`, which is
GMS-free) before calling `getLocation()`.

## Do NOT request background location unless the app truly needs it

Only request **foreground** location (`ACCESS_FINE_LOCATION` /
`ACCESS_COARSE_LOCATION`). Do **not** add `ACCESS_BACKGROUND_LOCATION` unless the
app genuinely tracks location while closed (e.g. a run/route tracker).

Why it costs you:

- **Google Play:** background location triggers a manual review where you must
  submit a **video / written justification** explaining why the app needs it.
  Slow, easy to fail, and can hold up or reject the whole release.
- **F-Droid / IzzyOnDroid:** background location is treated as a
  privacy/tracking concern and gets flagged as an anti-feature.

So by default, keep it off:

```xml
<!-- present: foreground only -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- ABSENT on purpose — do NOT add unless the app tracks in the background:
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" /> -->
```

If a transitive dependency drags `ACCESS_BACKGROUND_LOCATION` in, strip it:

```xml
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION"
    tools:node="remove" />
```

## Notes / gotchas

- No **fused** provider → positions come straight from GPS/network. Slightly slower
  first fix and no Google-side sensor fusion, but fully free.
- No built-in geocoding (address ↔ coords). For that use a FOSS option like
  Nominatim over HTTP, not the `geocoding` package (also GMS-adjacent on some setups).
- Verify a release build with `apkanalyzer` / the IzzyOnDroid scanner before publishing:
  make sure no `com.google.android.gms` classes slipped in via another dep.
- If you must keep `geolocator` for iOS reasons, isolate it behind a platform
  interface and only wire `libre_location` on Android.
