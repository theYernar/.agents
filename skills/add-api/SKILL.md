---
name: add-api
description: Add a new API endpoint to the Flutter project end-to-end. Use when the user asks to connect a backend endpoint, add an API call, integrate a new REST endpoint, or wire backend data from DTO to UI. Enforces the strict order DTO → Remote DS → Repository → Cubit → UI.
---

# Add API

Use this skill when wiring a new backend endpoint into the app. Follow the steps strictly **in order**: DTO → Remote Data Source → Repository → register in DI → Cubit → UI.

## Step 1 — DTO (`model/`)

Create the response model in `lib/src/feature/<feature>/model/<name>_dto.dart`.

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part '<name>_dto.freezed.dart';
part '<name>_dto.g.dart';

@freezed
sealed class ExampleDTO with _$ExampleDTO {
  const factory ExampleDTO({
    int? id,
    String? title,
    String? description,
    String? createdAt,
  }) = _ExampleDTO;

  factory ExampleDTO.fromJson(Map<String, dynamic> json) =>
      _$ExampleDTOFromJson(json);
}
```

Rules:
- All backend fields are **nullable** by default.
- No `@JsonKey(name:)` for standard camelCase→snake_case (handled by `build.yaml`).
- Use `@JsonKey` only for non-standard mappings or custom converters.
- Nested DTOs can live in the same file if they represent one API response.
- Name the file `*_dto.dart` and the class `*DTO`.

For response wrappers with lists:
```dart
@freezed
sealed class ExampleListResponseDTO with _$ExampleListResponseDTO {
  const factory ExampleListResponseDTO({
    List<ExampleDTO>? items,
    int? total,
  }) = _ExampleListResponseDTO;

  factory ExampleListResponseDTO.fromJson(Map<String, dynamic> json) =>
      _$ExampleListResponseDTOFromJson(json);
}
```

Run `make gen` after creating/modifying DTOs.

## Step 2 — Remote Data Source (`data/`)

Add the method to the existing remote DS interface and implementation, or create a new file if this is a new feature.

**Interface** — add the method signature:

```dart
abstract interface class IExampleRemoteDS {
  Future<ExampleDTO> getExample();
  Future<List<ExampleDTO>> getExamples();
  Future<void> createExample(Map<String, dynamic> body);
  Future<ExampleDTO> updateExample(int id, Map<String, dynamic> body);
  Future<void> deleteExample(int id);
}
```

**Implementation** — add the REST call:

```dart
final class ExampleRemoteDSImpl implements IExampleRemoteDS {
  const ExampleRemoteDSImpl({required this.restClient});

  final IRestClient restClient;

  @override
  Future<ExampleDTO> getExample() async {
    final response = await restClient.get('/api/examples/1');
    return ExampleDTO.fromJson(response);
  }

  @override
  Future<List<ExampleDTO>> getExamples() async {
    final response = await restClient.get('/api/examples');
    final list = response['data'] as List<dynamic>? ?? [];
    return list
        .cast<Map<String, dynamic>>()
        .map(ExampleDTO.fromJson)
        .toList();
  }

  @override
  Future<void> createExample(Map<String, dynamic> body) async {
    await restClient.post('/api/examples', body: body);
  }

  @override
  Future<ExampleDTO> updateExample(int id, Map<String, dynamic> body) async {
    final response = await restClient.put('/api/examples/$id', body: body);
    return ExampleDTO.fromJson(response);
  }

  @override
  Future<void> deleteExample(int id) async {
    await restClient.delete('/api/examples/$id');
  }
}
```

Rules:
- Constructor uses `required this.restClient` of type `IRestClient`.
- All JSON parsing stays here — do not parse in Repository.
- For list responses, handle the `data` wrapper if the backend uses one.
- Use `const` constructor when possible.

## Step 3 — Repository (`data/`)

Add the method to the repository interface and implementation.

```dart
abstract interface class IExampleRepository {
  Future<ExampleDTO> getExample();
  Future<List<ExampleDTO>> getExamples();
  Future<void> createExample(Map<String, dynamic> body);
}

final class ExampleRepositoryImpl implements IExampleRepository {
  const ExampleRepositoryImpl({required this.remoteDS});

  final IExampleRemoteDS remoteDS;

  @override
  Future<ExampleDTO> getExample() => remoteDS.getExample();

  @override
  Future<List<ExampleDTO>> getExamples() => remoteDS.getExamples();

  @override
  Future<void> createExample(Map<String, dynamic> body) =>
      remoteDS.createExample(body);
}
```

Rules:
- Constructor uses `required this.remoteDS` of the interface type.
- Repository delegates to Remote DS. Add orchestration only when needed (caching, combining calls, mapping).
- Do not parse JSON here — that belongs in the Remote DS.
- Use `const` constructor.

## Step 4 — Register in DI (`repository_storage.dart`)

If this is a **new feature**, register both the Remote DS and Repository in `lib/src/core/containers/repository_storage.dart`:

1. Add imports.
2. Add getters to `IRepositoryStorage`:
```dart
IExampleRepository get exampleRepository;
IExampleRemoteDS get exampleRemoteDataSource;
```
3. Add nullable cached fields to `RepositoryStorage`:
```dart
IExampleRepository? _exampleRepository;
IExampleRemoteDS? _exampleRemoteDataSource;
```
4. Add lazy initializers:
```dart
@override
IExampleRepository get exampleRepository =>
    _exampleRepository ??= ExampleRepositoryImpl(remoteDS: exampleRemoteDataSource);

@override
IExampleRemoteDS get exampleRemoteDataSource =>
    _exampleRemoteDataSource ??= ExampleRemoteDSImpl(restClient: restClient);
```
5. Add cleanup in `close()`:
```dart
_exampleRepository = null;
_exampleRemoteDataSource = null;
```

If **adding a method to an existing feature** (e.g., adding `getExamples()` to the already-registered `BonusRepository`), skip this step — DI is already wired.

## Step 5 — Cubit (`bloc/`)

Create or update the Cubit in `lib/src/feature/<feature>/bloc/`.

```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:gt_oil/src/core/utils/error_message_util.dart';

part '<name>_cubit.freezed.dart';

class ExampleCubit extends Cubit<ExampleState> {
  ExampleCubit({required this._repository}) : super(const ExampleState.initial());

  final IExampleRepository _repository;

  Future<void> loadExamples() async {
    if (state is _Loading) return;
    emit(const ExampleState.loading());

    try {
      final result = await _repository.getExamples();
      if (isClosed) return;
      emit(ExampleState.success(data: result));
    } on Object catch (error) {
      if (isClosed) return;
      emit(ExampleState.failure(message: errorMessageOf(error)));
    }
  }
}

@freezed
sealed class ExampleState with _$ExampleState {
  const factory ExampleState.initial() = _Initial;
  const factory ExampleState.loading() = _Loading;
  const factory ExampleState.success({required List<ExampleDTO> data}) = _Success;
  const factory ExampleState.failure({required String message}) = _Failure;
}
```

Run `make gen` after creating/modifying the cubit.

## Step 6 — UI integration

**Provide the Cubit.** Choose where to provide based on scope:

- **Global** (needed across multiple screens) → add to `AppBlocProviders` in `lib/src/feature/app/presentation/widgets/app_bloc_providers.dart`:
```dart
BlocProvider(
  create: (context) => ExampleCubit(
    repository: context.repository.exampleRepository,
  ),
),
```

- **Page-scoped** (needed on one screen only) → use `AutoRouteWrapper` in the page widget:
```dart
@RoutePage()
class ExamplePage extends StatelessWidget implements AutoRouteWrapper {
  const ExamplePage({super.key});

  @override
  Widget wrappedRoute(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider(
          create: (context) =>
              ExampleCubit(repository: context.repository.exampleRepository)
                ..loadExamples(),
        ),
      ],
      child: this,
    );
  }

  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: ExamplePageBody(),
    );
  }
}
```

Access repository via `context.repository.<name>` from `ContextExtension`.

**Consume the state** using `BlocBuilder` with `state.maybeWhen()`:
```dart
BlocBuilder<ExampleCubit, ExampleState>(
  builder: (context, state) {
    return state.maybeWhen(
      loading: () => const Center(child: CupertinoActivityIndicator()),
      success: (data) => ExampleListView(items: data),
      failure: (message) => Center(
        child: Text(
          message,
          textAlign: TextAlign.center,
        ),
      ),
      orElse: () => const SizedBox.shrink(),
    );
  },
),
```

Rules:
- Use `state.maybeWhen()` with `orElse` — not `switch` on private generated classes.
- Use `CupertinoActivityIndicator()` for loading, not `CircularProgressIndicator`.
- Use `AppTextStyles` and `AppColors` for error text styling.
- Return `SizedBox.shrink()` for `orElse` to hide unhandled states.
- Trigger data loading in `wrappedRoute` via `..loadExamples()` cascade.

## Checklist

After wiring the full pipeline:

- [ ] DTO created with nullable fields, `part` directives correct
- [ ] Remote DS parses JSON, uses `IRestClient`
- [ ] Repository delegates to Remote DS
- [ ] `repository_storage.dart` updated (if new feature)
- [ ] Cubit uses `errorMessageOf`, guards with `if (isClosed) return`
- [ ] Cubit provided via `wrappedRoute` (page-scoped) or `AppBlocProviders` (global)
- [ ] UI uses `state.maybeWhen()` with `orElse`, handles loading/success/failure
- [ ] Run `make gen` → `make check`
