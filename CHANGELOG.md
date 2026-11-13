# Changelog

Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow [SemVer](https://semver.org/).

## [Unreleased]

### Changed

- The whole plugin (guide, skills, agents, spec template, README and manifests) is now written in English. Agents reply in the user's language.
- Guide files renamed to English: `01-architecture.md`, `02-state-management.md`, `03-componentization.md`, `04-networking.md`, `05-errors-crashlytics.md`, `07-navigation.md`, `09-flavors-environments.md`.
- Spec IDs are now `FR` (functional requirement), `BR` (business rule) and `AC` (acceptance criterion); the draft status is `draft` and section 11 is "Open questions".

## [0.2.0] - 2026-11-12

### Added

- `flutter-spec` skill: writes a feature spec (product + technical design per the guide) from text, a Figma link (via the Figma MCP) and the API contract, asking about one gap at a time, and commits it to `docs/specs/<slug>.md` on the feature branch.
- Spec template with traceable IDs (`RF`, `RN`, `CA`), screen states, technical design, analytics, expected tests and open questions.
- `flutter-spec` agent: the same spec in non-interactive mode; gaps become "Open questions" and the spec stays as a draft.
- README: Figma MCP setup and the spec → implementation flow.

### Changed

- `flutter-dev`: the final report cites the requirement and acceptance criteria IDs when the input is a spec.

## [0.1.0] - 2026-11-11

### Added

- Flutter architecture guide (01 to 09), aligned with official Flutter, with 4 documented intentional divergences.
- `flutter-guidelines` skill: guide index, blocking rules, "task → document → official skill" map, legacy migration policy and per-project specifics via `CLAUDE.md`.
- `flutter-dev` agent: development following the guide, with per-slice validation and atomic commits.
- Dependency on the official `dart-flutter` plugin.
