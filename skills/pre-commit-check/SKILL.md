---
name: pre-commit-check
description: Check  changes before committing or staging. Use when the user asks to prepare a commit, check before commit, run pre-commit checks, verify current changes, commit safely, or when Gemini is about to create a git commit in this Flutter project.
---

# Pre Commit Check

Act as the final gate before a  commit. Verify the working tree, generated files, formatting, analyzer, and relevant tests before saying changes are ready.

Do not create a commit unless the user explicitly asks for it. Do not revert or overwrite unrelated user changes.

## Workflow

1. Inspect the working tree:
   - Run `git status --short`.
   - Inspect unstaged and staged changes with `git diff --name-only` and `git diff --cached --name-only`.
   - Read the relevant diffs before deciding which checks apply.
2. Classify changed files:
   - Dart/Flutter app code.
   - Tests.
   - ARB localization files.
   - Assets or generated asset references.
   - Routes, `freezed`, `json_serializable`, DTOs, or other build-runner inputs.
   - Docs, SVG, markdown, or config-only changes.
3. Protect the user's work:
   - Note unrelated modified files instead of reverting them.
   - If committing only part of the tree, verify staged vs unstaged content separately.
   - If generated commands or formatters rewrite files, inspect the resulting diff before reporting readiness.

## Required Checks

For Dart/Flutter code changes:

- Run `dart format .` when preparing changes for commit.
- Run `flutter analyze`.
- Run `flutter test` when behavior, widgets, routing, Bloc/Cubit, models, localization wiring, or reusable UI are affected.

For ARB changes:

- Run `flutter gen-l10n`.
- Verify generated localization files changed consistently or no generated diff was needed.

For `auto_route`, `freezed`, `json_serializable`, FlutterGen, DTO, router, or asset declaration changes:

- Run `dart run build_runner build --delete-conflicting-outputs` when generation can be affected.
- Verify generated files are either updated or intentionally unchanged.

For docs, markdown, SVG, or comment-only changes:

- Diff inspection is usually enough.
- Do not run heavy Flutter checks unless the change can affect build behavior.

Always run `git diff --check` before declaring the tree commit-ready.

## Makefile

The project has a `Makefile` with shortcut targets. Prefer these over raw commands:

- `make format` — `dart format .`
- `make analyze` — `flutter analyze`
- `make gen` — `dart run build_runner build --delete-conflicting-outputs`
- `make l10n` — `flutter gen-l10n`
- `make check` — format + analyze
- `make clean` — flutter clean + pub get + pod install
- `make apk` — release APK build
- `make aab` — release AAB build

## Failure Handling

If a command fails, stop the ready-to-commit claim and report:

- The command.
- The first blocking error.
- Whether the failure appears related to the current diff.
- The next concrete fix or investigation step.

If sandboxing or missing dependencies block a command, request the needed approval or say exactly what could not be verified.

## Final Report

Use a compact gate report:

- `Status`: `ready`, `not ready`, or `partially verified`.
- `Blockers`: concrete failures with file/line references when available.
- `Checks`: commands run and outcomes.
- `Diff notes`: generated/formatting changes, unrelated files, or staged-vs-unstaged caveats.

If the user asked to commit and all checks pass, stage only the intended files, commit with an appropriate message, and report the commit hash.
