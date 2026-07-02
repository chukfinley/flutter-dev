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

## Is this the same ntfy you're thinking of? Yes.

`ntfy` = the open-source pub/sub notification software (the thing where you POST a
topic and it pings a phone). `ntfy.sh` = its **public hosted instance**. Same
software. In UnifiedPush, ntfy is used purely as the **distributor + push server**;
you just never put real content in the message — only a wake-up poke.

## "Anyone can post to a topic" — the security concern, answered

On `ntfy.sh` a topic is public: whoever knows the topic string can subscribe and
publish. That feels broken. Two things make it fine:

1. **The topic is an unguessable random token.** UnifiedPush endpoints look like
   `https://ntfy.example.com/up<40-random-chars>`. Nobody can post to it without the
   URL — it's a capability, like a secret link. Don't log/leak it.
2. **You only ever send an opaque poke.** Even if someone guessed the URL, the worst
   they can do is trigger a **spurious wake-up** (battery annoyance / minor DoS).
   They **cannot** read or inject a message — real content is E2EE and fetched from
   *your* relay, which authenticates the app. No data leak, no spoofed message.

For a **production** app, don't rely on obscurity — **self-host ntfy with auth**:

```yaml
# ntfy server.yml (Docker, one container)
auth-file: /var/lib/ntfy/auth.db
auth-default-access: "deny-all"      # nobody can publish/subscribe by default
```

Then create a **publish token for your backend only**, and per-user subscribe
access. Now only your server can send pokes; users can only receive their own.
That closes the "anyone can post" hole for real.

## Distributor options (production, FOSS)

`ntfy.sh` is just the public instance — you don't have to use it:

- **Self-hosted ntfy** (Docker, one container, your domain, with the auth above) —
  the default recommendation for a production FOSS app. You control it.
- **Mollysocket** — the reference bridge for the Signal fork (Molly): turns
  Signal-style push into UnifiedPush wake-ups. The pattern to copy for a messenger.
- **Matrix Sygnal / sunup** — other self-hostable push gateways if you outgrow ntfy.
- **Embedded FCM distributor** — UnifiedPush falls back to FCM on Google phones with
  no distributor installed.

Recommendation for a production FOSS messenger: **self-hosted ntfy (auth-locked)**
for the F-Droid build, **FCM data-messages as fallback** in the Play Store build.
Both only carry wake-up pokes; message fetch/decrypt is identical either way.

## What you actually have to host + implement

**You host:** one **ntfy server** (Docker container + your domain + the auth config
above). That's it — no per-user infrastructure. Your existing backend/relay gains
one job: after storing a new ciphertext, `POST` an empty poke to the user's endpoint
URL using your publish token.

**The user needs a distributor app** (the ntfy app) installed and pointed at your
ntfy server — this is the one UnifiedPush catch on a de-Googled phone. On Play-Store
Google phones, the FCM fallback covers users who have no distributor.

**App side (Flutter):**

```yaml
dependencies:
  unifiedpush: ^5.0.0
```

```dart
// 1. register once; the distributor hands back an endpoint URL
await UnifiedPush.registerApp(instance: 'default');

// 2. receive the endpoint, send it to YOUR backend to store per-user
onNewEndpoint(String endpoint, String instance) {
  api.savePushEndpoint(endpoint); // backend POSTs pokes here later
}

// 3. a poke arrives -> app is woken -> fetch + decrypt from your relay
onMessage(Uint8List message, String instance) async {
  final ciphertext = await relay.fetchPending(); // your own channel
  final text = crypto.decrypt(ciphertext);       // local
  localNotifications.show(text);                  // flutter_local_notifications
}
```

**Backend side:** `POST https://ntfy.example.com/up<token>` with an empty body and
your `Authorization: Bearer <publish-token>` header → phone wakes.

So the total production checklist: (1) run an auth-locked ntfy container, (2) store
each user's endpoint, (3) POST empty pokes from your relay, (4) app fetches +
decrypts + shows a local notification, (5) ship FCM as the Play-Store fallback.

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
