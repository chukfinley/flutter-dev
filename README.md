# Flutter Dev Playbook

Personal + AI-readable memory bank for building & shipping Flutter apps.

Every time a "do it *this* way / use *this* thing, or you'll waste hours" lesson
comes up, it gets one file in [`rules/`](rules/) so it's never forgotten again.
Covers coding decisions, package choices, publishing, tooling — whatever bites.

## How to use

**Human:** browse [`rules/`](rules/). Each file = one decision or resource, with the *why*.

**AI (Claude Code / Codex / etc.):** drop [`SKILL.md`](SKILL.md) into your skills dir,
or point the agent at this repo. Rules are written to be loaded as context.

## Rules

| Area | Rule | Why |
|------|------|-----|
| Location | [Use `libre_location`, not `geolocator`](rules/location-libre-location.md) | `geolocator` pulls Google Play Services → IzzyOnDroid flags app as non-free |
| Publishing | [Get Play Store testers via TestersCommunity](rules/play-store-testers.md) | Google now requires 12 testers × 14 days before you can publish |

_Add a row above whenever you add a file in `rules/`._

## Format

One rule = one file. Keep it simple:

```
# <Short title>

**Rule:** <the one-line standard / tip>

**Problem:** <what the obvious/default choice breaks or costs you>

**Fix:** <concrete package, link, or steps>

**Notes / gotchas:** <edge cases>
```

License: [The Unlicense](LICENSE) — public domain, copy freely.
