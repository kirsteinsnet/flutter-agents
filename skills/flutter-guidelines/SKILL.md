---
name: flutter-guidelines
description: The team's Flutter architecture guidelines. Use before writing, changing, specifying or reviewing Flutter/Dart code in the team's projects, to learn the mandatory rules, which guide document to read and which official Flutter/Dart skill to use alongside it.
---

# Team Flutter guidelines

The full guide is in this skill's [`guide/`](guide/README.md) folder (absolute path: `${CLAUDE_SKILL_DIR}/guide/`). This skill is the **index**: rules that must never be broken and the map of which documents to read. Before implementing or reviewing, **read the documents the map points to** for the type of task. This summary does not replace the guide.

## Precedence

1. **The team's guide** (`${CLAUDE_SKILL_DIR}/guide/`), whenever it covers the subject. Exceptions approved for a project live in that project's `CLAUDE.md` (see "Per-project specifics").
2. **Official skills** from the `dart-flutter` plugin (flutter/agent-plugins) and from dart-lang/skills, for everything the guide doesn't cover and for the "how to" (commands, APIs, native setup).
3. **The project's existing code**, only as a domain reference. It is **not** a pattern reference: code outside the guide is legacy under migration.

**Intentional** divergences of the guide from official Flutter. Here the guide wins, even if an official skill says otherwise:

| Topic | Official | Guide |
|---|---|---|
| Folders | `lib/data` + `lib/domain` by type, `lib/ui/<feature>` | Everything inside `lib/features/<feature>/` (`data/`, `ui/view_models/`, `ui/widgets/`); shared code in `lib/shared/` |
| Errors between layers | Return `Result<T>` | Throw `Failure` (sealed) and catch it with `ErrorHandlerMixin.handle` in the Cubit |
| State | `ChangeNotifier` + `Command` | `Cubit` in the ViewModel role, immutable State with `status` |
| Models | `freezed`/`built_value` | Manual, strict `fromJson`/`toJson` (pattern matching + `FormatException`) |

## Per-project specifics

The guide is generic. What changes from project to project comes from the `CLAUDE.md` (and the `README.md`) of the repository you are working in:

- flavors, environments and run, build and test commands;
- integration branches and branch naming pattern;
- design system in use and internal packages;
- language of the code and of the commits.

If `CLAUDE.md` has a **"Guide exceptions"** section, the listed exceptions apply to that project. Any other part of `CLAUDE.md` that contradicts the guide describes legacy code, not a pattern to follow.

## Blocking rules

Never do what is listed below. If existing code does it, migrate it (see "Migration" below).

**Architecture and DI** ([01](guide/01-architecture.md))
- A feature importing another feature; `core/` importing `features/`.
- `static X get instance`, global variables, `get_it` or any service locator. Dependencies arrive through the constructor, in private fields; composition uses `RepositoryProvider`/`BlocProvider`, and `context.read()` only in routes and providers.
- A repository depending on another repository.
- Non-standard names: folders in `snake_case`; `ui/view_models/`, `ui/widgets/`; classes `XxxScreen`, `XxxCubit`, `XxxState`, `XxxRepository`(+`Impl`).

**State** ([02](guide/02-state-management.md))
- `BuildContext`, `material.dart`, navigation, dialog or toast inside the Cubit.
- A Cubit async operation outside `handle(...)`.
- Mutable State, without `Equatable`, or with `String errorMessage` instead of `Failure`.
- Side effect (toast, navigation) inside `BlocBuilder`; it must go in `BlocListener`.

**Components** ([03](guide/03-componentization.md))
- A widget below the Screen reading the Cubit (except a `XxxSection` of the same feature).
- `Widget _buildX()` methods instead of classes; hardcoded colors, fonts and spacing.
- A screen that loads data without the loading, error (with retry) and empty states.

**Networking and models** ([04](guide/04-networking.md))
- `Dio()` created outside `ApiClient`; a request outside `safeCall`.
- `throw Exception('...')` or `throw 'text'`. Always a subclass of `Failure`.
- A repository exposing `Response`, `Map` or `DioException`.
- A silent default value for a required field in `fromJson` (`?? ''`, `?? DateTime.now()`).

**Errors and privacy** ([05](guide/05-errors-crashlytics.md))
- Showing `error.toString()` to the user. Use `failure.userMessage`.
- Reporting the same error in two layers.
- Name, email, CPF, phone, address, request/response body or URL with a query string in Crashlytics or Analytics.

**Analytics** ([06](guide/06-analytics.md))
- `FirebaseAnalytics.instance` outside `FirebaseAnalyticsService`; `user_id` as an event parameter.
- Event name outside `<feature>_<object>_<action>` in `snake_case`.

**Navigation and push** ([07](guide/07-navigation.md), [08](guide/08-push-notifications.md))
- `Navigator.push(MaterialPageRoute(...))`; `GoRoute` without `name`; required data only in `extra`.
- Push routing logic outside `PushRouter`.

**Environment** ([09](guide/09-flavors-environments.md))
- `if (environment == ...)` or `kReleaseMode` to decide the environment; `appFlavor` read outside `main.dart`.
- Secrets in the code or in a committed `dart-define`.

## Map: task → guide → official skill

| Task | Read in the guide | Complementary official skill |
|---|---|---|
| New feature (end to end) | 01, 02, 03, 04, 06 | `flutter-apply-architecture-best-practices` (respecting the precedence) |
| Screen, component, layout | 03 | `flutter-build-responsive-layout`, `flutter-fix-layout-issues`, `flutter-add-widget-preview` |
| State / Cubit | 02 | `dart-use-pattern-matching` |
| API call, repository | 04, 05 | `flutter-use-http-package` (concepts only; the client is Dio) |
| Model / JSON | 04 | `flutter-implement-json-serialization`, `dart-use-pattern-matching` |
| Error handling, Crashlytics | 05 | `dart-fix-runtime-errors` |
| Analytics | 06 | — |
| Routes, deep link | 07 | `flutter-setup-declarative-routing` |
| Push notification | 08, 07 | — |
| Flavors, environments | 09 | — |
| Tests | 02 (Testing section) | `dart-add-unit-test`, `flutter-add-widget-test`, `dart-generate-test-mocks`, `flutter-add-integration-test` |
| Lint, analysis, formatting | — | `dart-run-static-analysis` |
| Dependency conflict | — | `dart-resolve-package-conflicts` |
| API documentation (`///`) | — | `dart-write-documentation` |

## Legacy code migration

All code written **or changed** follows the guide. When touching non-standard code:

- **Inside the touched feature:** migrate what is needed for the change to follow the pattern (e.g. a Cubit that uses `XService.instance` starts receiving the dependency through the constructor). Do it in a separate `refactor:` commit, **before** the behavior change, without changing behavior.
- **Outside the touched feature** (infrastructure in `core/`, app-wide DI, `ApiClient`, another feature): **stop and ask for confirmation**, describing what needs to change and how many files are affected.
- Divergences seen and not migrated go into the final report.

## Minimum validation

Before considering any change done:
1. `dart format` on the changed files.
2. `dart analyze` with no errors or warnings in the touched files (via the Dart MCP server's analysis tool, or the CLI).
3. New tests for what was created (Cubit with `bloc_test` and fakes from `testing/fakes/`; `fromJson` with valid and off-contract JSON) and affected existing tests passing.
