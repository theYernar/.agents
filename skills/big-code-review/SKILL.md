---
name: big-code-review
description: Perform a broad, read-only code review of the  Flutter project. Use when the user asks for "большое ревью", architecture review, project-wide review, duplicate UI review, layout architecture review, maintainability audit, or asks to inspect the whole app rather than a narrow diff.
---

# Big Code Review

Act as a senior reviewer for the  Flutter project. Keep the review read-only unless the user explicitly asks for fixes after the report.

## Scope

Review the whole project by default. If the user provides a narrower area, keep the project-wide mindset but focus the report on that area.

Use `AGENTS.md` as the project contract. Prioritize -specific architecture, localization, generated-file, route, DTO, Bloc/Cubit, theme, and verification rules over generic preferences.

If the user says the project is currently in layout/UI stage, do not treat mock data, temporary static catalogs, or placeholder values as architecture debt. Focus on UI architecture, layout primitives, design tokens, navigation, localization readiness, testability, and reuse.

## Project Map

Before judging, build a lightweight map:

- Read `pubspec.yaml`, `analysis_options.yaml`, `AGENTS.md`, and `test`.
- Inspect `lib/src/core`, `lib/src/feature`, routing, theme, localization, generated assets, shared widgets, REST/client infrastructure, and initialization/composition root.
- Identify the largest Dart files and the most repeated UI patterns with `rg`, `rg --files`, and line counts.
- For suspected issues, inspect nearby call sites before reporting.

## Review Passes

Look for concrete defects and maintainability risks in this order:

- Release blockers: crashes, broken flows, data loss, security/privacy problems, invalid app startup, broken routing.
- Behavioral regressions: async/lifecycle bugs, invalid Bloc/Cubit transitions, stale generated files, parsing/backend contract issues, missing error states.
- UI architecture debt: duplicated scaffolds, app bars, CTA buttons, tiles/cards, chips, switches, bottom sheets, dialogs, hardcoded colors/spacing, and one-off layout primitives that should share core widgets or theme tokens.
- Project convention drift: feature placement, DTO nullability, `freezed`/`json_serializable` naming, localization strings, generated assets, `auto_route`, dependency registration.
- Missing tests around risky behavior, models, routing, Bloc/Cubit, or reusable widgets.

## Verification

Run verification when feasible:

- `flutter analyze`
- `flutter test`

Do not run commands that rewrite files during review-only work, including `dart format .`, `flutter gen-l10n`, or `dart run build_runner build`, unless the user explicitly asks to fix or regenerate.

If a verification command cannot run, report the exact command and the first blocking reason.

## Report Format

Start with findings. Order by severity: `P0`, `P1`, `P2`, then `P3`.

Each finding must include:

- A concrete file and line reference.
- The user-visible or maintenance impact.
- The smallest practical direction for a fix.

Keep the summary short and place it after findings. Include a verification block with commands run and results. If no issues are found, say so clearly and mention any residual test gaps or commands not run.

Omit taste comments. If something is only a preference and has no concrete defect or future-risk path, leave it out.
