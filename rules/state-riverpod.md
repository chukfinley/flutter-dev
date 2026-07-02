# State management: always Riverpod

**Rule:** Every Flutter app uses [Riverpod](https://riverpod.dev/)
(`flutter_riverpod`, current 3.x) for state management. No exceptions, no
per-project debate. The only thing `setState` is still allowed for is purely
local, throwaway widget state (animation controllers, a text field's focus,
an expand/collapse toggle) that nothing else reads.

## Problem

Plain Flutter state gets painful fast:

- `setState` keeps state inside one widget. Sharing it across screens means
  lifting it up and threading it through constructors — turns into spaghetti.
- `InheritedWidget` / the old `provider` package are coupled to the widget tree,
  need `BuildContext`, and carry a lot of boilerplate.
- Async data (API, DB) forces manual `isLoading` / `error` flags everywhere.

Picking a different solution per app also means every codebase looks different,
which is wasted effort when the code is generated anyway.

## Fix

Standardize on Riverpod, everywhere:

- State lives **outside** the widget tree but stays scoped, auto-disposed, and
  testable. Read it without `BuildContext`.
- **Compile-safe** — misuse is a compile error, not a runtime crash.
- **Async first-class** — `FutureProvider` / `StreamProvider` hand you
  `AsyncValue` (loading / error / data) for free; no manual flag juggling.
- **Auto-dispose** — state cleans itself up when nothing listens.

```yaml
# pubspec.yaml
dependencies:
  flutter_riverpod: ^3.0.0   # check pub.dev for current version
  riverpod_annotation: ^3.0.0
dev_dependencies:
  riverpod_generator: ^3.0.0
  build_runner: ^2.4.0
```

```dart
// wrap the app once
void main() => runApp(const ProviderScope(child: MyApp()));

// a provider (code-gen style)
@riverpod
Future<List<Todo>> todos(TodosRef ref) => api.fetchTodos();

// consume it — loading/error/data handled explicitly
class TodoList extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todos = ref.watch(todosProvider);
    return todos.when(
      data: (list) => ListView(children: [/* ... */]),
      loading: () => const CircularProgressIndicator(),
      error: (e, _) => Text('$e'),
    );
  }
}
```

## Notes / gotchas

- Prefer the **code-gen** flavor (`@riverpod` + `riverpod_generator`); run
  `dart run build_runner watch -d` during dev. Manual `NotifierProvider` also
  works if you want zero code-gen.
- Use `Notifier` / `AsyncNotifier` for mutable state, plain providers for
  derived/read-only values.
- Don't reach for BLoC, GetX, signals, or `provider` — the point of this rule is
  one consistent choice across all apps.
- Riverpod is by the same author as the old `provider` package (name is an
  anagram); it's the mature successor, not an experiment.
