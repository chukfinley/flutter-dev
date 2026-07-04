# Testing: write Flutter tests, run them in CI

**Rule:** Write tests for the things that matter and run them in CI. Four levels,
used where each fits:

- **Unit** — pure logic (parsing, calculations, Riverpod notifiers, crypto). Fast,
  most numerous.
- **Widget** — a widget/screen renders and reacts correctly (`flutter_test`,
  `WidgetTester`).
- **Contract** — a test that hits the **real external API** and asserts its
  response actually has the shape/fields you're coding against. Run it **before**
  building the feature on top. See "Contract-test any network API first" below.
- **Integration** — a real end-to-end flow on a device/emulator
  (`integration_test`): login, send a message, complete a purchase.

## Contract-test any network API first

If the app makes network requests against an API you don't control (OSM Overpass
POI, a REST backend, any third-party endpoint), your **assumption about the
response is the bug** — write a Dart test that calls the live endpoint and asserts
the fields you depend on exist and mean what you think, then run it *before* you
wire the feature. This is exactly the lesson from the maps app: OSM POI came back
in a different shape than assumed, cost hours; one contract test up front would
have caught it instantly.

```dart
// test/overpass_poi_contract_test.dart
// Tagged so it only runs when we choose (real network, can be flaky/offline).
@Tags(['contract'])
library;

import 'package:test/test.dart';
import 'package:dio/dio.dart';

void main() {
  test('Overpass returns restaurants with the tags we rely on', () async {
    final res = await Dio().post<Map<String, dynamic>>(
      'https://overpass-api.de/api/interpreter',
      data: '[out:json];node["amenity"="restaurant"](54.30,10.10,54.34,10.16);out 5;',
      options: Options(contentType: Headers.formUrlEncodedContentType),
    );
    final elements = res.data!['elements'] as List;
    expect(elements, isNotEmpty);
    final first = elements.first as Map<String, dynamic>;
    // These are the exact fields our parser reads — prove they're here:
    expect(first['lat'], isA<num>());
    expect(first['lon'], isA<num>());
    expect((first['tags'] as Map)['name'], isNotNull);
  });
}
```

```bash
flutter test --tags contract      # run the real-API checks on demand / nightly
```

- Run contract tests **before coding the feature** (verify the API), and on a
  **nightly CI job** (catch the API changing under you) — not on every PR, so a
  third-party outage can't block merges.
- When a contract test reveals the real shape, feed that shape straight into your
  `freezed` model (`models-freezed.md`) and `dio` client (`http-dio-cert-pinning.md`).

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

- **Every external API you consume:** a contract test proving its response shape
  before you build on it (see above). Wrong assumptions about a third-party API are
  the classic hours-wasting bug.
- **Anything security/money:** certificate pinning rejects a bad cert
  (`http-dio-cert-pinning.md`), entitlement gating, E2EE encrypt/decrypt round-trip.
- **Core business logic** in Riverpod notifiers / domain layer.
- **Regression cases:** every fixed bug gets a test so it can't come back.

## Notes / gotchas

- Wire `flutter test` into CI (`ci-github-actions.md`) so a red test blocks merge.
- Don't chase 100% coverage — test logic and critical flows, skip trivial glue.
- `mocktail` avoids build_runner for mocks; keep tests fast so they actually get run.
- Unit/widget tests use a fake clock/network — never hit the real backend there.
  Hitting a **real** endpoint is only for **contract** tests (tagged, run on
  demand/nightly) and `integration_test` — keep it out of the per-PR unit/widget run.
