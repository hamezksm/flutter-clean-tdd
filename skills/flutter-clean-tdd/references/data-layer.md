# Data layer

## Dio
One `Dio` instance per base URL, provided by Riverpod:
```dart
@Riverpod(keepAlive: true)
Dio dio(Ref ref) {
  final dio = Dio(BaseOptions(
    baseUrl: const String.fromEnvironment('API_BASE_URL'),
    connectTimeout: const Duration(seconds: 15),
    receiveTimeout: const Duration(seconds: 15),
  ));
  dio.interceptors.addAll([
    AuthInterceptor(ref),            // reads token via a provider, handles 401 refresh
    if (kDebugMode) LogInterceptor(requestBody: true, responseBody: true),
  ]);
  return dio;
}
```
- Config comes from `--dart-define-from-file=env/<flavor>.json`. Never hardcode URLs or keys; keep env files with secrets out of git and list them in the setup report.
- Tokens live in `flutter_secure_storage` behind a `TokenStorage` interface.

## Remote data source
```dart
abstract interface class TodoRemoteDataSource {
  Future<List<TodoDto>> fetchTodos();
}

final class TodoRemoteDataSourceImpl implements TodoRemoteDataSource {
  TodoRemoteDataSourceImpl(this._dio);
  final Dio _dio;

  @override
  Future<List<TodoDto>> fetchTodos() async {
    final res = await _dio.get<List<dynamic>>('/todos');
    return res.data!.map((e) => TodoDto.fromJson(e as Map<String, dynamic>)).toList();
  }
}
```
Data sources don't catch errors - the repository does.

## ObjectBox
- Open once in `main()` with `openStore()` from the generated `objectbox.g.dart`, using a directory from `path_provider`; inject via the overridden `objectBoxStoreProvider`.
- On macOS (sandboxed) pass the app group: `openStore(macosApplicationGroup: '<TEAMID>.<group>')` and add the group to the entitlements - verify against current ObjectBox docs.
- A local data source wraps `Box<T>`:
```dart
final class TodoLocalDataSourceImpl implements TodoLocalDataSource {
  TodoLocalDataSourceImpl(Store store) : _box = store.box<TodoLocal>();
  final Box<TodoLocal> _box;

  @override
  List<TodoLocal> getAll() => _box.getAll();

  @override
  void putMany(List<TodoLocal> items) => _box.putMany(items);

  @override
  Stream<List<TodoLocal>> watchAll() =>
      _box.query().watch(triggerImmediately: true).map((q) => q.find());
}
```
- Reactive lists: expose `watch` streams through the repository and consume them with a `StreamNotifier`/stream provider.
- Use `@Index` on queried fields; use `putMany`/transactions for batches.

## Repository implementation
Translates exceptions to `Failure`s and decides caching strategy (default: network-first, fall back to cache, write-through on success):
```dart
@override
Future<Result<List<Todo>>> getTodos() async {
  try {
    final dtos = await _remote.fetchTodos();
    _local.putMany(dtos.map((d) => d.toLocal()).toList());
    return Ok(dtos.map((d) => d.toEntity()).toList());
  } on DioException catch (e) {
    final cached = _local.getAll();
    if (cached.isNotEmpty) return Ok(cached.map((l) => l.toEntity()).toList());
    return Err(_mapDioError(e));
  } on Object catch (e) {
    return Err(CacheFailure(e.toString()));
  }
}
```
