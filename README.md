# Flutter Dev Playbook

Personal + AI-readable memory bank for building & shipping Flutter apps.

Every time a "do it *this* way / use *this* thing, or you'll waste hours" lesson
comes up, it gets one file in [`rules/`](rules/) so it's never forgotten again.
Covers coding decisions, package choices, publishing, tooling — whatever bites.

## How to use

**Human:** browse [`rules/`](rules/). Each file = one decision or resource, with the *why*.

**AI (Claude Code / Codex / etc.):** drop [`SKILL.md`](SKILL.md) into your skills dir,
or point the agent at this repo. Rules are written to be loaded as context.

## Rules

| Area | Rule | Why |
|------|------|-----|
| State | [Always use Riverpod](rules/state-riverpod.md) | One consistent, compile-safe, async-friendly state solution across every app |
| Models | [freezed + json_serializable](rules/models-freezed.md) | Immutable models, copyWith, JSON — no hand-written boilerplate/bugs |
| Structure | [Feature-first folders](rules/project-structure.md) | Everything for a feature in one folder → no scattered or duplicated files |
| Routing | [Use `go_router`](rules/routing-go-router.md) | One declared route table; official package; handles deep links / back button |
| Lint | [Strict analysis from day one](rules/lint-strict.md) | Catches runtime-class bugs at edit time; keeps every app's style identical |
| Networking | [`dio` + certificate pinning (+ test)](rules/http-dio-cert-pinning.md) | Interceptors/retry in one place; pinning blocks MITM; a test proves the pin works |
| Storage | [secure_storage for secrets, SQL for data](rules/storage-secure-vs-sql.md) | Bulk data in secure_storage froze the app on desktop; SQL for anything that grows |
| Maps | [`flutter_map` + OpenStreetMap](rules/maps-flutter-map.md) | No GMS, no Mapbox account/billing; free OSM tiles |
| Location | [Use `libre_location`, not `geolocator`](rules/location-libre-location.md) | `geolocator` pulls Google Play Services → IzzyOnDroid flags app as non-free; no background permission |
| Notifications | [Local vs push; UnifiedPush wake-then-fetch](rules/notifications-push.md) | Push is just a wake-up poke; E2EE content stays on your relay; self-host ntfy / NextPush |
| Telemetry | [No crash/analytics SDK in the app](rules/no-client-telemetry.md) | Firebase = GMS + tracking flag; log server-side (PostHog) where you control it |
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
