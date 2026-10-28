# Architecture

## Layers and the dependency rule
```
presentation  ──►  domain  ◄──  data
(widgets, notifiers)  (entities, repo interfaces, use cases)  (DTOs, local models, data sources, repo impls)
```
- **domain** is pure Dart. Allowed imports: `dart:*`, `freezed_annotation`, `meta`. Never Flutter, Dio, ObjectBox, json_annotation or Riverpod.
- **data** implements domain interfaces. It is the only layer that knows about Dio, ObjectBox and JSON.
- **presentation** talks to use cases (or repository interfaces for trivial reads). Never to data sources or Dio directly.
- Cross-layer wiring lives only in Riverpod providers.

## Folder structure (feature-first)
```
lib/
  main.dart                     # opens Store, runs ProviderScope with overrides
  app/
    app.dart                    # CupertinoApp.router / MaterialApp.router
    router/                     # go_router provider + typed routes
  core/
    error/failure.dart          # sealed Failure
    error/result.dart           # sealed Result<T>
    network/dio_provider.dart
    storage/objectbox_provider.dart
    ui/theme/                   # tokens, typography, colors (light + dark)
    ui/motion/                  # springs, projection, rubber-band (pure Dart, unit-tested)
  features/<feature>/
    domain/
      entities/
      repositories/             # abstract interface class
      usecases/
    data/
      models/                   # *_dto.dart (JSON), *_local.dart (ObjectBox @Entity)
      mappers/
      datasources/              # <x>_remote_data_source.dart, <x>_local_data_source.dart
      repositories/             # <x>_repository_impl.dart
    presentation/
      providers/                # notifiers + DI providers for this feature
      screens/
      widgets/
test/                           # mirrors lib/ exactly
```

## SOLID, concretely
- **S**: one use case = one class = one public `call`. A data source does I/O only; mapping lives in mappers; error translation lives in the repository.
- **O**: new behaviour = new class (new use case, new data source implementation), not edits to stable ones.
- **L**: every implementation of a repository/data source interface must pass the same contract tests.
- **I**: keep interfaces small. Split read/write repositories if a consumer only needs one side.
- **D**: everything depends on `abstract interface class` types. Concrete classes are named only inside providers.

## DI with Riverpod (no get_it)
Every dependency is a provider typed as its interface:
```dart
@Riverpod(keepAlive: true)
TodoRemoteDataSource todoRemoteDataSource(Ref ref) =>
    TodoRemoteDataSourceImpl(ref.watch(dioProvider));

@Riverpod(keepAlive: true)
TodoRepository todoRepository(Ref ref) => TodoRepositoryImpl(
      remote: ref.watch(todoRemoteDataSourceProvider),
      local: ref.watch(todoLocalDataSourceProvider),
    );

@riverpod
GetTodos getTodos(Ref ref) => GetTodos(ref.watch(todoRepositoryProvider));
```
Async-initialised singletons (the ObjectBox `Store`) are declared as providers that throw by default and are overridden in `main.dart`:
```dart
@Riverpod(keepAlive: true)
Store objectBoxStore(Ref ref) => throw UnimplementedError('Override in main()');

// main.dart
final store = await openStore(directory: ...);
runApp(ProviderScope(
  overrides: [objectBoxStoreProvider.overrideWithValue(store)],
  child: const App(),
));
```
Tests swap any layer with `overrides:` - that's the whole point.

## Error handling
- Data sources throw (DioException, ObjectBox exceptions, custom `ServerException`/`CacheException`).
- Repositories catch everything and return `Result<T>` with a typed `Failure` - nothing throws past the repository.
- Use cases return `Result<T>`.
- Notifiers turn failures into `AsyncError` (or a state field) for the UI.

```dart
sealed class Failure { const Failure(this.message); final String message; }
final class NetworkFailure extends Failure { const NetworkFailure(super.message); }
final class ServerFailure extends Failure { const ServerFailure(super.message, {this.statusCode}); final int? statusCode; }
final class CacheFailure extends Failure { const CacheFailure(super.message); }
final class UnauthorizedFailure extends Failure { const UnauthorizedFailure(super.message); }

sealed class Result<T> { const Result(); }
final class Ok<T> extends Result<T> { const Ok(this.value); final T value; }
final class Err<T> extends Result<T> { const Err(this.failure); final Failure failure; }
```
Use `switch` with pattern matching on these - the compiler enforces exhaustiveness.
