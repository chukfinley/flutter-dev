---
name: flutter-dev-playbook
description: >
  Hard-won standards and go-to resources for building and shipping Flutter apps —
  package choices, FOSS/F-Droid compliance, Play Store publishing, tooling. Use
  when writing, reviewing, or releasing a Flutter app and you want the "do it this
  way, not the obvious way" answer. Triggers: "Flutter", "F-Droid", "IzzyOnDroid",
  "Play Store publish", "closed testing", "geolocator", "FOSS app", "no GMS".
---

# Flutter Dev Playbook

Memory bank of Flutter decisions that are easy to get wrong. Before choosing a
package or a release step, check the matching file in `rules/`.

## Rules index

- **State management** → `rules/state-riverpod.md`
  Always Riverpod. `setState` only for purely local throwaway widget state.
- **Data models** → `rules/models-freezed.md`
  freezed + json_serializable for immutable models / copyWith / JSON.
- **Project structure** → `rules/project-structure.md`
  Feature-first folders: `features/<name>/{data,domain,presentation}`, plus `core/` + `shared/`.
- **Routing** → `rules/routing-go-router.md`
  Use `go_router`; one route table; set `debugLogDiagnostics: false` to quiet logs.
- **Lint** → `rules/lint-strict.md`
  Strict `analysis_options.yaml` (`very_good_analysis`); wire `dart analyze` into CI.
- **Networking** → `rules/http-dio-cert-pinning.md`
  `dio` client; SPKI certificate pinning on backends you control, with a test that proves it rejects a bad cert.
- **Local storage** → `rules/storage-secure-vs-sql.md`
  `flutter_secure_storage` for small secrets only; SQL (`sqflite`/`drift`) for anything that grows (chats, AI history). Bulk in secure_storage froze the app on desktop.
- **Maps** → `rules/maps-flutter-map.md`
  `flutter_map` + OpenStreetMap tiles; not google_maps_flutter (GMS) or Mapbox.
- **Location / GPS** → `rules/location-libre-location.md`
  Use `libre_location`, never `geolocator` (Google Play Services → non-free flag). No background-location permission unless truly needed.
- **Notifications & push** → `rules/notifications-push.md`
  Local = flutter_local_notifications. Push is tiered: BASELINE = "the Signal way" — app-owned websocket in a `remoteMessaging` foreground service (Android 14+, exempt from Android 15's 6h cap, no distributor needed, works everywhere; auto-fallback from FCM). Battery-saver = UnifiedPush (now WebPush/VAPID) with self-hosted ntfy; content-free poke → app fetches E2EE from your relay; build backend as a WebPush sender. Google phones = FCM first via embedded distributor behind a flavor; F-Droid flavor stays Google-free (baseline websocket ships in both). iOS = separate APNs path (`unifiedpush` pkg is Android+Linux only). Endpoint URL = secret capability; leak = DoS/metadata only, never content. Always surface a notification (FCM deprioritizes silent data msgs after ~7d).
- **Live status-bar chip** → `rules/live-status-bar-notification.md`
  The "notification top-left" = Android 16 promoted-ongoing Live Update. Native Kotlin (setRequestPromotedOngoing + POST_PROMOTED_NOTIFICATIONS), OS-ticked chronometer countdown, no foreground service/loop. Reference: uhr_app ClockEngine.kt.
- **Client telemetry** → `rules/no-client-telemetry.md`
  No Firebase/Crashlytics/analytics SDK in the app. Log server-side (PostHog).
- **Build & signing** → `rules/build-release-signing.md`
  Always `--release`, never debug. Per-app keystore, generated + backed up on app creation, gitignored. Universal APK for F-Droid/direct, AAB for Play Store, split-per-abi for smaller self-distributed APKs.
- **Localization** → `rules/i18n-gen-l10n.md`
  gen-l10n ARB; EN + DE mandatory; ES/FR/IT/PT optional. No hard-coded strings.
- **Play Store publishing** → `rules/play-store-testers.md`
  Get the required 12 closed testers via testerscommunity.com/submit-app.

_(More rules get added as files under `rules/`. Always check there first.)_

## General principles

- **FOSS builds:** before adding a dependency, ask *does it link Google Play
  Services or another proprietary blob?* If yes, find the LocationManager-level /
  platform-native / self-hosted alternative. A working app flagged `NonFreeDep`
  on IzzyOnDroid is a distribution failure, not a success.
- **Publishing:** solve the closed-testing / tester requirements early — they gate
  the whole release, not the code.
