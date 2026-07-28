---
name: refactor
description: Safely refactor, rename, extract, or migrate code in the Flutter project. Use when the user asks to rename a feature, extract a widget to core, merge duplicate components, move files, migrate patterns, clean up imports, or restructure code without changing behavior.
---

# Refactor

Use this skill when restructuring code without changing external behavior.

## Before Starting

1. **Identify scope**: List all files and symbols affected by the refactor.
2. **Find all usages**: Use `grep` / `rg` to find every import, reference, and usage of the target symbol or file.
3. **Check generated files**: Determine if the refactor touches `freezed`, `json_serializable`, `auto_route`, `flutter_gen`, or localization inputs.
4. **Plan the order**: Dependencies first, dependents second. Update leaf files before files that import them.

## Rename Feature

When renaming a feature folder (`lib/src/feature/<old>/` → `lib/src/feature/<new>/`):

1. Rename the directory.
2. Update all `import 'package:gt_oil/src/feature/<old>/...'` across the project.
3. Update `repository_storage.dart` — interface getters, cached fields, lazy initializers, and `close()`.
4. Update `app_router.dart` — import paths and route entries.
5. Update ARB keys if the feature has localization strings with the old name.
6. Regenerate:
   ```sh
   make gen
   make l10n
   ```
7. Verify:
   ```sh
   make check
   ```

## Extract Widget to Core

When moving a feature-specific widget to `lib/src/core/presentation/widgets/`:

1. Create the target directory under the appropriate category (e.g., `buttons/`, `cards/`, `dialogs/`).
2. Move the widget file.
3. Remove feature-specific dependencies — the widget must not import from any `feature/` directory.
4. Replace hardcoded values with theme tokens (`AppColors`, `AppTextStyles`, `AppGaps`, `AppRadius`).
5. Update all imports from the old path to the new core path.
6. Verify no circular dependencies.

## Merge Duplicate Components

When consolidating duplicate widgets or utilities:

1. Identify all variants with `rg` — search for class names, similar widget structures.
2. Pick the most complete variant as the base, or create a new unified version.
3. Add parameters for the behavioral differences between variants.
4. Replace all usages one by one, verifying each replacement.
5. Delete the old duplicate files.
6. Run `make check` after each batch of replacements.

## Move Files

When moving a file to a different directory:

1. Move the file.
2. Update the `part`/`part of` directives if the file uses them.
3. Update all imports across the project — use `rg 'old/path'` to find them.
4. Update barrel exports if the feature uses them.
5. Regenerate if the file is a `freezed`, `json_serializable`, or route input.

## Update repository_storage.dart

When a refactor changes data layer files:

1. Update imports at the top of `repository_storage.dart`.
2. Update interface getter return types in `IRepositoryStorage`.
3. Update nullable cached fields in `RepositoryStorage`.
4. Update lazy initializer expressions.
5. Update `close()` to null out the correct fields.

## Checklist Before Finishing

- [ ] All imports updated (zero broken imports in `flutter analyze`).
- [ ] `repository_storage.dart` updated if data layer was touched.
- [ ] `app_router.dart` updated if routes were touched.
- [ ] Generated files regenerated (`make gen`, `make l10n`).
- [ ] `make check` passes (format + analyze).
- [ ] No unused imports or dead code left behind.
- [ ] Behavior is unchanged — the refactor is purely structural.

## Do Not

- Do not combine refactors with feature changes in the same commit.
- Do not rename generated files manually — regenerate them.
- Do not leave old files behind after moving or merging.
- Do not break the public API of core widgets during extraction.
