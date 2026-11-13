# Architecture Guidelines for Flutter Projects

Reference guide for developers who create or maintain Flutter apps following the same line of development. It grew out of experience accumulated on real projects, but it is **generic**: the examples use a fictional domain (`orders`, `Order`) and the `my_app` package.

The guide follows the official recommendations of the Flutter team (see [Alignment with official Flutter](#alignment-with-official-flutter)).

---

## Reference stack

| Concern | Choice |
|---|---|
| Architecture | MVVM from the official guide (View + ViewModel, Repository + Service) |
| State management | `flutter_bloc` (Cubit in the ViewModel role) + `equatable` |
| Dependency injection | `provider`, via `flutter_bloc`'s `RepositoryProvider`/`BlocProvider` |
| Navigation | `go_router` |
| HTTP | `dio` |
| Observability | Firebase Crashlytics + Firebase Analytics |
| Push | Firebase Cloud Messaging + `flutter_local_notifications` |
| Remote configuration | Firebase Remote Config (optional) |
| Tests | `bloc_test`, fakes, `mockito` (`@GenerateNiceMocks`) for interactions, `http_mock_adapter` |
| Preview | Widget Previewer (`@Preview`) |

Swapping a piece of this stack is an architecture decision: record the reason in the project's repository (ADR or a README section).

---

## Index

| # | Document | Topic |
|---|---|---|
| 01 | [Architecture](01-architecture.md) | MVVM layers, folder organization, dependency injection, bootstrap |
| 02 | [State management](02-state-management.md) | Cubit as ViewModel, State, side effects, scope, testing with fakes |
| 03 | [Componentization](03-componentization.md) | Design system, shared and local widgets, screen states, previews |
| 04 | [Networking](04-networking.md) | ApiClient, interceptors, `safeCall`, repositories, strict models |
| 05 | [Errors and Crashlytics](05-errors-crashlytics.md) | Typed `Failure`, `ErrorHandlerMixin`, global errors, privacy |
| 06 | [Analytics](06-analytics.md) | `AnalyticsService`, per-feature events, naming |
| 07 | [Navigation](07-navigation.md) | GoRouter, protected routes, navigation without `context`, deep links |
| 08 | [Push notifications](08-push-notifications.md) | FCM, local notifications, payload-based routing |
| 09 | [Flavors and environments](09-flavors-environments.md) | `appFlavor`, `AppConfig`, Firebase per environment |

Read them in order the first time. After that, use them as a reference.

---

## Principles

1. **One-way dependencies.** The UI knows the Cubit, the Cubit knows the Repository, the Repository knows the ApiClient. Never the other way around.
2. **Self-contained features.** Everything that belongs to a feature lives in its folder. Only what is used by two or more features moves up to `shared/`.
3. **Dependencies through the constructor.** No business class fetches its own dependencies, and there are no global objects. This makes everything testable with fakes.
4. **Typed errors, reported once.** Every failure becomes a `Failure` with a user-facing message and is reported in a single place.
5. **Privacy by default.** No personal data (name, email, ID document) goes to Crashlytics or Analytics.
6. **Simplicity.** Don't create a layer, abstraction or package that the current problem doesn't require.

---

## Alignment with official Flutter

Reference sources, in order of priority:

1. [docs.flutter.dev](https://docs.flutter.dev/), especially the [architecture guide](https://docs.flutter.dev/app-architecture/guide), the [recommendations](https://docs.flutter.dev/app-architecture/recommendations) and the [case study](https://docs.flutter.dev/app-architecture/case-study).
2. Skills maintained by the Flutter team: [flutter/agent-plugins](https://github.com/flutter/agent-plugins) and [dart-lang/skills](https://github.com/dart-lang/skills). Use them with AI agents in day-to-day work too.

**Intentional** divergences, each explained in its own document:

| Topic | Official | This guide | Where |
|---|---|---|---|
| Folders | Data by type (`lib/data`), UI by feature (`lib/ui`) | Data and UI inside the feature | [01](01-architecture.md#difference-from-the-official-guide) |
| Errors between layers | Return `Result<T>` | Throw `Failure` and catch it in the Cubit | [04](04-networking.md#difference-from-the-official-guide) |
| State tool | `ChangeNotifier` + `Command` (optional) | `Cubit` as ViewModel | [02](02-state-management.md#the-cubit-is-the-viewmodel) |
| Models | `freezed`/`built_value` (recommended) | Manual, strict parsing, no codegen | [04](04-networking.md#models) |

Any other difference from the official sources is a mistake in the guide: open a merge request.

---

## Use with AI agents (Claude Code)

This guide is distributed as part of the `flutter-agents` plugin, together with the `flutter-guidelines` skill (the guide's index) and the `flutter-dev` agent. Installation and usage are in the [repository README](../../../README.md).

---|---|
| `.claude/skills/flutter-guidelines/SKILL.md` | Guide index: precedence, blocking rules and the "task → document → official skill" map. Used by all agents. |
| `.claude/skills/flutter-spec/` + `.claude/agents/flutter-spec.md` | Feature spec (product + technical design following the guide) from text, Figma and the API contract, committed to `docs/specs/`. |
| `.claude/agents/flutter-dev.md` | Development agent: implements following the guide, validates and makes atomic commits on a feature branch (no push). |
| `.claude/settings.json` | Declares the official `dart-flutter` plugin (Flutter/Dart skills + Dart MCP server). |

The first time, each dev installs the official plugin:

```bash
claude plugin install dart-flutter@dart-flutter --scope project
```

To use the agent, ask Claude Code something like "use flutter-dev to implement ...", or pass the path to a spec.

To bring it to another project, copy `docs/flutter-guidelines/` and the `.claude/` folder (`settings.json`, `skills/flutter-guidelines/`, `agents/`).

---

## How each document is organized

- **Goal**: what the topic solves.
- **Rules**: list of ✅ DO / ❌ DON'T.
- **Why**: the reasoning behind the rules, to decide the cases the guide doesn't cover.
- **Reference code**: ready-to-copy implementation to adapt.

## How to propose changes

Open a merge request changing the topic's document. Rule changes (not just wording) need approval from at least one tech lead from another project that follows the guide.
