# Notifications & push

**Rule:** Separate two different things:

1. **Local notifications** (the app itself shows a notification) →
   [`flutter_local_notifications`](https://pub.dev/packages/flutter_local_notifications).
   No GMS, no server. Use for reminders, timers, or "app is running in the
   background" foreground-service notifications.
2. **Push** (a server wakes a *closed* app) → for FOSS apps use **UnifiedPush**.
   Self-host the push server (**ntfy**). Ship **one** build that carries an
   **embedded FCM distributor** so the same UnifiedPush code path rides FCM for
   Play-Store / Google-phone users. **iOS is separate — APNs, always.**

## The key insight (why "ntfy is just text" doesn't matter)

Push is **not** "deliver my message". Push is **"wake the app up"**.

For an E2EE messenger (Signal-style), you never send the real message through the
push service. The flow is:

1. Your relay/backend has a new ciphertext for a user.
2. Backend sends a **tiny, opaque wake-up ping** to the push service
   (empty / encrypted body — no plaintext, no sender).
3. The push service delivers that ping to the phone, which **wakes your app**.
4. Your app then **pulls the actual encrypted message from your own relay** over
   its normal channel and decrypts it **locally**.
5. Your app shows a **local notification** with the decrypted text.

So the push service (ntfy, FCM, whatever) only ever sees a meaningless "poke". It
never sees message content. "ntfy only carries simple text" is fine — you're
sending it a poke, not the message. Content stays E2EE on infrastructure you own.
This is exactly how the messenager app works (WebSocket relay + wake-up push).

**Bonus:** since UnifiedPush went WebPush-based (see below), the poke payload is
itself RFC-8291 encrypted end-to-end, so even the "a message exists" body is
opaque to the push server.

## Why you can't just use your own WebSocket

You already have a relay/WebSocket. Why involve a push service at all? Because
**Android kills background connections**, and the OS keeps tightening the screws:

- **Foreground service** — keep the socket alive with a permanent "App is running"
  notification. Works (Signal, SimpleX, Briar do this), but drains battery, and on
  **Android 15+** the `dataSync` / `mediaProcessing` FGS types are **capped at 6h
  per 24h** (`onTimeout` fires and kills it). Play Store also scrutinizes permanent
  FGS. Increasingly fragile as a *default*.
- **WorkManager polling** — cheap but not instant (min ~15 min, not guaranteed);
  Android 16 tightens job quotas further. This is a "slow mode" floor, not push.
- **UnifiedPush / FCM** — the phone keeps **one** shared, battery-efficient
  connection (via the distributor) that can wake **any** app. One socket for the
  whole phone instead of one per app. This is the right default.

## What UnifiedPush actually is (and the 2025 WebPush change)

Four roles:

- **Push server** — holds the persistent internet connection (self-hostable: ntfy,
  NextPush, Sunup). *You host this one.*
- **Distributor** — an app on the phone that keeps ONE socket to the push server and
  fans out to all UnifiedPush apps (the ntfy app, or an **embedded** one you bundle).
- **Connector** — the library inside your app; registers with the distributor and
  gets back an **endpoint URL**.
- **App server** — your backend; `POST`s to the endpoint URL to wake the app.

**Important 2025 change — UnifiedPush is now WebPush.** As of the Nov 2025 spec,
server→server delivery *is* Web Push (RFC 8030), payloads **must** be encrypted
(RFC 8291), and authorization uses **VAPID** (RFC 8292). Practical consequence:
**build your backend as a standard WebPush/VAPID sender**, not a bespoke
"UnifiedPush" or "ntfy" client. One compliant WebPush server then hits UnifiedPush
endpoints, browser WebPush, and (via the embedded distributor) FCM. Don't model UP
as its own weird protocol anymore.

## How other FOSS apps actually do it (reality check)

| App | Mechanism | GMS-free? |
|-----|-----------|-----------|
| Signal (upstream) | **FCM only**; websocket+FGS fallback, no UnifiedPush | partial |
| **Molly** (Signal fork) + **Mollysocket** | UnifiedPush via a self-hosted bridge that holds Signal's socket server-side, pokes the app | yes |
| Element / **FluffyChat** (Flutter) | UnifiedPush via Matrix **Sygnal** gateway | yes |
| SimpleX | persistent background socket on Android; **content-free APNs** on iOS | yes (Android) |
| Briar | **no push at all** — Tor + foreground service; no iOS app | yes |
| Session | own push server; FCM "fast mode" or polling "slow mode" | yes (slow) |
| Tusky / Fedilab (Mastodon) | **WebPush (VAPID)** over UnifiedPush | yes |

Takeaway: **UnifiedPush + self-hosted push server is the mainstream FOSS answer**;
FCM is only the Google-phone fast path; iOS is always APNs.

## The distributor problem, and the one-build answer

On a de-Googled phone the user must have *some* distributor installed (e.g. the ntfy
app). You don't want two separate push code paths, though. The modern pattern:

- **One app**, using the UnifiedPush connector API.
- Bundle an **embedded FCM distributor** (`embedded_fcm_distributor`) so on phones
  *with* Play Services, UnifiedPush transparently rides FCM — no separate FCM logic.
- On GMS-free phones, the user's real UnifiedPush distributor (ntfy) handles it.
- Gate the FCM/GMS bits behind a **build flavor** so the **F-Droid flavor ships
  zero Google code** (see `build-release-signing.md` for flavors).

## Security: endpoint secrecy + self-hosted ntfy auth

The endpoint URL is a **capability** — a long random `up*` topic
(`https://ntfy.example.com/up<40-random-chars>`). Whoever has it can *send* to it.
So:

- **Treat the endpoint as a secret.** Don't log or leak it. Rotate (re-register) to
  invalidate a leaked one.
- **If an endpoint leaks, the damage is bounded:** an attacker can spam wake-ups
  (battery / minor DoS) and learn *that* a message exists (timing metadata). They
  **cannot read content** — the WebPush payload is RFC-8291 encrypted end-to-end and,
  in a good design, carries no plaintext anyway. No content compromise, no spoofed
  message (your relay authenticates the real fetch).
- **Self-host ntfy and lock it down.** The UnifiedPush model needs the app server to
  have *write* to the `up*` topics, so the correct hardening is: deny everything by
  default, then **allow (anonymous or token) write only to the `up*` prefix**, deny
  read to everyone else:

```yaml
# ntfy server.yml (Docker, one container)
auth-file: /var/lib/ntfy/auth.db
auth-default-access: "deny-all"          # nothing allowed unless granted
# then, via `ntfy access`:
#   allow write to topic prefix  up*      (your backend / senders)
#   deny  read  to everyone else
```

If *you* control the sender, require a scoped **publish token** for writes instead
of anonymous — tighter, at the cost of not accepting generic third-party senders
(fine for your own messenger).

## What you host + implement

**You host:** one **ntfy server** (Docker + your domain + the auth above). No
per-user infrastructure. Your existing relay gains one job: after storing a
ciphertext, send a WebPush poke to the user's endpoint.

**App side (Flutter):** use [`unifiedpush`](https://pub.dev/packages/unifiedpush)
— **v6.2.0**, verified publisher, actively maintained, **Android + Linux only (no
iOS)**. FluffyChat is the reference consumer.

```yaml
dependencies:
  unifiedpush: ^6.2.0
```

```dart
// 1. register once; the distributor hands back an endpoint URL
await UnifiedPush.register(instance: 'default');

// 2. receive the endpoint, send it to YOUR backend to store per-user
onNewEndpoint(PushEndpoint endpoint, String instance) {
  api.savePushEndpoint(endpoint.url); // backend WebPushes here later
}

// 3. a poke arrives -> app is woken -> fetch + decrypt from your relay
onMessage(PushMessage message, String instance) async {
  final ciphertext = await relay.fetchPending(); // your own channel
  final text = crypto.decrypt(ciphertext);       // local
  localNotifications.show(text);                  // flutter_local_notifications
}
```

**Backend side:** a standard **WebPush (VAPID)** send to the stored endpoint URL,
empty/encrypted body → phone wakes.

**iOS:** the `unifiedpush` package does **not** cover iOS. Implement an **APNs**
path via a (self-hostable) notification relay that holds the APNs token and sends
**content-free** pushes (the SimpleX model). Keep the wake-then-fetch design so only
the transport differs per platform.

Production checklist: (1) auth-locked ntfy container, (2) WebPush/VAPID sender in
your backend, (3) store each user's endpoint, (4) content-free poke → app fetch +
decrypt + local notification, (5) embedded FCM distributor behind a flavor for the
Play-Store build, (6) separate APNs path for iOS.

## Non-messenger apps

If the app only needs to keep a connection alive for its own reason (live socket,
ongoing sync) and not deliver server push to a *closed* app, you don't need
UnifiedPush — a **foreground service + `flutter_local_notifications`** ("App is
active") is enough (mind the Android 15 6h FGS cap for `dataSync`).

## Notes / gotchas

- Push payload = **content-free poke only**. Never put message plaintext (or
  anything privacy-sensitive) in the body, even with FCM.
- **FCM penalizes silent pushes:** high-priority FCM *data* messages that don't
  produce a user-facing notification get **deprioritized after ~7 days** of that
  behavior. So always surface a notification when you wake. (ntfy/UnifiedPush has no
  such penalty.) FCM without GMS needs microG.
- **iOS: APNs is unavoidable.** No UnifiedPush on iOS. Every FOSS messenger that has
  iOS push (SimpleX, Session, Element) routes through APNs with content-free pushes.
- Keep the F-Droid flavor **free of all Google code** — FCM/embedded-FCM lives only
  in the Play flavor.
