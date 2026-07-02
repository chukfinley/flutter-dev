# Routing: go_router

**Rule:** Use [`go_router`](https://pub.dev/packages/go_router) (the official
Flutter routing package) for navigation between screens. Define all routes in one
place.

## What "routing" even means here

"Routing" = how the app moves from one screen to another and how it knows which
screen a deep link (e.g. `myapp://trip/42` or a web URL) should open.

Plain Flutter gives you `Navigator.push(...)` scattered across the code. That
works for a tiny app but:

- there's no single list of "what screens exist",
- deep links / back button / web URLs are painful,
- passing data between screens is ad-hoc and easy to get wrong.

`go_router` replaces that with **one declared route table**. It's maintained by
the Flutter team, so it's the safe long-term default.

## Fix

One file lists every route; navigate by path.

```yaml
dependencies:
  go_router: ^14.0.0
```

```dart
final router = GoRouter(
  debugLogDiagnostics: false, // <-- keep the noisy route logs OUT of the terminal
  routes: [
    GoRoute(path: '/', builder: (c, s) => const HomeScreen()),
    GoRoute(
      path: '/trip/:id',
      builder: (c, s) => TripScreen(id: s.pathParameters['id']!),
    ),
  ],
);

MaterialApp.router(routerConfig: router);
```

```dart
context.go('/trip/42');   // replace stack
context.push('/settings'); // push on top
```

## Notes / gotchas

- **The "weird terminal logs" are just `debugLogDiagnostics: true`.** go_router
  prints every route change when that's on. Set it to `false` and the terminal is
  quiet — that noise was config, not a problem with the package.
- go_router is a *recommendation*, not a hard law like Riverpod. It's the modern
  default and worth using on new apps, but it's not worth rewriting a working app
  that already uses plain `Navigator`.
- Pair typed params with the app's models; for fully type-safe routes there's
  `go_router_builder` (code-gen) if you want it.
- Keep the whole route table in **one file** (`lib/router.dart` or
  `lib/core/router/`) so screens are never "hidden" behind a random push call.
