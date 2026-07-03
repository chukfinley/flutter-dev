# Flutter Dev Playbook

Personal + AI-readable memory bank for building & shipping Flutter apps.

Every time a "do it *this* way / use *this* thing, or you'll waste hours" lesson
comes up, it gets one file in [`rules/`](rules/) so it's never forgotten again.
Covers coding decisions, package choices, publishing, tooling — whatever bites.

**Two app types — you'll be told which up front:**

- **FOSS app** → maximize free & open-source: same repo, and we push every
  FOSS-specific rule as hard as possible (no GMS, `libre_location`, `flutter_map`,
  no client telemetry, GMS-free notifications).
- **Monetization app** → money is the priority, but **we still default to FOSS /
  open-source components** wherever they don't hurt the money goal. We only reach for
  a proprietary dependency (FCM, Google Maps, Crashlytics, Play Billing) when it
  genuinely converts better or is required — not by default.

Either way, the **quality/architecture** rules (state, structure, models,
networking, testing, signing, monetization) apply to **every** app. The
**FOSS-specific** rules are mandatory for FOSS apps and the *preferred default* for
money apps — deviate only for a real monetization reason.

**Universal by default.** Whatever can be built cross-platform (one codebase for
Android/iOS/desktop/web, platform-agnostic packages, a shared widget) → build it
universal. Drop to platform-specific native code only when there's no cross-platform
way. Prefer packages that support all target platforms.

## How to use

**Human:** browse [`rules/`](rules/). Each file = one decision or resource, with the *why*.

**AI (Claude Code / Codex / etc.):** drop [`SKILL.md`](SKILL.md) into your skills dir,
or point the agent at this repo. Rules are written to be loaded as context.

## Rules

| Area | Rule | Why |
|------|------|-----|
| State | [Always use Riverpod](rules/state-riverpod.md) | One consistent, compile-safe, async-friendly state solution across every app |
| Models | [freezed + json_serializable](rules/models-freezed.md) | Immutable models, copyWith, JSON — no hand-written boilerplate/bugs |
| Theming | [Material 3 Expressive](rules/theming-material3-expressive.md) | M3 on, seed-driven light/dark, no hard-coded colors (detailed skill WIP) |
| Structure | [Feature-first folders](rules/project-structure.md) | Everything for a feature in one folder → no scattered or duplicated files |
| Routing | [Use `go_router`](rules/routing-go-router.md) | One declared route table; official package; handles deep links / back button |
| Lint | [Strict analysis from day one](rules/lint-strict.md) | Catches runtime-class bugs at edit time; keeps every app's style identical |
| Networking | [`dio` + certificate pinning (+ test)](rules/http-dio-cert-pinning.md) | Interceptors/retry in one place; pinning blocks MITM; a test proves the pin works |
| Storage | [secure_storage for secrets, SQL for data](rules/storage-secure-vs-sql.md) | Bulk data in secure_storage froze the app on desktop; SQL for anything that grows |
| Maps | [`flutter_map` + OpenStreetMap](rules/maps-flutter-map.md) | No GMS, no Mapbox account/billing; free OSM tiles |
| Location | [Use `libre_location`, not `geolocator`](rules/location-libre-location.md) | `geolocator` pulls Google Play Services → IzzyOnDroid flags app as non-free; no background permission |
| Notifications | [Tiered push: Signal-way websocket → UnifiedPush → FCM](rules/notifications-push.md) | Baseline = own websocket + `remoteMessaging` FGS (no distributor, works everywhere); push is a wake-up poke; E2EE stays on your relay |
| Live chip | [Android 16 status-bar Live Update (native Kotlin)](rules/live-status-bar-notification.md) | The "notification top-left"; OS-ticked countdown chip, no Flutter package can do it |
| Telemetry | [No crash/analytics SDK in the app](rules/no-client-telemetry.md) | Firebase = GMS + tracking flag; log server-side (PostHog) where you control it |
| Secrets | [`.env`, never hardcoded](rules/secrets-dotenv.md) | `flutter run` just works via flutter_dotenv; bundled ≠ secret, real keys stay backend-side |
| Monetization | [RevenueCat, server-truth](rules/monetization-revenuecat.md) | One SDK for all stores; `app_user_id == backend id`; entitlement checked server-side |
| Icons | [Launcher icons + splash](rules/app-icons-splash.md) | One source image → all platforms via flutter_launcher_icons + native_splash |
| Testing | [Unit / widget / integration, in CI](rules/testing-strategy.md) | Always test security/money logic + every fixed bug; red test blocks merge |
| CI | [GitHub Actions builds release APK](rules/ci-github-actions.md) | Analyze+test on PR, build+release on tag; keystore from secrets |
| Flutter version | [Pin with fvm, keep updated](rules/flutter-version-fvm.md) | Reproducible builds; moving pin — bump often, sync with CI |
| Build | [Release APK, per-app keystore, back it up](rules/build-release-signing.md) | Never debug builds; lost keystore = can't update app ever; APK vs AAB vs split-per-abi |
| i18n | [gen-l10n (ARB), EN + DE always](rules/i18n-gen-l10n.md) | No hard-coded strings; new languages become copy-translate of one file |
| Publishing | [Get Play Store testers via TestersCommunity](rules/play-store-testers.md) | Google now requires 12 testers × 14 days before you can publish |

_Add a row above whenever you add a file in `rules/`._

## Format

One rule = one file. Keep it simple:

```
# <Short title>

**Rule:** <the one-line standard / tip>

**Problem:** <what the obvious/default choice breaks or costs you>

**Fix:** <concrete package, link, or steps>

**Notes / gotchas:** <edge cases>
```

License: [The Unlicense](LICENSE) — public domain, copy freely.
