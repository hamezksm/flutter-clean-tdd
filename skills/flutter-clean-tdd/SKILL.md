---
name: flutter-clean-tdd
description: Build, extend, test and configure Flutter mobile apps using Clean Architecture, strict test-driven development, Riverpod (the only state management and DI), flutter_hooks instead of StatefulWidgets, ObjectBox local storage, Dio networking, go_router typed routes, and freezed/json_serializable generated models. Use this skill for ANY Flutter or Dart app work - creating a project, adding a feature, screen, widget, model, repository, API call, provider, route or test, fixing a Flutter bug, adding a package, or configuring Android/iOS/web/desktop platforms - even if the user doesn't mention architecture, testing or Riverpod.
---

# Flutter Clean Architecture + TDD

This skill encodes one opinionated, non-negotiable way of building Flutter apps. The goal is code that is easy to change: every dependency is swappable, every behaviour is covered by a test written before it, and nothing is hand-written that a generator can produce.

## Non-negotiable rules

1. **Tests first, always.** Write the failing test, run it and see it fail, write the minimum code to pass, then refactor. No production code without a failing test that demanded it. See `references/testing.md`.
2. **Clean Architecture with SOLID.** Domain knows nothing about Flutter, Dio, ObjectBox or JSON. Dependencies point inward. See `references/architecture.md`.
3. **Riverpod is the only state management AND the only DI container.** No get_it, no Provider package, no Bloc, no setState. Use `riverpod_generator` (`@riverpod`).
4. **No StatefulWidget or ConsumerStatefulWidget anywhere.** Use `HookConsumerWidget` (from `hooks_riverpod`) or `ConsumerWidget`. Hooks are only for lifecycle objects and purely visual state. See `references/state-and-widgets.md`.
5. **Dependencies are added only from the command line** with `flutter pub add` / `flutter pub add --dev`, never by editing `pubspec.yaml` version constraints by hand. The CLI resolves the newest version compatible with the SDK and the rest of the graph. If resolution fails, run `flutter pub outdated`, explain the conflict, and propose a fix rather than pinning silently.
6. **Never hand-write serialization, equality, copyWith or routes.** Use `freezed`, `json_serializable`, `objectbox_generator`, `riverpod_generator` and `go_router_builder`, then run `dart run build_runner build -d`. See `references/models.md`.
7. **Networking is Dio. Local storage is ObjectBox. Routing is go_router with typed routes.** Each sits behind an interface so it can be replaced.
8. **Always ask which platforms are targeted before creating or configuring a project**, configure every one of them, and finish with a setup report. See below.

## Workflow

### 1. Starting a new project
Before writing anything, ask the user which platforms they are building for (Android, iOS, web, macOS, Windows, Linux), and whether the UI should be Cupertino everywhere or platform-adaptive. Then:
- `flutter create --org <reverse.domain> --platforms=<list> <app_name>`
- Add the baseline packages with the commands in `references/dependencies.md`.
- If **web** is selected, flag that ObjectBox does not support web and propose a web implementation of the local data source interface (e.g. `hive_ce` or `sembast`) selected at runtime through a Riverpod provider. Ask before proceeding.
- Configure each platform per `references/platform-setup.md`.
- Create the folder structure from `references/architecture.md`.

### 2. Adding a feature (the TDD loop, inside-out)
Work in this order, completing red → green → refactor at every step:
1. **Domain**: entity (freezed) → repository interface → use case. Test the use case with a mocked repository.
2. **Data**: DTO + local model + mappers → remote data source (Dio) → local data source (ObjectBox) → repository implementation. Test mappers, data sources (mocked Dio / in-memory ObjectBox store), and the repository (mocked data sources, including failure paths).
3. **Presentation**: Riverpod notifier → widgets → route. Test the notifier with a `ProviderContainer` and overrides; widget-test the screen for loading, data, empty and error states.
4. Run `dart run build_runner build -d`, `flutter analyze`, `dart format .`, and `flutter test`. Everything must be clean before the feature is called done.

Show the user each failing test before its implementation, briefly. Don't skip the "see it fail" step: a test that never failed proves nothing.

### 3. UI
Before building any screen, read `references/ui.md`. If the `apple-design` skill is installed, read it too for the design principles, but implement them with the Flutter equivalents in `references/ui.md` - that skill's code samples are for the web. UI widgets stay dumb: they render state from a provider and forward user events to a notifier. Motion math (springs, projection, rubber-band) is pure Dart in `core/ui/motion/` and is unit-tested like everything else.

## The setup report

End any task that creates a project, adds a dependency with native setup, or touches platform configuration with a report in this shape. Be specific - exact file paths, console names, menu locations, commands.

```
## Setup report

### Done
- what was configured, per platform, with file paths

### You need to do (external)
For each item: what it is, why it's needed, and step-by-step how to do it
(e.g. API keys, Apple Developer / App Store Connect steps, signing,
Google Play Console, Firebase console, backend URLs, entitlements).

### Verify
Commands or checks that prove it works (e.g. `flutter run -d ios`).

### Open questions / risks
```

## Reference files
Read the ones relevant to the current task:
- `references/architecture.md` - layers, folder structure, SOLID, DI with Riverpod, error handling
- `references/state-and-widgets.md` - Riverpod generator patterns, hooks rules
- `references/models.md` - entity / DTO / ObjectBox model split and mappers
- `references/data-layer.md` - Dio client, data sources, repositories, ObjectBox setup
- `references/routing.md` - go_router + go_router_builder, auth redirects
- `references/testing.md` - TDD loop, mocktail, provider overrides, widget and ObjectBox tests
- `references/dependencies.md` - exact CLI commands for every package
- `references/platform-setup.md` - per-platform configuration checklist
- `references/ui.md` - Cupertino/HIG fallback UI guidance
