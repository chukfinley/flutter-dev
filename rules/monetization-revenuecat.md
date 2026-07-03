# Monetization: RevenueCat, server-truth entitlements

**Rule:** All in-app purchases / subscriptions go through
[RevenueCat](https://pub.dev/packages/purchases_flutter). The entitlement (is this
user Pro?) is **server-side truth**, never a client-only flag. RevenueCat's
`app_user_id` **must equal your backend user id** so RC webhooks and your backend
agree on who paid.

## Why RevenueCat

- One SDK wraps Google Play Billing, App Store, Amazon, and web (Stripe) — you don't
  hand-roll billing per store.
- Its dashboard + webhooks give you the receipt validation and the "is Pro" answer
  without running your own billing backend.
- Proven pattern in ai-memes: **RC Pro = server truth**, `app_user_id == backend
  user id`, entitlement checked server-side before unlocking paid features.

## Fix

```yaml
dependencies:
  purchases_flutter: ^8.0.0
```

```dart
await Purchases.configure(
  PurchasesConfiguration(apiKey)..appUserID = backendUserId, // MUST match backend
);
final info = await Purchases.getCustomerInfo();
final isPro = info.entitlements.active.containsKey('pro');
```

- Gate real value **on the backend**, keyed off the RC webhook / a server check.
  Treat the client `isPro` as UI hint only — a modified client must not unlock paid
  server features.
- Wire the **RevenueCat webhook** to your backend so entitlement changes (renew,
  refund, chargeback) update your own DB.

## FOSS vs. paid — the honest tradeoff

- Billing needs **Google Play Billing** (proprietary) → a paid app **cannot** be a
  pure GMS-free F-Droid build. So: **paid features live in the Play-Store build.**
- We still make the **app source** as open as sensible, but a paid app has closed
  API/billing dependencies and likely won't pass IzzyOnDroid's anti-feature scan.
  **That's fine — we don't care whether paid apps land on IzzyOnDroid.** Money first.
- The **F-Droid / free build** (if any) ships without billing — free, or
  donation-only (e.g. a "support" link, Liberapay), no Play billing lib.

## Two app types — but FOSS components stay the default

You'll be told up front which kind you're building:

- **FOSS app** → maximize free & open-source; push every FOSS-specific rule hard.
- **Monetization app** → money is the priority, but **still default to FOSS /
  open-source components** wherever they don't hurt the money goal. Reach for a
  proprietary dependency (FCM, Google Maps, Crashlytics, Play Billing) **only** when
  it genuinely converts better or is required — not by default.

The architecture/quality rules (Riverpod, structure, freezed, dio, testing, signing)
**apply to every app**, open or closed. The FOSS-specific rules are mandatory for a
FOSS app and the preferred default for a money app.

## Notes / gotchas

- **`app_user_id` mismatch is the classic bug:** if RC's user id ≠ your backend id,
  the webhook updates a user your backend doesn't know, and Pro silently never
  unlocks. Set it explicitly at configure time; don't rely on RC's anonymous id.
- Handle restore purchases (`Purchases.restorePurchases()`).
- Refunds/chargebacks arrive via webhook — revoke access server-side, or you leak
  paid features (and money).
