---
name: dto-instruction
description: Create, update, or review backend response DTO models in the  Flutter project. Use when the user asks to add a DTO, update a DTO, parse backend JSON, model an API response, work with freezed/json_serializable DTOs, or fix generated DTO code.
---

# DTO Instruction

Use this skill when working with backend response models in .

## Project Contract

- Put feature-specific DTOs in `lib/src/feature/<feature_name>/model`.
- Name files `*_dto.dart` and classes `*DTO`.
- Use `freezed` with `json_serializable`.
- Keep backend response fields nullable by default. Use non-null only for values that are locally guaranteed.
- Do not edit generated `*.freezed.dart` or `*.g.dart` files by hand.

## File Pattern

Use this pattern for new DTO files:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_dto.freezed.dart';
part 'user_dto.g.dart';

@freezed
sealed class UserDTO with _$UserDTO {
  const factory UserDTO({
    int? id,
    String? name,
    String? phone,
    String? avatarPath,
    String? createdAt,
  }) = _UserDTO;

  factory UserDTO.fromJson(Map<String, dynamic> json) => _$UserDTOFromJson(json);
}
```

For new Dart 3 / Freezed 3 DTOs, prefer `sealed class`, matching the newer auth DTOs. When editing an older DTO file that already uses `abstract class`, keep that file's existing style unless the user asks for a style migration.

## JSON Keys

The root `build.yaml` configures `json_serializable` with:

```yaml
targets:
  $default:
    builders:
      json_serializable:
        options:
          field_rename: snake
          create_factory: true
          create_to_json: true
          include_if_null: false
```
Because of this, do not add `@JsonKey(name: '...')` for ordinary camelCase-to-snake_case fields:

- `avatarPath` maps to `avatar_path`.
- `createdAt` maps to `created_at`.
- `accessToken` maps to `access_token`.

Use `@JsonKey` only when the backend key does not match the automatic snake_case mapping, when a Dart name must differ from the backend name, or when a field needs custom behavior such as a converter, default value, or explicit include/exclude rule.

Do not add `includeIfNull: false` per field; the global generator option already handles it.

## Multiple DTOs

It is acceptable to keep small, tightly coupled nested DTOs in one file when they represent one API response, for example `RegistrationDTO`, `RegistrationUserDTO`, and `RegistrationTokenDTO`.

Use a response wrapper DTO when the backend wraps collections or metadata:

```dart
@freezed
sealed class CarMakesResponseDTO with _$CarMakesResponseDTO {
  const factory CarMakesResponseDTO({
    List<CarMakeDTO>? carMakes,
  }) = _CarMakesResponseDTO;

  factory CarMakesResponseDTO.fromJson(Map<String, dynamic> json) =>
      _$CarMakesResponseDTOFromJson(json);
}
```

## Behavior Boundary

DTOs should describe transport data. Keep UI formatting, navigation decisions, and business flow logic out of DTO classes. Put mapping or interpretation in repositories, cubits/blocs, or domain/value objects when needed.

## Generation And Verification

After creating or changing DTO source files, run:

```sh
dart run build_runner build --delete-conflicting-outputs
```

Do not add `--force-jit` unless the user reports a local toolchain issue that specifically requires it.

Then run the normal Dart/Flutter checks for the touched behavior:

```sh
dart format .
flutter analyze
flutter test
```

Run `flutter test` especially when parsing behavior, repository mapping, Bloc/Cubit flow, or API tests are affected.
