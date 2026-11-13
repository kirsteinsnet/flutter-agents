# Flutter Agents

Claude Code plugin with the **team's Flutter architecture guidelines** and the **agents** that apply them. A single repository serves every Flutter project: when the guide or an agent changes here, every project gets the update.

It works alongside the official [`dart-flutter`](https://github.com/flutter/agent-plugins) plugin from the Flutter team, which provides the official Flutter/Dart skills and the Dart MCP server. It is declared as a dependency and installed with it.

## What's here

| Component | Type | Role |
|---|---|---|
| [`flutter-guidelines`](skills/flutter-guidelines/SKILL.md) | Skill | Guide index: precedence, intentional divergences from official Flutter, blocking rules, "task → document → official skill" map and legacy migration policy. Used by every agent. |
| [Full guide](skills/flutter-guidelines/guide/README.md) | Documentation | 9 documents: architecture, state, componentization, networking, errors/Crashlytics, analytics, navigation, push, flavors. |
| [`flutter-spec`](skills/flutter-spec/SKILL.md) | Skill | Writes the spec for a feature (product + technical design per the guide) from text, a Figma link and the API contract, **talking with you** to close the gaps. Commits the spec to `docs/specs/` on the feature branch. |
| [`flutter-spec`](agents/flutter-spec.md) | Agent | Same spec, in **non-interactive** mode: for input that is already complete or to chain with `flutter-dev`. Gaps become "Open questions" instead of questions. |
| [`flutter-dev`](agents/flutter-dev.md) | Agent | Implements features, bugfixes and refactorings following the guide, validates (format, analyze, tests) and makes atomic commits on a feature branch, without pushing. |

Upcoming agents: code review and distribution (Firebase App Distribution and stores).

## Prerequisites

- Claude Code installed and authenticated.
- `dart` and `flutter` on the `PATH` of the shell that launches Claude Code, so the Dart MCP server can start. With FVM, add to `~/.zshrc`:

  ```bash
  export PATH="$HOME/fvm/default/bin:$PATH"
  ```

- Figma MCP configured, to generate specs (see below).

### Figma MCP

The `flutter-spec` skill and agent read the screens from the Figma link, using Figma's official MCP. The plugin does **not** declare it: each dev configures it once, at user scope. Without it, `flutter-spec` stops and asks for it to be configured (screenshots don't replace the link).

Remote server (doesn't need the desktop app open):

```bash
claude mcp add --scope user --transport http figma https://mcp.figma.com/mcp
```

Then, in a Claude Code session in the terminal, run `/mcp`, choose `figma` and authenticate with your Figma account.

Alternative, desktop app server (requires Figma desktop open with the Dev Mode MCP server enabled in preferences):

```bash
claude mcp add --scope user --transport http figma-desktop http://127.0.0.1:3845/mcp
```

Check with `claude mcp list` and open a new session.

## Add to a project

1. In the project's `.claude/settings.json` (versioned), declare both marketplaces and enable the plugins:

   ```json
   {
     "extraKnownMarketplaces": {
       "flutter-agents": {
         "source": {
           "source": "github",
           "repo": "kirsteinsnet/flutter-agents"
         }
       },
       "dart-flutter": {
         "source": {
           "source": "github",
           "repo": "flutter/agent-plugins"
         }
       }
     },
     "enabledPlugins": {
       "flutter-agents@flutter-agents": true,
       "dart-flutter@dart-flutter": true
     }
   }
   ```

2. Each dev, once per machine and per project:

   ```bash
   claude plugin install flutter-agents@flutter-agents --scope project
   ```

   `dart-flutter` is installed as a dependency. If it isn't, run `claude plugin install dart-flutter@dart-flutter --scope project`.

3. In the project's `CLAUDE.md`, describe what is specific to it (the guide is generic):
   - flavors, environments and run, build and test commands;
   - integration branches (e.g. `main`, `develop`, `hom`) and branch naming pattern;
   - design system and internal packages;
   - optional: a **"Guide exceptions"** section with approved deviations for that project.

   Recommended: a short section saying that code outside the guide is legacy under migration, so the agent doesn't copy it as a pattern.

4. Open a **new session** of Claude Code (plugins and agents load at session start).

## Usage

### 1. Spec

Conversational (recommended for a new feature or a vague requirement):

> /flutter-agents:flutter-spec order detail screen. Figma: <link>. API: `docs/api/order.json`

The skill reads the project, the guide, Figma and the API contract and asks one thing at a time until the gaps are closed (screen states, optional fields, rules, navigation, analytics). Then it writes `docs/specs/<slug>.md`, shows a summary and commits on the `feature/<slug>` branch, without pushing.

Delegating (input already complete, or chaining with the implementation):

> use the flutter-spec agent to generate the order detail spec; Figma: <link>; API in `docs/api/order.json`; cancellation only when pending

The agent doesn't ask: whatever is missing goes into the spec's "Open questions" section, with the assumption adopted, and the spec stays as `draft`.

The spec has traceable IDs (`FR-xx`, `BR-xx`, `AC-xx`) and is versioned for auditing. The skeleton is in [`skills/flutter-spec/template.md`](skills/flutter-spec/template.md).

### 2. Implementation

On the same branch as the spec, ask in Claude Code:

> use flutter-dev to implement the order detail screen according to `docs/specs/order-detail.md`

The agent reads the project's `CLAUDE.md`, the guide documents the task requires and the corresponding official skills. Then it plans, creates the branch, implements in validated slices and delivers a report with commits, migrations and what was not verified.

## Update

```bash
claude plugin marketplace update flutter-agents
claude plugin update flutter-agents@flutter-agents
```

## Contributing

- Guide change: edit `skills/flutter-guidelines/guide/` and, if a rule changes, also update the blocking rules and the map in `skills/flutter-guidelines/SKILL.md`.
- A rule change (not just wording) needs approval from at least one tech lead of another project that follows the guide.
- Bump the `version` in `.claude-plugin/plugin.json` (semver) and record it in the [CHANGELOG](CHANGELOG.md).
- Before the merge request, run:

  ```bash
  claude plugin validate .
  ```

## Structure

```
.claude-plugin/
  plugin.json          # plugin manifest (version, dart-flutter dependency)
  marketplace.json     # this repo is also the marketplace
agents/
  flutter-dev.md
  flutter-spec.md      # spec in non-interactive mode
skills/
  flutter-guidelines/
    SKILL.md           # guide index
    guide/             # full guide (01..09 + README)
  flutter-spec/
    SKILL.md           # spec flow and rules
    template.md        # spec skeleton
```
