# Secrets & config: `.env`, never hardcoded

**Rule:** Never hardcode API keys, base URLs, or config in Dart source. Put them in
a **`.env`** file that loads automatically, so a plain **`flutter run -d <platform>`
just works** — no extra flags, no manual wiring. Use
[`flutter_dotenv`](https://pub.dev/packages/flutter_dotenv).

## Fix

```yaml
# pubspec.yaml
dependencies:
  flutter_dotenv: ^5.1.0
flutter:
  assets:
    - .env            # bundled + auto-loaded; `flutter run` needs no extra flags
```

```dart
// main.dart
Future<void> main() async {
  await dotenv.load(fileName: '.env');
  runApp(const MyApp());
}

// anywhere
final apiBase = dotenv.env['API_BASE_URL']!;
final rcKey   = dotenv.env['REVENUECAT_KEY']!;
```

```bash
# .env  (gitignored)
API_BASE_URL=https://api.example.com
REVENUECAT_KEY=appl_xxx
```

Commit a **`.env.example`** with the keys but no values, so a fresh checkout knows
what to fill in. `.env` goes in `.gitignore`.

## Critical caveat: bundled ≠ secret

Anything shipped in the app — `.env` asset, `--dart-define`, hardcoded — is
**extractable from the APK**. So `.env` is for **client config and public keys**
(API base URL, RevenueCat *public* SDK key, a Sentry DSN). It is **not** a hiding
place for real secrets.

**True secrets (DB passwords, private API keys, signing tokens) never ship in a
client app at all** — they live on the backend, and the app talks to the backend.
If a key would let an attacker cost you money when extracted, it does not belong in
the client. (See `no-client-telemetry.md` — the same "keep it server-side" logic.)

## Notes / gotchas

- `.env` and `.env.*` (except `.env.example`) → `.gitignore`. Leaking a real key in
  git history can cost money; rotate immediately if it happens.
- Per-environment: `.env.dev` / `.env.prod`, load the right one (pairs with build
  flavors if you use them).
- Alternative for compile-time constants: `--dart-define-from-file=.env` — but that
  needs the flag on every run/build, so it breaks the "just `flutter run`" goal.
  `flutter_dotenv` (asset + runtime load) is the default for that reason.
