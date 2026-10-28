# Testing (TDD)

## The loop - every time
1. **Red**: write one small test describing the next behaviour. Run it. Confirm it fails for the right reason (not a compile error you didn't intend).
2. **Green**: write the minimum production code to pass.
3. **Refactor**: clean up with tests green. Run the full suite.

Test files mirror `lib/`: `lib/features/todo/domain/usecases/get_todos.dart` → `test/features/todo/domain/usecases/get_todos_test.dart`.

## What gets tested
| Unit | How |
|---|---|
| Use cases | mocktail-mocked repository |
| Mappers | round-trip, nulls, edge values |
| DTOs | `fromJson` with real API fixture JSON in `test/fixtures/` |
| Remote data sources | mocked `Dio` (mocktail) or `http_mock_adapter`; assert paths, params, parsing |
| Local data sources | real ObjectBox `Store` in a temp directory |
| Repositories | mocked data sources; success, network failure with/without cache, unexpected error |
| Notifiers | `ProviderContainer` with overrides; assert state transitions |
| Router redirect | pure-function tests |
| Screens/widgets | widget tests for loading, data, empty, error; interactions call the notifier |
| Motion math | unit tests for projection, rubber-band, snap-point selection |

## mocktail
```dart
class MockTodoRepository extends Mock implements TodoRepository {}

setUpAll(() => registerFallbackValue(const Todo(id: '', title: '', done: false)));
```

## Notifier tests
```dart
test('emits data from use case', () async {
  final getTodos = MockGetTodos();
  when(() => getTodos()).thenAnswer((_) async => const Ok([todo]));

  final container = ProviderContainer(overrides: [
    getTodosProvider.overrideWithValue(getTodos),
  ]);
  addTearDown(container.dispose);

  expect(await container.read(todoListProvider.future), [todo]);
});
```

## Widget tests
```dart
await tester.pumpWidget(ProviderScope(
  overrides: [todoListProvider.overrideWith(() => FakeTodoList(const [todo]))],
  child: const CupertinoApp(home: TodoListScreen()),
));
```
Also test with `MediaQuery(data: ...copyWith(disableAnimations: true, textScaler: TextScaler.linear(2)))` for reduced-motion and large text.

## ObjectBox in tests
```dart
late Directory dir; late Store store;
setUp(() { dir = Directory.systemTemp.createTempSync('obx'); store = Store(getObjectBoxModel(), directory: dir.path); });
tearDown(() { store.close(); dir.deleteSync(recursive: true); });
```
Host-side tests need the ObjectBox native library installed on the dev machine/CI (ObjectBox provides an install script - check their docs). Include that in the setup report.

## Done means
`dart run build_runner build -d && flutter analyze && dart format --set-exit-if-changed . && flutter test` all pass. Use `flutter test --coverage` when the user asks for coverage numbers.
