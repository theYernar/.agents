---
name: error-handling
description: Handle errors, exceptions, network failures, and user-facing error messages in the Flutter project. Use when the user asks to add error handling, fix error display, handle API errors, show snackbars/dialogs for failures, standardize try/catch patterns, or work with RestClientException.
---

# Error Handling

Use this skill when adding, fixing, or reviewing error handling in the project.

## Project Contract

Follow the rules from `AGENTS.md` section 7 (Error Handling). This skill provides additional detail.

## Error Extraction

Use `errorMessageOf(error)` from `lib/src/core/utils/error_message_util.dart` to convert any `Object` error into a user-facing string.

The function handles:
- `RestClientException` — extracts `.message` or `.cause['message']`.
- Any other `Object` — falls back to `.toString()`.

Extend `errorMessageOf` when a new exception type needs a custom user-facing message. Do not create per-feature error extractors.

## Cubit / Bloc Pattern

Every async method in a Cubit or Bloc must follow this structure:

```dart
Future<void> doSomething() async {
  emit(const MyState.loading());
  try {
    final result = await _repository.doSomething();
    if (isClosed) return;
    emit(MyState.loaded(data: result));
  } on Object catch (error) {
    if (isClosed) return;
    emit(MyState.failure(message: errorMessageOf(error)));
  }
}
```

Rules:
- Always emit `loading` before the async call.
- Always guard emissions after `await` with `if (isClosed) return;`.
- Always catch `Object`, not `Exception` — Dart can throw non-Exception types.
- Always use `errorMessageOf(error)` — never expose raw types or stack traces.
- Use `failure` for the state name, not `error`.

## Network Error Levels

Errors are handled at three levels:

1. **Interceptor level** (`DioInterceptor`):
   - 401 → triggers `onUnauthorized` callback → `AuthRepository.notifySessionExpired()`.
   - Do not handle 401 in individual cubits.

2. **Repository level**:
   - Rethrow transport exceptions as-is. Do not swallow errors silently.
   - Add domain-specific mapping only when the backend error format differs from the standard.

3. **Cubit/Bloc level**:
   - Catch errors and emit `failure` state with `errorMessageOf`.
   - Do not retry automatically unless the feature specifically requires it.

## UI Error Display

When showing errors to the user:
- **Inline error** — use `failure` state in the widget tree to show error text in place of content.
- **SnackBar** — use for transient errors on actions (submit, delete, refresh).
- **Dialog** — use for blocking errors that require user acknowledgment.

Prefer inline error states over snackbars for page-level data loading failures.

## Do Not

- Do not create per-feature error utility functions.
- Do not catch errors silently (`catch (_) {}`).
- Do not show raw exception messages like `DioException` or `SocketException` to users.
- Do not handle session expiry (401) outside the interceptor.
