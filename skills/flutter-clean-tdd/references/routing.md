# Routing: go_router + go_router_builder

go_router is maintained by the Flutter team and considered feature-complete (bug-fix focus). Use typed routes via `go_router_builder` so navigation is compile-checked - no string paths in widgets.

## Router as a Riverpod provider
```dart
@Riverpod(keepAlive: true)
GoRouter router(Ref ref) {
  final auth = ValueNotifier<AuthStatus>(AuthStatus.unknown);
  ref.listen(authStatusProvider, (_, next) => auth.value = next, fireImmediately: true);
  ref.onDispose(auth.dispose);

  return GoRouter(
    routes: $appRoutes,               // generated
    refreshListenable: auth,
    redirect: (context, state) {
      final status = auth.value;
      final loggingIn = state.matchedLocation == const LoginRoute().location;
      if (status == AuthStatus.unknown) return const SplashRoute().location;
      if (status == AuthStatus.signedOut) return loggingIn ? null : const LoginRoute().location;
      if (loggingIn) return const HomeRoute().location;
      return null;
    },
  );
}
```
Redirect logic is a pure function of state - extract it and unit-test it.

## Typed routes
```dart
@TypedGoRoute<HomeRoute>(path: '/', routes: [
  TypedGoRoute<TodoDetailRoute>(path: 'todos/:id'),
])
class HomeRoute extends GoRouteData with _$HomeRoute {
  const HomeRoute();
  @override
  Widget build(BuildContext context, GoRouterState state) => const HomeScreen();
}

class TodoDetailRoute extends GoRouteData with _$TodoDetailRoute {
  const TodoDetailRoute({required this.id});
  final String id;
  @override
  Widget build(BuildContext context, GoRouterState state) => TodoDetailScreen(id: id);
}

// usage
TodoDetailRoute(id: todo.id).push(context);
```
The exact generated API (mixin vs extension, `$appRoutes` naming) changes between go_router_builder versions - read the README of the installed version and match it.

- Tabs: `StatefulShellRoute.indexedStack` (it's a route type, not a StatefulWidget - allowed).
- On iOS, return `CupertinoPage` via `buildPage` where the default transition isn't right.
- Pass IDs in routes, not whole objects; screens load data through providers. This keeps deep links working.
