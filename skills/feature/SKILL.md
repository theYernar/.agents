---
name: feature
description: Create, update, or review product features in the  Flutter project. Use when the user asks to add a feature, scaffold feature folders, create screens/pages/widgets, add feature data/repository layers, wire Bloc/Cubit state, register feature repositories, add routes, or align a feature with  architecture.
---

# Feature

Use this skill when adding or reshaping a product feature in .

## Feature Structure

Place product code under `lib/src/feature/<feature_name>`.

Use the existing folder names:

```text
lib/src/feature/<feature_name>/
├── bloc/
├── data/
├── model/
└── ui/
    ├── pages/
    └── widgets/
```

Create only the folders the feature actually needs. For UI-only features, `ui/pages` and `ui/widgets` may be enough. For API-backed features, add `data`, `model`, and usually `bloc`.

Avoid the old names `models` and `cubit`; the current project uses `model`, `ui`, and `bloc`.

## Data Layer

For API-backed features, follow the current repository/data-source style:

```dart
abstract interface class IExampleRemoteDS {
  Future<ExampleResponseDTO> getExample();
}

final class ExampleRemoteDSImpl implements IExampleRemoteDS {
  const ExampleRemoteDSImpl({required this.restClient});

  final IRestClient restClient;

  @override
  Future<ExampleResponseDTO> getExample() async {
    final response = await restClient.get('/api/example');
    final json = response as Map<String, dynamic>;
    final data = json['data'] as Map<String, dynamic>? ?? json;
    return ExampleResponseDTO.fromJson(data);
  }
}
```

```dart
abstract interface class IExampleRepository {
  Future<ExampleResponseDTO> getExample();
}

final class ExampleRepositoryImpl implements IExampleRepository {
  const ExampleRepositoryImpl({required this.remoteDS});

  final IExampleRemoteDS remoteDS;

  @override
  Future<ExampleResponseDTO> getExample() {
    return remoteDS.getExample();
  }
}
```

Use `package:/...` imports. Keep transport parsing in remote data sources, orchestration/mapping in repositories, and screen state in Cubit/Bloc.

For DTO rules, use the `dto-instruction` skill.

## Dependency Registration

Register app-level repositories and data sources in `lib/src/core/containers/repository_storage.dart`:

- Add imports for the feature data files.
- Add getters to `IRepositoryStorage`.
- Add nullable cached fields to `RepositoryStorage`.
- Initialize getters lazily with `??=`.
- Reset cached fields in `close()`.

Use repositories from UI with `context.repository.<featureRepository>` through `ContextExtension`. Do not instantiate repositories directly in widgets.

Use `DependenciesContainer` and `CompositionRoot` only for dependencies that are built once at app startup and are not simply feature repositories.

## Bloc And Cubit

Put feature state managers in `bloc/`, even when the class is a Cubit.

Follow `AGENTS.md`:

- Use `freezed` for new Bloc/Cubit state and event by default.
- Use union states for mutually exclusive loading/loaded/empty/failure flows.
- Use one data-state for forms, wizards, and screens with saved input fields.
- Prefer `failure` for UI error state; reserve `error` for exceptions/logging/low-level errors.
- Use domain-specific events and methods, not generic change-state events.

When a Cubit awaits async work, guard late emits with `if (isClosed) return;` before emitting after an `await`.

## Presentation

Put route-level screens in `ui/pages` and extracted screen pieces in `ui/widgets`.

Reuse core UI before creating new widgets:

- Theme/resources from `lib/src/core/theme`.
- Shared widgets from `lib/src/core/presentation/widgets`.
- `context.localized`, `context.repository`, and other helpers from `context_extension.dart`.

Do not hardcode user-facing strings in widgets when they belong in localization. Add strings to ARB files under `lib/src/core/constant/localization/translations` and run `flutter gen-l10n`.

Use generated assets from `lib/src/core/constant/generated`; do not reference asset paths by hand when a generated accessor exists.

## Routing

For a new routable page:

- Add `@RoutePage()` to the page widget.
- Import the page in `lib/src/feature/app/router/app_router.dart`.
- Add the `AutoRoute` entry in the correct place.
- Add auth guards when the screen requires authentication.
- Run `dart run build_runner build --delete-conflicting-outputs`.

Do not edit `app_router.gr.dart` manually.

## Verification

After changing Dart/Flutter feature code, run:

```sh
dart format .
flutter analyze
```

Run `flutter test` when behavior, widgets, routing, Bloc/Cubit, models, repositories, or parsing are affected.

Run generation when relevant:

```sh
flutter gen-l10n
dart run build_runner build --delete-conflicting-outputs
```

Inspect generated diffs before finishing. Do not include unrelated refactors in feature work.
