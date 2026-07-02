# Data models: freezed + json_serializable

**Rule:** Write data/model classes with
[`freezed`](https://pub.dev/packages/freezed) (+ `json_serializable` for JSON).
Don't hand-write model classes with manual `copyWith`, `==`, `toJson`.

## What this solves (plain version)

A "model" is a class that holds data — a `User`, a `Message`, a `Trip`. By hand,
a correct one needs: a constructor, `==`/`hashCode` (so two equal objects compare
equal), `copyWith` (make a changed copy without mutating the original — Riverpod
relies on this), `toString`, and `toJson`/`fromJson` for API data. That's ~60
lines of boilerplate per model, and one mistake (forgetting a field in `==`)
causes subtle bugs where the UI doesn't rebuild.

`freezed` generates all of it from a short declaration. Immutable by default,
which is exactly what Riverpod wants.

## Fix

```yaml
dependencies:
  freezed_annotation: ^2.4.0
  json_annotation: ^4.9.0
dev_dependencies:
  freezed: ^2.5.0
  json_serializable: ^6.8.0
  build_runner: ^2.4.0
```

```dart
import 'package:freezed_annotation/freezed_annotation.dart';
part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    @Default(false) bool isAdmin,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

```bash
dart run build_runner watch -d   # regenerates .freezed.dart / .g.dart on save
```

Now you get `==`, `hashCode`, `copyWith`, `toString`, and JSON for free:

```dart
final u2 = user.copyWith(name: 'Neo');   // immutable copy
```

## Notes / gotchas

- Requires `build_runner` running (same as Riverpod code-gen — one watch process
  covers both).
- Commit the generated `*.freezed.dart` / `*.g.dart`? Team preference; simplest is
  to gitignore them and regenerate, but committing avoids "did you run build_runner"
  friction.
- `freezed` also does sealed/union types (`@freezed` with multiple factories) —
  handy for state machines and `AsyncValue`-like result types.
- Models live in the feature's `domain/` folder (see `project-structure.md`).
