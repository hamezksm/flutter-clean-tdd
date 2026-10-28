# Models

Each concept has up to three representations. Keep them separate - this is what lets Dio, the API shape, or ObjectBox change without touching domain code.

| Kind | Layer | File | Tooling | Mutable? |
|---|---|---|---|---|
| Entity | domain | `entities/todo.dart` | freezed | No |
| DTO | data | `models/todo_dto.dart` | freezed + json_serializable | No |
| Local model | data | `models/todo_local.dart` | objectbox `@Entity` | Yes (ObjectBox needs it) |

Never hand-write `fromJson`, `toJson`, `==`, `hashCode` or `copyWith`.

## Entity (domain)
```dart
import 'package:freezed_annotation/freezed_annotation.dart';
part 'todo.freezed.dart';

@freezed
abstract class Todo with _$Todo {
  const factory Todo({
    required String id,
    required String title,
    required bool done,
    DateTime? dueAt,
  }) = _Todo;
}
```
No JSON here - the domain doesn't know JSON exists. Domain unions use `@freezed sealed class` so `switch` is exhaustive.

## DTO (remote JSON)
```dart
import 'package:freezed_annotation/freezed_annotation.dart';
part 'todo_dto.freezed.dart';
part 'todo_dto.g.dart';

@freezed
abstract class TodoDto with _$TodoDto {
  const factory TodoDto({
    required String id,
    required String title,
    @Default(false) bool done,
    @JsonKey(name: 'due_at') DateTime? dueAt,
  }) = _TodoDto;

  factory TodoDto.fromJson(Map<String, dynamic> json) => _$TodoDtoFromJson(json);
}
```
Project-wide JSON conventions go in `build.yaml`, not repeated per field:
```yaml
targets:
  $default:
    builders:
      json_serializable:
        options:
          field_rename: snake
          explicit_to_json: true
          include_if_null: false
```
Custom types (enums with fallbacks, epoch timestamps) get a `JsonConverter` class, applied with an annotation - still no manual parsing in models.

## Local model (ObjectBox)
```dart
import 'package:objectbox/objectbox.dart';

@Entity()
class TodoLocal {
  TodoLocal({this.obxId = 0, required this.id, required this.title, required this.done, this.dueAt});

  @Id()
  int obxId;               // ObjectBox's internal id; 0 = new

  @Unique(onConflict: ConflictStrategy.replace)
  @Index()
  String id;               // the domain/server id

  String title;
  bool done;

  @Property(type: PropertyType.date)
  DateTime? dueAt;
}
```
After changing any `@Entity`, run build_runner and **commit `lib/objectbox-model.json`** - ObjectBox uses it for schema migrations. Renames need `@Property(uid: ...)`; follow the ObjectBox docs, never delete the model file.

## Mappers
Plain extension methods in `data/mappers/`, unit-tested in both directions:
```dart
extension TodoDtoX on TodoDto {
  Todo toEntity() => Todo(id: id, title: title, done: done, dueAt: dueAt);
  TodoLocal toLocal() => TodoLocal(id: id, title: title, done: done, dueAt: dueAt);
}
extension TodoLocalX on TodoLocal {
  Todo toEntity() => Todo(id: id, title: title, done: done, dueAt: dueAt);
}
extension TodoX on Todo {
  TodoDto toDto() => TodoDto(id: id, title: title, done: done, dueAt: dueAt);
}
```
Write the mapper test first (round-trip + edge cases like nulls).
