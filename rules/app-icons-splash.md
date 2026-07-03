# App icons & splash screen

**Rule:** Generate launcher icons and the splash screen from a single source image
with [`flutter_launcher_icons`](https://pub.dev/packages/flutter_launcher_icons) and
[`flutter_native_splash`](https://pub.dev/packages/flutter_native_splash). Don't
hand-place icon files per platform.

## Why

Every platform (Android legacy + adaptive, iOS, web, desktop) wants icons in a dozen
sizes/formats, and a native splash needs platform-specific setup. Doing it by hand is
tedious and easy to get wrong (blurry icons, wrong adaptive background, white flash on
launch). These two packages generate everything from one config.

## Fix

```yaml
dev_dependencies:
  flutter_launcher_icons: ^0.14.0
  flutter_native_splash: ^2.4.0

flutter_launcher_icons:
  image_path: "assets/icon/icon.png"        # one square ~1024px source
  android: true
  ios: true
  adaptive_icon_background: "#0B0B0B"
  adaptive_icon_foreground: "assets/icon/foreground.png"

flutter_native_splash:
  color: "#0B0B0B"
  image: "assets/icon/splash.png"
  android_12:                                # Android 12+ uses its own splash API
    image: "assets/icon/splash.png"
    color: "#0B0B0B"
```

```bash
dart run flutter_launcher_icons
dart run flutter_native_splash:create
```

Re-run after changing the source image.

## Notes / gotchas

- **Adaptive icons (Android):** provide a transparent-background foreground with safe
  padding — Android masks it to circle/squircle/etc. A full-bleed image gets cropped.
- **Android 12+** ignores the old splash image and shows the app icon on a colored
  background via the system splash API — configure the `android_12:` block or it
  looks wrong on new phones.
- Commit the generated files (they're part of the native projects).
- Provide a dark-mode splash color if your app is dark by default.
