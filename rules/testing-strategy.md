# Testing: write Flutter tests, run them in CI

**Rule:** Write tests for the things that matter and run them in CI. Three levels,
used where each fits:

- **Unit** — pure logic (parsing, calculations, Riverpod notifiers, crypto). Fast,
  most numerous.
- **Widget** — a widget/screen renders and reacts correctly (`flutter_test`,
  `WidgetTester`).
- **Integration** — a real end-to-end flow on a device/emulator
  (`integration_test`): login, send a message, complete a purchase.

## Fix

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter
  mocktail: ^1.0.0   # mocks without code-gen
```

```dart
// test/todo_notifier_test.dart  (unit)
test('completing a todo moves it to done', () {
  final n = TodoNotifier();
  n.add('buy milk');
  n.complete(0);
  expect(n.state.first.done, isTrue);
});
```

```dart
// test/home_page_test.dart  (widget)
testWidgets('shows empty state when no todos', (tester) async {
  await tester.pumpWidget(const ProviderScope(child: MyApp()));
  expect(find.text('Nothing here yet'), findsOneWidget);
});
```

```bash
flutter test                                   # unit + widget
flutter test integration_test                  # e2e on a device
```

## What to always test

- **Anything security/money:** certificate pinning rejects a bad cert
  (`http-dio-cert-pinning.md`), entitlement gating, E2EE encrypt/decrypt round-trip.
- **Core business logic** in Riverpod notifiers / domain layer.
- **Regression cases:** every fixed bug gets a test so it can't come back.

## Notes / gotchas

- Wire `flutter test` into CI (`ci-github-actions.md`) so a red test blocks merge.
- Don't chase 100% coverage — test logic and critical flows, skip trivial glue.
- `mocktail` avoids build_runner for mocks; keep tests fast so they actually get run.
- Widget tests use a fake clock/network — never hit the real backend in unit/widget
  tests; reserve that for `integration_test`.
