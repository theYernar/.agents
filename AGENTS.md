# Project-Scoped Rules

This file contains the consolidated coding style and architectural rules derived from the project's `.agents/skills/feature` instructions and existing codebase. All future feature implementations and code edits should adhere to these guidelines.

## 1. Feature Architecture & Folder Structure
Features are located under `lib/src/feature/<feature_name>`. 
Each feature should stick to a separation of concerns using the following standardized sub-directories (avoid older names like `ui`, `models`, or `cubit` for new features):
- **`bloc/`**: Contains Bloc/Cubit state managers.
- **`data/`**: Contains repositories and remote data sources.
- **`model/`**: Contains DTOs and feature-specific models.
- **`ui/`**: 
  - `pages/` for route-level screens.
  - `widgets/` for extracted reusable feature-specific UI parts.

## 2. Data Layer Style
The data layer relies strictly on interfaces and Dependency Injection.
- **Interfaces**: Always define `abstract interface class I...` for Repositories and Data Sources.
- **Implementations**: Use `final class ... implements I...` with required named parameters in constructors for dependencies (e.g., `IRestClient`, `IAuthRemoteDataSource`).
- **Data Flow**: Transport parsing (from JSON) stays in Remote Data Sources; mapping/orchestration happens in Repositories.
- **Dependency Registration**: App-level repositories must be registered in `lib/src/core/containers/repository_storage.dart` with lazy initialization (`??=`) and proper disposal in `close()`. Use `context.repository.<featureRepository>` to access them from the UI.

## 3. State Management (Bloc / Cubit)
- **Folder**: Always place state managers in the `bloc/` folder, even if using Cubit.
- **States & Events**: Use union states for mutually exclusive flows (e.g., loading, loaded, failure). You can use `freezed` by default, or Dart 3 `sealed class` combined with `Equatable`.
- **Naming & Content**: 
  - Use `failure` for UI error states (e.g., `AppState.failure(message: ...)`).
  - Use domain-specific events (`AppLoggedOut`), not generic ones (`ChangeState`).
- **Async Safety**: When awaiting async tasks inside a Cubit/Bloc, guard later emissions with a check to see if the state manager is closed (`if (isClosed) return;`).

## 4. Presentation & UI Guidelines
- **Routing**: Use `auto_route`. Decorate pages with `@RoutePage()`, register them in `lib/src/feature/app/router/app_router.dart`, and run `build_runner` to generate routes. Never edit `app_router.gr.dart` manually.
- **Localization**: Do not hardcode user-facing strings. Add strings to ARB files (`lib/src/core/constant/localization/translations`) and generate with `flutter gen-l10n`. Access them via `context.localized`.
- **Assets**: Use generated asset classes (`Assets.icons...` from `assets.gen.dart`); do not use string paths.
- **Theming & Components**: Prefer reusable core UI. Use colors/styles from `AppColors`, `AppTextStyles`, spacing like `Gap(AppGaps.sm)`. Re-use widgets from `lib/src/core/presentation/widgets/`.

## 5. Verification & Code Quality
Before completing any Dart/Flutter feature code, always format and lint:
```sh
dart format .
flutter analyze
```
Run generation commands when necessary:
```sh
flutter gen-l10n
dart run build_runner build --delete-conflicting-outputs
```
Do not include unrelated refactors in feature work.

## 6. Engineering Standards

Act as a Senior Flutter Engineer with strong experience in building scalable, production-ready applications.

### Code Principles
Always write code that follows these principles:
- **SOLID**
- **DRY (Don't Repeat Yourself)**
- **KISS (Keep It Simple, Stupid)**
- Prefer composition over inheritance.
- Keep business logic out of UI widgets.
- Write code that is easy to read, maintain, and extend.
- Optimize for long-term maintainability rather than the shortest implementation.

### Widget Guidelines
- **Do not create methods like `_buildWidget()`, `_buildHeader()`, `_buildItem()`, `_buildBody()` or similar just to split a `build()` method.**
- Extract reusable or complex UI into **separate StatelessWidget/StatefulWidget files** located in `ui/widgets/`.
- A widget should have a single responsibility.
- If a widget becomes large or is reused, move it into its own file instead of creating private builder methods.
- Keep the `build()` method declarative and easy to scan.

### Clean Code
- Use meaningful names for variables, methods, classes, and widgets.
- Avoid unnecessary comments. Code should be self-explanatory.
- Minimize nesting by using early returns.
- Avoid duplicated logic.
- Keep methods short and focused on a single responsibility.
- Prefer immutable objects and `final` wherever possible.

### Architecture
- Respect the existing project architecture.
- Never bypass dependency injection.
- Never place networking, storage, or business logic inside UI widgets.
- Reuse existing components before creating new ones.
- Prefer extension methods and reusable utilities over duplicated code.

### Before Finishing
Before considering a task complete, verify that:
- The code is clean and readable.
- There is no duplicated logic.
- SOLID, DRY, and KISS principles are respected.
- No `_buildWidget()`-style helper methods were introduced.
- New widgets are extracted into separate files when appropriate.
- The implementation is consistent with the existing project architecture.

## 7. Error Handling
- Use `errorMessageOf(error)` from `core/utils/error_message_util.dart` to extract user-facing messages from `RestClientException` and other errors.
- In Cubit/Bloc: wrap async calls in `try { ... } on Object catch (error)`.
- Emit `failure(message: errorMessageOf(error))` — never expose raw exception types or stack traces to UI.
- Handle 401 (session expired) at the interceptor level (`DioInterceptor.onUnauthorized`), not in individual cubits.
- Standard async Cubit pattern:
```dart
Future<void> loadData() async {
  emit(const MyState.loading());
  try {
    final result = await _repository.getData();
    if (isClosed) return;
    emit(MyState.loaded(data: result));
  } on Object catch (error) {
    if (isClosed) return;
    emit(MyState.failure(message: errorMessageOf(error)));
  }
}
```

## 8. Core Widget Inventory
Reusable widgets live in `lib/src/core/presentation/widgets/`. Check here before creating feature-specific widgets:
- `bottom_sheet/` — bottom sheet templates
- `buttons/` — CTA, icon, text buttons
- `cards/` — card layouts
- `checkboxes/` — checkbox variants
- `chips/` — chip/tag components
- `dialogs/` — alert, confirmation dialogs
- `forms/` — form containers, validation
- `indicators/` — loading, progress indicators
- `layout/` — scaffold wrappers, spacing helpers
- `media/` — image, video viewers
- `menu/` — popup, dropdown menus
- `textfields/` — input fields, search bars

## 9. Makefile Shortcuts
Use `Makefile` targets instead of raw commands:
- `make clean` — flutter clean + pub get + pod install
- `make gen` — `dart run build_runner build --delete-conflicting-outputs`
- `make format` — `dart format .`
- `make analyze` — `flutter analyze`
- `make l10n` — `flutter gen-l10n`
- `make check` — format + analyze
- `make apk` — release APK build (with .env)
- `make aab` — release AAB build (with .env)

## 10. Extension Methods
Extensions live in `lib/src/core/utils/extensions/`. **Always prefer using an existing extension over writing inline utility logic.** If a useful helper does not yet exist, add it as a new extension method in the appropriate file (or create a new file in the same directory) rather than duplicating logic across features.

### Available Extensions

| File | Extension | Key Methods / Properties |
|------|-----------|--------------------------|
| `context_extension.dart` | `ContextExtension` on `BuildContext` | `context.dependencies`, `context.repository`, `context.localized`, `context.mediaQuery`, `context.screenSize`, `context.viewPadding`, `context.viewInsets`, `context.theme`, `context.textTheme`, `context.platform`, `context.navigator`, `context.focusScope` |
| `string_extension.dart` | `StringExtension` on `String` | `.limit(length)`, `.formatAsPhone()`, `.cleanPhone()`, `.formatAsLicensePlate()`, `.formatAsDate()`, `.formatAsReadableDate()` |
| `string_extension.dart` | `NullableStringExtension` on `String?` | `.formatAsReadableDate()` |
| `integer_extension.dart` | `IntegerExtension` on `num?` | `.thousandFormat()` |
| `duration_extension.dart` | `DurationExtension` on `Duration` | `.delayed()`, `.sleep` |
| `bloc_extension.dart` | `StateNotifierMixin` on `BlocBase` | `notify(state, notifyDelay:, then:)` |

### Rules
- **Never duplicate extension logic inline.** For example, use `context.localized` instead of `Localization.of(context)`, and `myNumber.thousandFormat()` instead of manual `NumberFormat` calls.
- **Check this table first** before writing any utility helper. If the logic fits an existing extension, use it.
- **When adding new helpers**, place them in the matching extension file or create a new `<type>_extension.dart` in the same directory. Follow the existing naming convention (`<Type>Extension on <Type>`).