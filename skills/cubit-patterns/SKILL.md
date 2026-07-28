---
name: cubit-patterns
description: Create, update, or review Cubit/Bloc state managers in the Flutter project. Use when the user asks to add a cubit, create a bloc, fix state management, add loading/error states, wire cubit to UI, or review cubit patterns. Covers all cubit variations found in the project.
---

# Cubit Patterns

Use this skill when creating or modifying Cubit/Bloc state managers. All patterns below are derived from the actual project codebase.

## File Location

Always place cubits in `lib/src/feature/<feature>/bloc/`, even when using Cubit (not Bloc).

File naming: `<name>_cubit.dart`. State goes in the same file unless it is complex enough to warrant a separate `<name>_state.dart`.

## Pattern 1 — Simple Data Fetch

The most common pattern. Fetches data from a repository, handles loading and errors.

Used by: `BonusCubit`, `MyInvitationsCubit`, `HomeFeedCubit`, `HomeStoriesCubit`, `BranchesCubit`, `CitiesCubit`.

```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:gt_oil/src/core/utils/error_message_util.dart';
import 'package:gt_oil/src/feature/example/data/example_repository.dart';
import 'package:gt_oil/src/feature/example/model/example_dto.dart';

part 'example_cubit.freezed.dart';

class ExampleCubit extends Cubit<ExampleState> {
  ExampleCubit({required this._repository}) : super(const ExampleState.initial());

  final IExampleRepository _repository;

  Future<void> loadData() async {
    if (state is _Loading) return;
    emit(const ExampleState.loading());

    try {
      final result = await _repository.getData();
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
  const factory ExampleState.success({required ExampleDTO data}) = _Success;
  const factory ExampleState.failure({required String message}) = _Failure;
}
```

Key rules:
- Guard against duplicate loads: `if (state is _Loading) return;`
- Guard after await: `if (isClosed) return;`
- Use `errorMessageOf(error)` for user-facing messages.
- Repository field is `final` and **private** (`_repository`).
- Constructor uses `required this._repository`.

## Pattern 2 — Data Fetch with Persistent State

When the UI needs to keep showing the previous data during refresh or error.

Used by: `ProfileCubit`.

```dart
class ProfileCubit extends Cubit<ProfileState> {
  ProfileCubit({required this._repository}) : super(const ProfileState.initial());

  final IProfileRepository _repository;

  Future<void> loadProfile() async {
    if (state is _Loading) return;
    final currentUser = state.user;
    emit(ProfileState.loading(user: currentUser));

    try {
      final user = await _repository.getProfile();
      if (isClosed) return;
      emit(ProfileState.success(user: user));
    } on Object catch (error) {
      if (isClosed) return;
      emit(ProfileState.failure(message: errorMessageOf(error), user: currentUser));
    }
  }
}

@freezed
sealed class ProfileState with _$ProfileState {
  const factory ProfileState.initial() = _Initial;
  const factory ProfileState.loading({UserDTO? user}) = _Loading;
  const factory ProfileState.success({required UserDTO user}) = _Success;
  const factory ProfileState.failure({required String message, UserDTO? user}) = _Failure;
}

extension ProfileStateX on ProfileState {
  UserDTO? get user => when(
    initial: () => null,
    loading: (user) => user,
    success: (user) => user,
    failure: (_, user) => user,
  );
}
```

Key differences:
- `loading` and `failure` states carry optional previous data.
- Extension getter `user` extracts data from any state.
- Previous data is preserved across state transitions.

## Pattern 3 — Action with Callbacks

When the Cubit performs a mutation (POST/PUT/DELETE) and the UI needs success/error feedback without changing the list state.

Used by: `BonusCubit.activateReferralCode`.

```dart
Future<void> performAction({
  required String param,
  required VoidCallback onSuccess,
  required void Function(String) onError,
}) async {
  try {
    await _repository.performAction(param);
    if (isClosed) return;
    onSuccess();
    loadData(); // Refresh the list after success
  } on Object catch (error) {
    if (isClosed) return;
    onError(errorMessageOf(error));
  }
}
```

Key rules:
- No state emission during the action — use callbacks.
- Reload the data after successful mutation.
- Guard with `if (isClosed) return;` before callbacks.

## Pattern 4 — Single Data State (non-union)

When the state is a mutable data object with multiple fields, not mutually exclusive states.

Used by: `StoryViewCubit`.

```dart
@freezed
sealed class StoryViewState with _$StoryViewState {
  const factory StoryViewState({
    @Default(0) int currentIndex,
    @Default([]) List<double> progress,
    @Default(false) bool isCompleted,
  }) = _StoryViewState;
}
```

Use this when:
- The widget manages continuous UI state (progress, form fields, animation).
- All fields coexist — there are no mutually exclusive states.
- `copyWith` is the primary state update mechanism.

## Pattern 5 — Local Data State (no API)

When the Cubit works with local data only (search, filtering, in-memory state).

Used by: `MapSearchCubit`.

```dart
class MapSearchCubit extends Cubit<MapSearchState> {
  MapSearchCubit({required this.repository, required this.allBranches})
    : super(const MapSearchState.initial());

  final IMapRepository repository;
  final List<BranchDTO> allBranches;

  void init() {
    if (isClosed) return;
    final historyIds = repository.getSearchHistory();
    final recentBranches = historyIds
        .map((id) => allBranches.where((b) => b.id?.toString() == id).firstOrNull)
        .whereType<BranchDTO>()
        .toList();

    emit(MapSearchState.loaded(
      query: '',
      recentBranches: recentBranches,
      searchResults: [],
    ));
  }

  void search(String query) {
    if (state is! _Loaded) return;
    final currentState = state as _Loaded;
    // ... filtering logic ...
    emit(currentState.copyWith(query: query, searchResults: results));
  }
}
```

Key rules:
- Synchronous methods for local operations.
- Cast state to access `copyWith` on the loaded variant.
- Guard with type check: `if (state is! _Loaded) return;`

## Pattern 6 — Full Bloc (Events + States)

For complex flows with multiple event sources (streams, lifecycle, external triggers).

Used by: `AppBloc`, `AppRestartBloc`.

```dart
final class ExampleBloc extends Bloc<ExampleEvent, ExampleState> {
  ExampleBloc({required this._repository}) : super(const ExampleState.loading()) {
    on<ExampleStarted>(_onStarted);
    on<ExampleRefreshed>(_onRefreshed);
  }

  final IExampleRepository _repository;

  Future<void> _onStarted(ExampleStarted event, Emitter<ExampleState> emit) async {
    try {
      final data = await _repository.getData();
      emit(ExampleState.loaded(data: data));
    } on Object catch (error) {
      emit(ExampleState.failure(message: errorMessageOf(error)));
    }
  }
}
```

Use Bloc when:
- Multiple events map to different state transitions.
- Stream subscriptions trigger events (e.g., `sessionExpired`).
- You need `on<Event>` handler registration.

For simple request-response flows, **prefer Cubit** over Bloc.

## State Declaration

Use `@freezed sealed class` with union factories:

```dart
@freezed
sealed class ExampleState with _$ExampleState {
  const factory ExampleState.initial() = _Initial;
  const factory ExampleState.loading() = _Loading;
  const factory ExampleState.success({required ExampleDTO data}) = _Success;
  const factory ExampleState.failure({required String message}) = _Failure;
}
```

Rules:
- Use `sealed class` (Dart 3 / Freezed 3 style).
- Standard state names: `initial`, `loading`, `success`, `failure`.
- `failure` always has a `required String message`.
- `success` uses named parameters: `{required ExampleDTO data}`.
- Private generated classes: `_Initial`, `_Loading`, `_Success`, `_Failure`.

Access repository via `context.repository.<name>` from `ContextExtension`.

## Timer / Lifecycle Cubits

When the Cubit manages timers or subscriptions:

```dart
@override
Future<void> close() {
  _timer?.cancel();
  return super.close();
}
```

Always cancel timers/subscriptions in `close()`.

## Do Not

- Do not use `catch (e)` without type — use `on Object catch (error)`.
- Do not hardcode error messages in Russian — use `errorMessageOf(error)`.
- Do not emit after `await` without `if (isClosed) return;`.
- Do not put business logic in widgets — keep it in the Cubit.
- Do not create `_build*()` methods in the UI for state rendering — use `switch` or `.when()`.
- Do not use `Equatable` for new Cubit states — use `freezed`.
- Do not place cubit files in `cubit/` folder — use `bloc/` folder.
