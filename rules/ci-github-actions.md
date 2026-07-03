# CI: GitHub Actions builds the release APK

**Rule:** A GitHub Actions workflow analyzes, tests, and builds a **release APK** —
on every push (analyze + test) and on a version **tag** (build + attach to a
Release). No "works on my machine" releases.

## Fix

```yaml
# .github/workflows/build.yml
name: build
on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          channel: stable
          flutter-version: ${{ vars.FLUTTER_VERSION }}   # keep in sync with fvm pin
      - run: flutter pub get
      - run: dart analyze
      - run: flutter test
      # release build only on a version tag
      - if: startsWith(github.ref, 'refs/tags/v')
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > android/app/key.jks
          cat > android/key.properties <<EOF
          storeFile=key.jks
          storePassword=${{ secrets.KEYSTORE_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          EOF
      - if: startsWith(github.ref, 'refs/tags/v')
        run: flutter build apk --release
      - if: startsWith(github.ref, 'refs/tags/v')
        uses: softprops/action-gh-release@v2
        with:
          files: build/app/outputs/flutter-apk/app-release.apk
```

## Notes / gotchas

- **Keystore stays out of git.** Store it base64 in a GitHub secret
  (`KEYSTORE_BASE64`) + the passwords as secrets; the workflow reconstructs
  `key.jks` / `key.properties` at build time. See `build-release-signing.md`.
- `dart analyze` + `flutter test` on every PR gates bad code before merge.
- Tag-triggered build (`v1.2.3`) keeps the published APK reproducible from a commit.
- For F-Droid the store builds from source itself — this workflow is for your own
  direct-download / GitHub Releases distribution.
- Keep `FLUTTER_VERSION` matching the fvm pin (`flutter-version-fvm.md`) so CI and
  local builds agree.
