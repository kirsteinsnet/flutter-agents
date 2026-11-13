# <Feature name>

> **Status:** draft | ready
> **Figma:** <file/frame link>
> **API contract:** <path to the OpenAPI/JSON or "pasted in the conversation">
> **Date:** <YYYY-MM-DD> · **Author:** <git user.name>

<!--
Rules for this template:
- Keep the IDs (FR-xx, BR-xx, AC-xx) stable across versions. When removing an item, strike it through instead of renumbering.
- The spec is only "ready" with section 11 empty.
- No real personal data in examples (names, emails, CPF, phone numbers, addresses). Use fictitious values.
- Section 7 describes structure and names, not code.
- Delete this comment in the final spec.
-->

## 1. Context and goal

<Why the feature exists, for whom and what problem it solves. 2 to 5 sentences.>

## 2. Out of scope

- <What is explicitly not part of this delivery.>

## 3. Functional requirements

| ID | Requirement |
|---|---|
| FR-01 | <The app must ...> |

## 4. Business rules

| ID | Rule |
|---|---|
| BR-01 | <E.g.: the "Cancel" button only appears when `status == pending`.> |

## 5. Screens and states

### <Screen name> — [Figma](<node link>)

| State | What the user sees | Source |
|---|---|---|
| Loading | <skeleton/spinner> | <Figma node or "design system default"> |
| Success | <content> | |
| Empty | <message + action> | |
| Error | <`failure.userMessage` + "Try again" button> | |
| <Variation> | <e.g.: in-store pickup order> | |

**Design system components:** <components and tokens used, mapped from the Figma variables.>

## 6. Navigation

| Route (`name`) | Path | Required parameters (in the path) | Optional `extra` |
|---|---|---|---|
| `<orderDetail>` | `<:id>` (child of `</orders>`) | `<id>` | <—> |

- **Entry:** <where this screen is reached from>
- **Exit:** <where it goes and what happens on going back>
- **Deep link / push:** <payload type and destination, or "not applicable">

## 7. Technical design

Follows the `flutter-guidelines` guide. Folder: `lib/features/<feature>/`.

### Models (`data/models/`)

| Model | Field | Dart type | Required | API field | Notes |
|---|---|---|---|---|---|
| `<Order>` | `<id>` | `String` | yes | `<id>` | |

### Repository (`data/repositories/`)

| Method | Endpoint | Returns | Expected `Failure`s |
|---|---|---|---|
| `<getOrder(String id)>` | `<GET /orders/{id}>` | `<Future<Order>>` | `<NotFoundFailure, NetworkFailure, ...>` |

### Cubit and State (`ui/view_models/`)

- **State:** `status` (`initial`, `loading`, `success`, `failure`), <fields>, `Failure? failure`.
- **Cubit actions:** `<load()>`, `<cancel()>` — all via `handle(...)`.

### Widgets (`ui/widgets/`)

- `<XxxScreen>` (reads the Cubit), `<XxxSection>` (reads the Cubit, same feature), <pure components>.

## 8. Analytics

| Event | When | Parameters |
|---|---|---|
| `<order_detail_cancel_tap>` | <tap on "Cancel"> | `<order_status>` |

> `screen_view` is automatic via the navigation observer; do not list it here.

## 9. Acceptance criteria

| ID | Scenario | Reference |
|---|---|---|
| AC-01 | **Given** <context> **when** <action> **then** <result> | FR-01 |

## 10. Expected tests

- **Models:** `fromJson` with valid JSON and with JSON outside the contract (<fields>).
- **Repository:** success and each `Failure` from section 7.
- **Cubit (`bloc_test`):** <state sequences per action>.
- **Widget:** <states from section 5 that need a test>.

## 11. Open questions

| # | Question | Assumption adopted |
|---|---|---|
| 1 | <question> | <assumption, or "none — blocking"> |
