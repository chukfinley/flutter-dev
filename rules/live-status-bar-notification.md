# Live status-bar chip (Android 16 "Live Update") — native Kotlin

**Rule:** The nice **live notification in the top-left of the status bar** — a
self-updating countdown / progress chip, like a timer ticking down or a "now
playing" pill — is an **Android 16 promoted ongoing notification ("Live Update")**.
Build it **natively in Kotlin**, not with a Flutter notification package. When the
user says *"the notification top-left"* / *"Live Update"* / *"status-bar chip"*,
**this** is what's meant.

Reference implementation that works:
`uhr_app/android/app/src/main/kotlin/.../clock/ClockEngine.kt` (`showOngoing`).

## Why native, not a Flutter package

Flutter notification plugins (`flutter_local_notifications`) can post an ongoing
notification, but they do **not** expose:

- `setRequestPromotedOngoing(true)` — the Android 16 status-bar **chip** promotion.
- `setUsesChronometer` / `setChronometerCountDown` — an **OS-driven** ticking
  countdown (no foreground service, no per-second update loop from your code).
- `NotificationCompat.ProgressStyle()` — the new rich progress bar.
- `setShortCriticalText` — the tiny text shown in the collapsed chip.

So you drop to a native Kotlin `NotificationCompat.Builder` and drive it from Dart
over a `MethodChannel` (or a broadcast, as uhr_app does).

## The key trick: let the OS tick, don't loop

A countdown chip does **not** need a foreground service or a timer that re-posts
the notification every second. You post it **once** and tell Android the end time;
the OS renders the live countdown itself:

```kotlin
val builder = NotificationCompat.Builder(ctx, CHANNEL_RUNNING)
    .setSmallIcon(R.drawable.ic_stat_clock)
    .setContentTitle(title)
    .setOngoing(true)
    .setOnlyAlertOnce(true)                          // don't re-alert on updates
    .setContentIntent(openAppIntent())
    .setCategory(NotificationCompat.CATEGORY_PROGRESS)

// Android 16 status-bar chip ("Live Update"). No-op below API 36.
builder.setRequestPromotedOngoing(true)

if (running && Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    builder.setUsesChronometer(true)                 // OS renders the ticking time
    builder.setChronometerCountDown(true)            // count DOWN, not up
    builder.setWhen(endAtEpochMs)                    // when it hits zero
    // Android 16 (BAKLAVA): rich progress bar
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.BAKLAVA) {
        val pct = ((done * 100) / total).toInt()
        builder.setStyle(NotificationCompat.ProgressStyle().setProgress(pct))
    }
    builder.addAction(0, "+1 Min", adjustIntent(...))
    builder.addAction(0, "Pause",  pauseIntent(...))
} else {
    // paused / old Android: feed the remaining time directly
    builder.setContentTitle(formatMs(remaining))
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.BAKLAVA)
        builder.setShortCriticalText(formatMs(remaining))
}
notifier.notify(NOTIF_ID_BASE + id, builder.build())
```

Manifest requirement for the promoted chip:

```xml
<uses-permission android:name="android.permission.POST_PROMOTED_NOTIFICATIONS" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" /> <!-- API 33+ runtime -->
```

## Notes / gotchas

- **API gating everywhere:** `setRequestPromotedOngoing` needs API 36 (Android 16),
  `ProgressStyle` / `setShortCriticalText` need `BAKLAVA`, chronometer needs API 24
  (`N`). Always guard with `Build.VERSION.SDK_INT` and provide a plain-text fallback
  for the paused / older-OS case (post the formatted remaining time as the title).
- **Chronometer beats a loop:** with `setUsesChronometer` + `setWhen`, the countdown
  is smooth and costs zero battery — you never wake up to re-render. Only re-post on
  a real *state* change (pause, +1 min, stop).
- `setOnlyAlertOnce(true)` so updates don't buzz/sound every time.
- Actions (Pause / +1 min / Stop) route back via `PendingIntent` → a
  `BroadcastReceiver` that talks to your engine; keep the command contract in one
  place (uhr_app's `ClockContract`).
- Request the `POST_NOTIFICATIONS` runtime permission (API 33+) before posting, or
  the notify silently no-ops.
