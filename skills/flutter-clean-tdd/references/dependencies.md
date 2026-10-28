# Dependencies (CLI only)

Never edit version constraints in `pubspec.yaml` by hand. The CLI picks the newest version compatible with the SDK and everything else already in the graph.

## Baseline (new project)
```bash
flutter pub add hooks_riverpod flutter_hooks riverpod_annotation \
  freezed_annotation json_annotation dio go_router \
  objectbox objectbox_flutter_libs:any path_provider path

flutter pub add --dev build_runner riverpod_generator freezed json_serializable \
  go_router_builder objectbox_generator:any mocktail
```
- `hooks_riverpod` replaces `flutter_riverpod` - don't add both.
- `objectbox_flutter_libs:any` / `objectbox_generator:any` follow ObjectBox's own install instructions so they match the `objectbox` version.
- Lints: add `riverpod_lint` and set it up exactly as its current pub.dev README says (the analyzer-plugin setup has changed across versions).

## Add only when needed
| Need | Command |
|---|---|
| Secure token storage | `flutter pub add flutter_secure_storage` |
| Dio HTTP mocking in tests | `flutter pub add --dev http_mock_adapter` |
| Web local storage (ObjectBox has no web) | `flutter pub add hive_ce hive_ce_flutter` + `flutter pub add --dev hive_ce_generator` |

## When resolution fails
1. Run `flutter pub outdated` and read the conflict.
2. Explain which packages conflict and why.
3. Propose options (upgrade the other package, wait, alternative package). Only pin with `pkg:^x.y.z` if the user agrees, and note it in the report.

## After adding/upgrading
`dart run build_runner build -d`, then `flutter analyze` and `flutter test`. Mention any native setup the package needs in the setup report.
