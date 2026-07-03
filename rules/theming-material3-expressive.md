# Theming: Material 3 Expressive

**Rule:** Every app uses **Material 3** (`useMaterial3: true`, now the default) and
aims for the new **Material 3 Expressive** styling. Build the color scheme from a
seed, support light **and** dark, and don't scatter hard-coded colors/sizes.

> A dedicated **theming skill** is in the works and will carry the detailed
> Expressive guidance. Until it lands, this rule is the baseline: M3 on, Expressive
> as the target look.

## Fix

```dart
final seed = const Color(0xFF6750A4);

MaterialApp(
  theme: ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(seedColor: seed),
  ),
  darkTheme: ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(seedColor: seed, brightness: Brightness.dark),
  ),
  themeMode: ThemeMode.system, // respect the OS setting
);
```

- Pull colors from `Theme.of(context).colorScheme` (`primary`, `surface`,
  `surfaceContainer`, …), **not** literal hex values in widgets.
- Same for text: `Theme.of(context).textTheme`.
- Keep the theme in `core/theme/` (see `project-structure.md`).

## What "Expressive" adds

Material 3 Expressive (Google's 2025 update) leans into bolder shapes, larger/looser
type, more motion, and richer container roles. Prefer the newer expressive components
and motion where available rather than the flat default M3 look.

## Notes / gotchas

- **No hard-coded colors** in widgets — one seed drives light + dark. Hard-coded hex
  breaks dark mode and theming.
- Respect `ThemeMode.system` so the app follows the phone's light/dark setting.
- Detailed Expressive patterns will come from the theming skill; don't over-engineer
  a custom design system by hand in the meantime.
