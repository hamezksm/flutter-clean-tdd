# State management and widgets

## Riverpod (with riverpod_generator)
- Always use `@riverpod` / `@Riverpod(keepAlive: true)` annotations; never hand-write `Provider(...)` objects.
- Async screen state = `AsyncNotifier` (class-based `@riverpod` with `Future<T> build()`).
- `ref.watch` inside `build`; `ref.read` inside callbacks/methods; `ref.listen` for one-off side effects (navigation, toasts, haptics).
- After every `await` in a notifier method, check `if (!ref.mounted) return;` before touching state.
- Notifiers call use cases only. No Dio, no ObjectBox, no BuildContext.

```dart
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    final result = await ref.watch(getTodosProvider).call();
    return switch (result) {
      Ok(:final value) => value,
      Err(:final failure) => throw failure,
    };
  }

  Future<void> add(String title) async {
    state = const AsyncLoading<List<Todo>>().copyWithPrevious(state);
    final result = await ref.read(addTodoProvider).call(title);
    if (!ref.mounted) return;
    state = switch (result) {
      Ok() => AsyncData([...?state.value, result.value]),
      Err(:final failure) => AsyncError(failure, StackTrace.current),
    };
  }
}
```
(Adapt to the exact API of the installed Riverpod version - check its changelog if the analyzer complains.)

## Widgets: no StatefulWidget, ever
Use `ConsumerWidget` when no lifecycle objects are needed, `HookConsumerWidget` when they are. `StatefulWidget`, `ConsumerStatefulWidget` and `setState` must not appear anywhere in `lib/`. Verify before finishing a task:
```
grep -rnE "StatefulWidget|setState\(" lib/ && echo "VIOLATION"
```

### What hooks are allowed to do
Hooks are for **lifecycle objects and purely visual state only**:
- `useTextEditingController`, `useFocusNode`, `useScrollController`, `usePageController`, `useTabController`
- `useAnimationController` (+ `useAnimation`) for motion
- `useState` ONLY for visual-only state: expanded/collapsed, obscure-password toggle, pressed highlight
- `useEffect` for subscribing/unsubscribing to things owned by the widget

Anything that affects data, navigation, validation results or business rules lives in a Riverpod notifier, where TDD unit tests can reach it. If you're tempted to `useState` a loading flag, a selected entity, or form validity - it belongs in a notifier.

### Rules of hooks (bugs if broken)
1. Call hooks only inside `build` of a Hook widget (or inside custom `useX` functions).
2. Call them unconditionally and in the same order every build: no hooks inside `if`, loops, or callbacks.
3. Custom hooks are functions named `useSomething` that only call other hooks.

```dart
class LoginScreen extends HookConsumerWidget {
  const LoginScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final email = useTextEditingController();
    final password = useTextEditingController();
    final obscure = useState(true); // visual only
    final state = ref.watch(loginControllerProvider);

    ref.listen(loginControllerProvider, (_, next) {
      if (next is AsyncData<Session?> && next.value != null) const HomeRoute().go(context);
    });

    return LoginForm(
      email: email,
      password: password,
      obscure: obscure.value,
      onToggleObscure: () => obscure.value = !obscure.value,
      isLoading: state.isLoading,
      error: state.error,
      onSubmit: () => ref.read(loginControllerProvider.notifier)
          .login(email.text, password.text),
    );
  }
}
```
Keep screens thin; split into small `const` stateless widgets that take plain values and callbacks. Use `const` constructors everywhere possible.
