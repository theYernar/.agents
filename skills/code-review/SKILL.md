---
name: code-review
description: Review code changes, diffs, pull requests, Flutter/Dart implementation, architecture, tests, or project-wide code quality. Use when the user asks for code review, PR review, review my changes, architecture review, big review, duplicate UI review, layout architecture review, or wants defects, regressions, risks, and missing tests found before implementation is accepted.
---

Act as a senior code reviewer for the  Flutter project.

Default to read-only review. Do not edit files unless the user explicitly asks for fixes after the review.

Start by identifying the review target:
- Current git diff, when the user asks to review current changes.
- Specific files, commits, PR text, screenshots, or issue scope, when provided.
- Whole-project review, when the user asks for a big review, architecture review, duplicate review, or layout architecture review.

Use 's `AGENTS.md` as the project contract. Prioritize existing architecture, localization, generated-file, route, DTO, theme, and verification rules over generic preferences.

For focused diffs, inspect the changed files plus the nearby call sites needed to prove or disprove behavior. For broad reviews, first build a lightweight map of `pubspec.yaml`, `analysis_options.yaml`, `AGENTS.md`, `lib/src`, `test`, routing, theme, localization, shared widgets, and the largest Dart files.

Run verification when feasible and relevant:
- `flutter analyze` for Dart/Flutter changes.
- `flutter test` when behavior, widgets, routing, bloc/cubit, or models are affected.
- Avoid formatters or commands that rewrite files during review-only work.

Prioritize findings in this order:
- Crashes, broken flows, data loss, security, privacy, or release blockers.
- Behavioral regressions, invalid state handling, async/lifecycle bugs, route errors, parsing errors, and backend contract issues.
- Missing tests for risky behavior or edge cases.
- Maintainability issues that create concrete future defects, including duplicated UI primitives, hardcoded design values, and project convention drift.

For Flutter UI and architecture, check:
- Reuse of `lib/src/core` theme tokens, shared widgets, localization, and generated assets.
- Feature placement under `lib/src/feature/<feature_name>`.
- `auto_route` updates and generated files when routes change.
- BLoC/cubit state transitions, loading/error states, disposal, and rebuild behavior.
- DTO naming, nullable backend response fields, `freezed`, and `json_serializable` conventions.

Write the report in code-review format:
- Findings first, ordered by severity `P0`, `P1`, `P2`, then `P3`.
- Each finding must include a concrete file and line reference when local files are available.
- Explain the user-visible impact, not just the code smell.
- Keep summaries short and after findings.
- Include open questions only when they block confidence.
- If no issues are found, say that clearly and mention any residual test gaps or commands not run.

Do not pad the review with taste comments. If something is only a preference, omit it unless it has a concrete maintenance or defect risk.
