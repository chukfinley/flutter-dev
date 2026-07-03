# Flutter version: pin with fvm, but keep it updated

**Rule:** Pin the Flutter version per project with [fvm](https://fvm.app) so builds
are reproducible and CI matches local. **But** we track the latest Flutter — so
*deliberately update the pin often*, don't let it rot on an old version.

## Why (and the caveat)

Different Flutter versions produce different behavior/output; "works on my machine"
usually means mismatched SDK versions. Pinning fixes that. The tension: we *want* the
newest Flutter (new Material 3 Expressive, API fixes), so the pin is a **moving**
pin — update it on purpose, commit the bump, don't treat it as frozen.

## Fix

```bash
fvm install stable         # or a specific version
fvm use 3.35.0             # writes .fvmrc, pins this project
fvm flutter run            # use fvm-prefixed commands
```

- Commit **`.fvmrc`** (the pinned version) so everyone + CI use the same SDK.
- CI reads the same version (`flutter-version` in `ci-github-actions.md` → keep in
  sync with `.fvmrc`).

## Update discipline

- Bump the pin regularly (we stay current): `fvm use <newer>`, run `flutter pub get`,
  `dart analyze`, `flutter test`, smoke-test, commit the bump.
- Update CI's `FLUTTER_VERSION` in the **same** commit so they never drift.
- Treat a version bump like any change: it goes through the tests, not straight to a
  release.

## Notes / gotchas

- Add `.fvm/` (the cached SDK symlink dir) to `.gitignore`; commit only `.fvmrc`.
- If a teammate/CI ignores fvm and uses a system Flutter, the pin does nothing — use
  `fvm flutter ...` everywhere (or an alias) so the pinned SDK is actually used.
- Keeping the pin current is the rule, not pinning-and-forgetting — an outdated pin is
  as much a problem as no pin.
