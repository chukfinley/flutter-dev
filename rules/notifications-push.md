# Notifications & push

**Rule:** Separate two different things:

1. **Local notifications** (the app itself shows a notification) →
   [`flutter_local_notifications`](https://pub.dev/packages/flutter_local_notifications).
   No GMS, no server. Use for reminders, timers, or "app is running in the
   background" foreground-service notifications.
2. **Push** (a server wakes a *closed* app) → for FOSS apps use **UnifiedPush**
   with a distributor you control (self-hosted **ntfy** or **NextPush** on your
   Nextcloud). Ship an **FCM** path only as a fallback for the Play Store build.

## The key insight (why "ntfy is just text" doesn't matter)

Push is **not** "deliver my message". Push is **"wake the app up"**.

For an E2EE messenger (Signal-style), you never send the real message through the
push service. The flow is:

1. Your relay/backend has a new ciphertext for a user.
2. Backend sends a **tiny, opaque wake-up ping** to the push service
   (empty body, or a random/encrypted token — no plaintext).
3. The push service delivers that ping to the phone, which **wakes your app**.
4. Your app then **pulls the actual encrypted message from your own relay** over
   its normal channel and decrypts it **locally**.
5. Your app shows a **local notification** with the decrypted text.

So the push service (ntfy, FCM, whatever) only ever sees a meaningless "poke". It
never sees message content. "ntfy only carries simple text" is fine — you're
sending it a poke, not the message. Content stays E2EE on infrastructure you own.
This is exactly how the messenager app works (WebSocket relay + wake-up push).

## Why you can't just use your own WebSocket

You already have a relay/WebSocket. Why involve a push service at all? Because
**Android kills background connections.** When the app is closed/swiped away, your
socket dies. Options:

- **Foreground service** — keep the socket alive with a permanent "App is running"
  notification. Works, but drains battery and Play Store scrutinizes it.
- **UnifiedPush / FCM** — the phone keeps **one** shared, battery-efficient
  connection (via the distributor) that can wake **any** app. That's the whole
  point: one persistent socket for the whole phone instead of one per app.

## What UnifiedPush actually is

UnifiedPush is an open **wake-up protocol**, not a server product. Three parts:

- **Distributor app** on the phone — holds the one persistent connection. Examples:
  the ntfy app, NextPush, etc. The user installs one.
- **Your app** — registers with whatever distributor is present, gets back an
  **endpoint URL**.
- **Your backend** — HTTP `POST`s the wake-up to that endpoint URL.

Your app doesn't care *which* distributor the user chose — that's the "unified"
part. The distributor forwards the poke to your app and wakes it.

## Distributor options (you dislike ntfy.sh — fine)

`ntfy.sh` is just the **public** ntfy instance. You don't have to use it:

- **Self-hosted ntfy** (Docker, one container) — same software, *your* server, your
  domain. Solves the "I don't want ntfy.sh" problem while keeping ntfy the app.
- **NextPush** — a UnifiedPush provider that runs on **your Nextcloud** (you already
  self-host Nextcloud). The user installs the "NextPush" distributor app pointed at
  your Nextcloud; no separate push server to run.
- **Mollysocket** — the reference bridge for a Signal fork (Molly): turns Signal's
  push into UnifiedPush wake-ups. Good pattern to copy for a Signal-style app.
- **Matrix Sygnal / sunup** — other self-hostable push gateways if you outgrow ntfy.
- **Embedded FCM distributor** — UnifiedPush can fall back to FCM on Google phones
  where the user has no distributor installed.

Recommendation for a FOSS messenger: **self-hosted ntfy OR NextPush** for the
F-Droid build, **FCM data-messages as fallback** in the Play Store build (both
just send wake-up pokes; message fetch/decrypt stays identical).

## Non-messenger apps

If the app only needs to keep a connection alive for its own reason (live web
socket, ongoing sync) and not deliver server push to a closed app, you don't need
UnifiedPush — a **foreground service + `flutter_local_notifications`** ("App is
active") is enough.

## Notes / gotchas

- Push payload = **opaque poke only**. Never put message plaintext (or anything
  privacy-sensitive) in the push body, even with FCM.
- **iOS** has no UnifiedPush — you must use **APNs** there. Keep the wake-then-fetch
  design so only the transport differs per platform.
- FCM pulls **Google Play Services** → the FCM path makes that build non-FOSS. Keep
  it out of the F-Droid flavor; gate it behind a build flavor.
