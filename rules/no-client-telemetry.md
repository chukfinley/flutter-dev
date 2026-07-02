# No telemetry/crash SDK in the app — log server-side instead

**Rule:** Do **not** put Firebase Crashlytics, Firebase Analytics, or any tracking
SDK in the Flutter app. Keep the client clean. Do your logging/analytics in the
**backend, which you control** — currently **PostHog** (self-host / your project).

## Why

- Firebase/Crashlytics/Analytics pull **Google Play Services** → IzzyOnDroid
  `NonFreeDep` + `Tracking` anti-features. Kills the FOSS listing.
- A client-side tracking SDK is a privacy liability and, for E2EE-style apps,
  contradicts the whole promise.
- You mostly don't want client telemetry anyway. When you *do* need insight, the
  backend already sees the meaningful events — and there you can log as much as you
  want because it's your infrastructure.

## Fix

- **Client:** ship nothing. No crash SDK, no analytics SDK. At most, local logging
  for debugging (not shipped/uploaded).
- **Backend:** log + analyze with **PostHog** (current choice). It's under your
  control, so log freely there. Sentry (self-hosted) is a fine alternative for
  error tracking if a project needs stack traces.

## Notes / gotchas

- Default is **log nothing** on the client; add backend events only when a specific
  question needs answering.
- If you ever need crash reports from the app itself for a non-FOSS build, prefer a
  self-hosted Sentry over Crashlytics — still avoids GMS.
- This keeps every app's client side identical and store-safe by default.
