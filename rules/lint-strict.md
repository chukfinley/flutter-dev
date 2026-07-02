# Lint: turn on strict analysis from day one

**Rule:** Every app has strict linting enabled via an `analysis_options.yaml`.
Use a strong ruleset — [`very_good_analysis`](https://pub.dev/packages/very_good_analysis)
(strictest, recommended) or at least the official `flutter_lints`.

## What a linter is (plain version)

A **linter** is an automatic code checker. It reads your Dart code without running
it and warns about mistakes, risky patterns, and messy style — e.g. an unused
variable, a missing `await`, a widget rebuilt for no reason, a `null` that could
crash. Think of it as a reviewer that runs instantly, every save.

Why it matters even if AI writes the code: the linter catches the class of bugs
that compile fine but crash or misbehave at runtime, and it keeps every app's
style identical so nothing looks "off" when you open a project later.

## Fix

Add a dev dependency and an `analysis_options.yaml` at the project root:

```yaml
# pubspec.yaml
dev_dependencies:
  very_good_analysis: ^6.0.0   # or: flutter_lints: ^4.0.0
```

```yaml
# analysis_options.yaml (project root)
include: package:very_good_analysis/analysis_options.yaml

analyzer:
  language:
    strict-casts: true
    strict-raw-types: true
  errors:
    # make the important ones hard errors, not just hints:
    invalid_await_not_future: error
    missing_required_param: error
    unawaited_futures: error
```

Then it runs automatically in the IDE, and on demand:

```bash
dart analyze          # fails CI if anything is flagged
dart format .         # consistent formatting
```

## Notes / gotchas

- Wire `dart analyze` into CI so a lint failure blocks a bad build.
- `very_good_analysis` is intentionally strict (unawaited futures, required
  trailing commas, etc.) — that strictness is the point; it prevents whole
  categories of bugs. Drop to `flutter_lints` only if the strict set is too noisy
  for a given project.
- Fix lints, don't blanket-ignore them. If a rule genuinely doesn't fit, disable
  that one rule in `analysis_options.yaml` with a comment saying why.
