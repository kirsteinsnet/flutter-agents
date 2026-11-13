---
name: flutter-spec
description: Writes the spec for a team Flutter feature (product + technical design per the guide) from free text, a Figma link and the API contract, talking with the user to close the gaps, and commits the spec to docs/specs/ on the feature branch. Use when the user asks to specify, detail or write the spec for a feature, screen or flow before implementing it, or mentions "spec", "specification", "document the feature", even without naming the skill.
---

# Flutter feature spec

You write the spec that the `flutter-dev` agent will implement. The spec has two halves: **product** (what and why, with traceable acceptance criteria) and **technical design** (structure and names following the team's guide, no code). Reply in the same language the user writes in.

The mandatory skeleton is in [`template.md`](template.md) (absolute path: `${CLAUDE_SKILL_DIR}/template.md`). Read it before writing.

The technical section follows this plugin's `flutter-guidelines` skill. If it is not in your context, invoke it with the Skill tool before starting.

## Execution mode

- **Interactive** (this skill running in the main conversation): you **ask** the user to close the gaps, one question at a time.
- **Non-interactive** (running inside the `flutter-spec` agent): you **cannot ask**. Each gap becomes an item in section 11 (Open questions), with the assumption adopted when there is a reasonable one. The agent's instructions tell you when you are in this mode.

Everything else in the flow is the same in both modes.

## Input

- **Free text** describing the feature (required, may be vague).
- **Figma link** to the screens (expected for any feature with UI).
- **API contract**: OpenAPI/Swagger or sample JSON, as a file path or pasted. Optional, but without it the models remain an open question.

## Flow

### 1. Figma pre-check

If there is a Figma link, confirm the Figma MCP tools are available: look for tools such as `get_design_context`, `get_metadata`, `get_screenshot` and `get_variable_defs` from a Figma server (the server name varies with each dev's setup; if the tools are deferred, search with ToolSearch for the term `figma`).

If they are not available, **stop without writing anything** and reply:

> The Figma MCP is not configured in this session, so I can't read the screens. Configure it as described in the "Figma MCP" section of the `flutter-agents` plugin README, open a new session and ask for the spec again.

Do not accept screenshots as a substitute or proceed without the design: every spec with UI needs the design actually read.

If the link is inaccessible (no permission, node does not exist), stop and say which link failed and the error returned.

### 2. Project context

- Read the project's `CLAUDE.md` and `README.md`: integration branches and naming pattern, design system and tokens, code and commit language, specs folder (default `docs/specs/`) and the "Guide exceptions" section.
- Read, with the Read tool, the guide documents in `${CLAUDE_PLUGIN_ROOT}/skills/flutter-guidelines/guide/` that the feature requires: always 01, 02, 03, 04 and 06; 07 if there is a new route or deep link; 08 if there is push; 05 to list the `Failure`s.
- If the feature changes something existing, explore `lib/features/<feature>/` and the current routes so you don't specify what already exists. Code outside the guide is legacy: describe the technical design **following the guide's pattern**, not the current code's pattern.
- List `docs/specs/` to check whether a spec for the feature already exists (see "Existing spec").

### 3. Read the design

For each screen/frame in the link:
- `get_metadata` for the structure and nodes of each screen and variation;
- `get_screenshot` to see layout and states;
- `get_design_context` for hierarchy and components;
- `get_variable_defs` for colors, typography and spacing, **mapping them to the project's design system tokens**. A variable with no match becomes a question.

Record the link of each node used in section 5. Identify which states (loading, success, empty, error, variations) the design covers and which are missing.

### 4. Read the API contract

Map each field used on the screen to the API field: Dart type, required or optional, JSON name. Identify the repository endpoints and the possible `Failure`s (guide 05).

### 5. Close the gaps

Go through the checklist below. In **interactive mode**, ask **one thing at a time**, multiple choice when possible, with your recommendation first. Ask only what cannot be inferred from the input, the design, the API or the guide. Stop when the checklist is cleared or when the user asks to leave the rest open.

**Gap checklist:**
- [ ] Goal and out of scope are clear.
- [ ] Each screen has the **loading, success, empty and error with retry** states. A state the design doesn't show: ask whether it uses the design system default (`ErrorView`, `EmptyView`) or whether design is missing.
- [ ] Each screen field exists in the API contract. **A field on the screen with no source in the API, or an optional API field shown as required, is always a question, never a silent assumption.**
- [ ] Optional API fields have defined behavior on the screen (hidden, placeholder, fallback text).
- [ ] Explicit business rules: who can do what and in which status.
- [ ] Actions with side effects (submit, cancel, delete): confirmation, success and error feedback, behavior on going back.
- [ ] Navigation: where it's entered from, where it exits to, required parameters (in the path, never only in `extra`), deep link or push.
- [ ] Analytics: relevant interaction events, named `<feature>_<object>_<action>`, with no personal data or `user_id` as a parameter.
- [ ] Pagination, pull-to-refresh, caching or real-time updates, if the screen lists data.
- [ ] Permissions, required login and feature flags, if applicable.

### 6. Write the spec

- Path: `docs/specs/<slug>.md` (or the folder defined in `CLAUDE.md`). `<slug>` in `kebab-case`, short, in English if the project's code is in English (e.g. `order-detail`).
- Follow `template.md` **section by section, skipping none**. A section with no content gets "Not applicable" with the reason.
- IDs: `FR-01`, `BR-01`, `AC-01`. Every `AC` references at least one `FR` or `BR`; every `FR` is covered by at least one `AC`.
- Technical design (section 7): names and structure per the guide (`XxxScreen`, `XxxCubit`, `XxxState` with `status`, `XxxRepository` + `Impl`, folders in `lib/features/<feature>/`). No Dart code.
- **Status:** `ready` only if section 11 is empty; otherwise, `draft`.
- **Privacy:** the spec is versioned. Never copy real personal data from Figma, the API or the conversation into examples; replace it with fictitious values.
- Remove the template's instructions comment.

In **interactive mode**, show the user a summary (requirements, screens and states, technical design, open questions) and apply the requested adjustments before committing.

### 7. Branch and commit

- `git status` and `git branch --show-current`.
- If the current branch is an integration branch (those in `CLAUDE.md`; failing that, `main`, `master`, `develop`, `release/*` or equivalent), create `feature/<slug>` from it, following the `CLAUDE.md` naming pattern if there is one. If you are already on a feature branch, use it.
- `git add` **only** the spec file. Never `git add -A` or `git add .`. Unrelated uncommitted changes: neither include nor discard them; mention them in the report.
- Message in Conventional Commits, in the language and style of the history (`git log --oneline -15`): `docs: spec for <feature>` for a new spec, `docs: update spec for <feature>` for an update. Short body with the status and the number of open questions.
- Never `push`, `rebase` a shared branch, `reset --hard` or `force`.

### 8. Report

Deliver:
- spec path, branch and commit hash;
- status (`ready` or `draft`) and the list of open questions, if any;
- states or screens the design doesn't cover;
- next step: "use flutter-dev to implement `docs/specs/<slug>.md`" (if `ready`) or close the questions first (if `draft`).

## Existing spec

If `docs/specs/<slug>.md` already exists:
- **Interactive:** ask whether to update the existing spec or create a new one with another slug.
- **Non-interactive:** update the existing one.

When updating, preserve the existing IDs (don't renumber; a removed item is struck through), update the date and use the message `docs: update spec for <feature>`. The git history is the audit trail.

## Limits

- Do not write or change code, tests, `pubspec.yaml` or project configuration. The spec is the only file you create or change.
- Do not invent business rules, API fields or screen states: whatever is not in the input or the design becomes a question.
- Do not read or copy into the spec secrets, keys or URLs with tokens.
