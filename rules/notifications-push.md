# Notifications & push

## TL;DR — what we do

Same pattern on every platform: **the push only says "wake up, pull your message."
It never carries the message content.** Because the poke is contentless, the push
transport is untrusted and irrelevant — it can never leak or read anything. The app
wakes, fetches the real (encrypted) message from *our* relay, decrypts locally, and
shows the notification.

- **Android → our own WebSocket service** (the "Signal way": a `remoteMessaging`
  foreground service holding our socket). No Google, no Apple, no distributor. Works
  on every Android phone. This is all we build by default.
- **iOS → Apple's official APNs** (only when we actually ship an iPhone app). Apple
  forbids background sockets, so APNs is the *only* way to wake a closed app. Our
  backend sends a contentless poke straight to Apple; the app then pulls + decrypts
  from our relay, exactly like Android.

Everything else below (UnifiedPush, FCM, WebPush, build flavors) is **optional
battery optimization** — automatic fallbacks, not a menu, not required.

---

**Default for our apps: build ONLY the "Signal way" (own websocket +
`remoteMessaging` foreground service).** It gives instant notifications on every
Android phone — Google or de-Googled — needs no distributor app, and needs nothing
hosted beyond the relay you already have. That's the whole requirement.

Everything else below (UnifiedPush, FCM, WebPush, build flavors) is **optional
battery optimization you can add later**. The tiers are **automatic fallbacks**, not
a menu you pick from or wire by hand — you are NOT required to implement 2/3/4.

- **iOS is unrelated** — it can't do background websockets, so *if and only if* you
  ship an iPhone app, you add Apple's APNs. Building Android only? Ignore all iOS bits.
- **Build "flavors" are an Android-only concept** (F-Droid build vs Play-Store build)
  and are only needed once you add the FCM path. Signal-way alone needs no flavors.

---

**Rule:** Separate two different things:

1. **Local notifications** (the app itself shows a notification) →
   [`flutter_local_notifications`](https://pub.dev/packages/flutter_local_notifications).
   No GMS, no server. Use for reminders, timers, or "app is running in the
   background" foreground-service notifications.
2. **Push** (a server wakes a *closed* app). On Android there are **two** GMS-free
   ways to get instant delivery — use both, in this order:
   - **Baseline: "the Signal way"** — the app holds its **own persistent WebSocket**
     via a `remoteMessaging` foreground service. Works on *any* phone, **needs no
     distributor app**, always available. Cost: battery + a persistent notification.
   - **Optional, battery-saving: UnifiedPush** — a shared distributor socket
     (self-hosted **ntfy**). Lighter, but the user must have a distributor installed.
   - **Play-Store / Google phones:** try **FCM first** (cheapest); fall back to the
     websocket automatically when FCM is absent/failing. One build, embedded FCM
     distributor behind a flavor. **iOS is separate — APNs, always.**

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
**Android kills background connections** when the app is closed. Three ways to keep
delivery working, GMS-free:

- **Own websocket + foreground service** ("the Signal way", see below) — the app
  keeps *its own* socket alive. Instant, no distributor needed, works everywhere.
  Higher battery + a persistent notification. **Use the `remoteMessaging` FGS type**
  (Android 14+) — it's purpose-built for chat and, crucially, is **NOT** subject to
  Android 15's 6h/24h cap (that cap only hits `dataSync`/`mediaProcessing`). So a
  messaging socket is *not* fragile if you pick the right FGS type.
- **UnifiedPush / FCM** — the phone keeps **one** shared, battery-efficient
  connection (via a distributor / Play Services) that can wake **any** app. Lighter
  than a per-app socket, but UnifiedPush needs a distributor installed and FCM needs
  GMS.
- **WorkManager polling** — cheap but not instant (min ~15 min, not guaranteed);
  Android 16 tightens job quotas further. A "slow mode" *floor*, not real push.

The robust design uses **all three tiers**: FCM/UnifiedPush when available (cheap),
own websocket+FGS as the always-works fallback, WorkManager as the last-resort floor.

## "The Signal way" — own websocket + FGS, no distributor (baseline)

This is why Signal (installed from signal.org / GitHub, no Play Store) still gives
**instant** notifications on a de-Googled phone: it doesn't depend on FCM *or* on a
UnifiedPush distributor. It keeps **its own** persistent WebSocket alive in a
foreground service. Self-contained.

How Signal does it (verified against its manifest, 2026):

- **Runtime auto-fallback, not a build split.** Signal always *tries* FCM first (even
  with no Play Services); if FCM registration fails — or fails for 3 days straight —
  it **automatically switches on the websocket**. There's also a manual toggle:
  *Settings → Data and storage → "Stay connected in background."*
- **What the user sees:** a low-importance persistent notification **"Background
  connection enabled"** (the icon with two arrows). The user can silence it.
- **No distributor, nothing else to install.** Contrast with UnifiedPush, which needs
  a separate distributor app.

Recipe for a new app copying this:

- Foreground service with **`android:foregroundServiceType="remoteMessaging"`** +
  permission **`FOREGROUND_SERVICE_REMOTE_MESSAGING`**. This is the type Signal and
  Molly declare for the incoming-message socket. Introduced in Android 14 for exactly
  this "transfer messages device-to-device" purpose.

  ```xml
  <uses-permission android:name="android.permission.FOREGROUND_SERVICE_REMOTE_MESSAGING" />
  <service
      android:name=".push.MessageSocketService"
      android:foregroundServiceType="remoteMessaging"
      android:exported="false" />
  ```

- Hold the websocket + a **partial wakelock**; post a low-importance persistent
  notification (let the user silence it).
- **Try FCM / embedded distributor first; fall back to this websocket** when GMS/FCM
  is absent or failing (Signal's auto-detect pattern).
- Add a **`WorkManager` slow-poll** as the floor for when even the FGS gets killed.
- Ensure the service **restarts from background** after Doze/reboot: `remoteMessaging`
  is allowed to start from the background; re-arm via `BOOT_COMPLETED` + connectivity
  callbacks.

Why `remoteMessaging` specifically (not `dataSync` / `specialUse`):

- **Not capped:** Android 15's 6h/24h FGS timeout hits `dataSync` and
  `mediaProcessing` — **`remoteMessaging` is exempt**, so the chat socket can stay up.
- **No Play justification:** `specialUse` requires a written justification reviewed in
  Play Console (and may be rejected if a standard type fits). `remoteMessaging` is the
  purpose-built standard type → no special review needed.

This is the **baseline** for our apps: it works on every Android phone with zero
external dependencies. Layer UnifiedPush and FCM on top only as battery optimizations.

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

Production checklist (tiered — ship the baseline first, add the rest as optimizations):
1. **Baseline:** app-owned websocket + `remoteMessaging` foreground service +
   WorkManager slow-poll floor. Works on every Android phone, no server push needed
   beyond your existing relay.
2. Auth-locked **ntfy** container + a **WebPush/VAPID sender** in your backend;
   store each user's endpoint; content-free poke → app fetch + decrypt + local
   notification. (Battery-saving path for phones with a distributor.)
3. **Embedded FCM distributor** behind a build flavor for the Play-Store build;
   try FCM first, fall back to the baseline websocket automatically.
4. Separate **APNs** path for iOS (content-free pushes).

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
  in the Play flavor. The baseline websocket+FGS ships in *both* flavors.
- **FGS type matters:** use `remoteMessaging` for a chat socket (exempt from Android
  15's 6h cap, no Play justification). Don't use `dataSync` (capped) or `specialUse`
  (needs review) for it.
- Molly inherits Signal's `remoteMessaging` declaration; SimpleX also runs its own
  FGS (exact type unverified). All confirm: **own-websocket + FGS needs no distributor.**
