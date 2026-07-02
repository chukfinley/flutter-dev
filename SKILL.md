---
name: flutter-dev-playbook
description: >
  Hard-won standards and go-to resources for building and shipping Flutter apps —
  package choices, FOSS/F-Droid compliance, Play Store publishing, tooling. Use
  when writing, reviewing, or releasing a Flutter app and you want the "do it this
  way, not the obvious way" answer. Triggers: "Flutter", "F-Droid", "IzzyOnDroid",
  "Play Store publish", "closed testing", "geolocator", "FOSS app", "no GMS".
---

# Flutter Dev Playbook

Memory bank of Flutter decisions that are easy to get wrong. Before choosing a
package or a release step, check the matching file in `rules/`.

## Rules index

- **State management** → `rules/state-riverpod.md`
  Always Riverpod. `setState` only for purely local throwaway widget state.
- **Location / GPS** → `rules/location-libre-location.md`
  Use `libre_location`, never `geolocator` (pulls Google Play Services → non-free flag).
- **Play Store publishing** → `rules/play-store-testers.md`
  Get the required 12 closed testers via testerscommunity.com/submit-app.

_(More rules get added as files under `rules/`. Always check there first.)_

## General principles

- **FOSS builds:** before adding a dependency, ask *does it link Google Play
  Services or another proprietary blob?* If yes, find the LocationManager-level /
  platform-native / self-hosted alternative. A working app flagged `NonFreeDep`
  on IzzyOnDroid is a distribution failure, not a success.
- **Publishing:** solve the closed-testing / tester requirements early — they gate
  the whole release, not the code.
