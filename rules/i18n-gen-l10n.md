# Localization: Flutter gen-l10n (ARB), English + German always

**Rule:** Localize with Flutter's built-in
[`gen-l10n`](https://docs.flutter.dev/ui/accessibility-and-internationalization/internationalization)
using ARB files. Never hard-code user-facing strings. **English and German are
mandatory** in every app; ES / FR / IT / PT are optional (add them freely — it's
free work for the AI, no downside).

## Why

- Hard-coded strings mean a rewrite to add a language later. ARB from the start
  costs nothing and makes new languages a copy-translate of one file.
- `gen-l10n` is built into Flutter (no third-party package), type-safe (missing key
  = compile error), and supports plurals/placeholders.

## Fix

```yaml
# pubspec.yaml
flutter:
  generate: true
```

```yaml
# l10n.yaml (project root)
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
```

```json
// lib/l10n/app_en.arb
{ "greeting": "Hello {name}", "@greeting": { "placeholders": { "name": {} } } }
```

```json
// lib/l10n/app_de.arb
{ "greeting": "Hallo {name}" }
```

```dart
Text(AppLocalizations.of(context)!.greeting(user.name));
```

Wire `AppLocalizations.delegate` + `supportedLocales` into `MaterialApp`.

## Notes / gotchas

- **Mandatory:** `app_en.arb`, `app_de.arb`. Optional: `app_es`, `app_fr`,
  `app_it`, `app_pt` — add them unless there's a reason not to.
- `app_en.arb` is the **template** — every other locale must have the same keys.
  A missing key falls back to English.
- Keep keys descriptive (`inbox_empty_title`, not `text1`).
- Regenerated automatically on build when `generate: true`.
