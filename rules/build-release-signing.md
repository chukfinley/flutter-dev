# Building & signing: release APK, per-app keystore, back it up

**Rule:** Always build **`--release`**, never ship or test-measure a debug build.
Every app gets its **own signing keystore**, generated when the app is created,
stored outside the repo, and **backed up**. Default output is a **universal
release APK** (works everywhere, sideload/F-Droid friendly).

## Always release, never debug

```bash
flutter build apk --release
```

Debug builds are **much** slower on-device (no AOT, JIT + assertions) and behave
differently. Never judge performance or ship from a debug build. On desktop the
difference is huge; on Android it's still real. Test the release build.

## Keystore: one per app, generate up front, back it up

A keystore signs the app. **If you lose the keystore, you can never update that
app again** — Google/F-Droid reject an update signed by a different key. That's a
money/reputation risk (a published app you can't patch).

So, on **every new app**, immediately:

1. Generate a **fresh** keystore (new key per app — don't reuse one key across apps):

   ```bash
   keytool -genkey -v -keystore <appname>.jks \
     -keyalg RSA -keysize 2048 -validity 10000 -alias <appname>
   ```

2. Store it **outside the git repo** in a consistent place (e.g.
   `~/keystores/<appname>.jks`) — or ask where to save it — and **back it up**
   (password manager / encrypted backup). Never commit the `.jks` or its passwords.

3. Wire it via a **gitignored** `android/key.properties`:

   ```properties
   storeFile=/home/user/keystores/<appname>.jks
   storePassword=...
   keyAlias=<appname>
   keyPassword=...
   ```

   (Reference it from `android/app/build.gradle`'s `signingConfigs`.)

`key.properties`, `*.jks`, `*.keystore` go in `.gitignore`.

## APK vs AAB vs split-per-abi (which output)

- **Universal APK** (`flutter build apk`) — one file, all CPU architectures, works
  everywhere. Biggest file. **Default** for direct download / F-Droid / sideload.
- **Split per ABI** (`flutter build apk --split-per-abi`) — one smaller APK per CPU
  architecture (arm64, armeabi, x86_64). Each is smaller than the universal because
  it drops the other architectures' native code. Use when you distribute APKs
  yourself and want smaller downloads; the user needs the one matching their phone.
- **App Bundle / AAB** (`flutter build appbundle`) — **Play Store only**. You upload
  one `.aab`; Google generates and signs a minimal per-device APK for each user
  (smallest download of all). The Play Store *requires* AAB for new apps. It is not
  installable directly — it's an upload format, not a shippable file.

Rule of thumb: **universal APK** for F-Droid/direct, **AAB** for the Play Store,
**split-per-abi** when you self-distribute and care about size.

## Notes / gotchas

- R8/shrinking (`minifyEnabled`) further reduces size for release builds — on by
  default for release; keep it.
- With AAB, Google holds a signing key (Play App Signing) — you still keep your
  **upload** key safe; losing it means re-registering the upload key with Google.
- Back up the keystore the day you create it, not "later". Later never comes.
