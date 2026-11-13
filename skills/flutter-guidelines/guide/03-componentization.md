# 03 · Componentization

## Goal

Define where each widget lives, how it receives data and how to keep the interface consistent across screens and across projects.

---

## Three component levels

| Level | Where | What it is | Example |
|---|---|---|---|
| **1. Design system** | External package (Git) or `lib/core/theme/` | Tokens (colors, typography, spacing, radius) and basic components with no app rules | `DsButton`, `DsTextField`, `DsCard` |
| **2. App shared** | `lib/shared/ui/` | App components used by 2+ features | `ErrorView`, `EmptyView`, `AppScaffold`, `LoginRequiredView` |
| **3. Feature local** | `features/<f>/ui/widgets/` | Everything that only makes sense in one feature | `OrderCard`, `OrderStatusChip` |

### Promotion rule

1. Every widget is born at **level 3**.
2. When a **second** feature needs it, it moves up to **level 2** (and loses any reference to the original feature).
3. When it is useful to **other apps**, propose adding it to **level 1**.

Never copy a widget from one feature to another. Promote it.

---

## Presentational components

Widgets below the Screen do not read the Cubit: they receive data and callbacks. This is what the official guide calls keeping Views "dumb".

```dart
// ✅ Receives ready-made data and returns intents
class OrderCard extends StatelessWidget {
  const OrderCard({
    super.key,
    required this.order,
    required this.onTap,
    this.onFavoriteToggle,
  });

  final Order order;
  final VoidCallback onTap;
  final VoidCallback? onFavoriteToggle;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return DsCard(
      onTap: onTap,
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(order.title, style: theme.textTheme.titleMedium),
          const SizedBox(height: AppSpacing.xs),
          OrderStatusChip(status: order.status),
        ],
      ),
    );
  }
}
```

```dart
// ❌ Component that knows the Cubit: only works inside this screen
class OrderCard extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final order = context.watch<OrdersCubit>().state.orders[index];
    ...
  }
}
```

### Why

A component that receives data can be reused, tested with `pumpWidget` without setting up a Cubit, and viewed in isolation in the Widget Previewer (see below). A component that reads the Cubit is stuck to the screen where it was born.

### Who may read the Cubit

- The **Screen**.
- Large **sections** of the Screen (`OrdersHeaderSection`, `OrdersListSection`) when that avoids passing ten parameters down. The section lives in the same feature and its name ends in `Section`.

---

## Building rules

- ✅ Extract widgets into **classes**, not `Widget _buildHeader()` methods. Classes can be `const`, have their own `build` (Flutter redraws only them) and show up by name in DevTools.
- ✅ `const` on every possible constructor and instance.
- ✅ Colors, fonts, spacing and radii come from the theme/tokens: `Theme.of(context)`, `AppSpacing.md`, `AppColors.primary`.
- ✅ A feature's fixed texts live in a feature constants class (`OrdersStrings`), not scattered across widgets.
- ✅ One widget per file, except small private widgets used only there.
- ✅ Images and icons with `semanticLabel`; touch targets of at least 48×48.
- ❌ Loose `Color(0xFF...)`, `fontSize: 14`, `EdgeInsets.all(13)` in the code.
- ✅ Layout decisions based on available space: `LayoutBuilder` (parent's space) and `MediaQuery.sizeOf(context)` (window size), with `Expanded`/`Flexible` to distribute space.
- ✅ `ListView.builder`/`GridView.builder` for long lists or lists of unknown size.
- ❌ `MediaQuery.of(context).size` (redraws on any `MediaQuery` change; use `sizeOf`).
- ❌ Deciding layout by device type ("is it a tablet?") or by orientation at the top of the tree. Apps run in resizable windows, multi-window and foldables.
- ❌ Widgets with more than ~150 lines of `build`. Split them.

---

## Required screen states

Every screen that loads data handles the four states, using the shared components:

| State | Component | Note |
|---|---|---|
| Loading | Feature skeleton (`OrdersSkeleton`) | A skeleton shaped like the content, not a spinner in the middle of the screen |
| Error | `ErrorView(failure:, onRetry:)` | Shows `failure.userMessage` and a "Try again" button |
| Empty | `EmptyView(message:, action:)` | Text explaining why and, if there is one, the action to get out of the empty state |
| Success | Content | With `RefreshIndicator` when the data can change |

```dart
// lib/shared/ui/error_view.dart
class ErrorView extends StatelessWidget {
  const ErrorView({super.key, required this.failure, this.onRetry});

  final Failure failure;
  final VoidCallback? onRetry;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(AppSpacing.lg),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(failure.icon, size: 48),
            const SizedBox(height: AppSpacing.md),
            Text(failure.userMessage, textAlign: TextAlign.center),
            if (onRetry != null) ...[
              const SizedBox(height: AppSpacing.md),
              DsButton.secondary(
                label: 'Try again',
                onPressed: onRetry,
              ),
            ],
          ],
        ),
      ),
    );
  }
}
```

`failure.icon` can be an extension on `Failure` inside `shared/ui/` (a "no connection" icon for `NetworkFailure`, for example), keeping `core/errors/` free of Flutter.

---

## User feedback

Centralize toasts/snackbars and dialogs in `shared/ui/` (`AppToast.success`, `AppToast.error`, `ConfirmDialog.show`). Features call these helpers from the `BlocListener` and never build a `SnackBar` by hand.

---

## Widget previews

Use the official [Widget Previewer](https://docs.flutter.dev/tools/widget-previewer) to develop and review components in isolation, without navigating to the screen.

```dart
// features/orders/ui/widgets/order_card.dart
import 'package:flutter/widget_previews.dart';

@Preview(name: 'Open order', group: 'Orders')
Widget orderCardOpenPreview() => OrderCard(order: Order.sample(), onTap: () {});

@Preview(name: 'Completed order', group: 'Orders', brightness: Brightness.dark)
Widget orderCardDonePreview() =>
    OrderCard(order: Order.sample(status: OrderStatus.done), onTap: () {});
```

- ✅ Previews for level 2 and 3 components with more than one visual state.
- ✅ Create your own annotations (extending `Preview`) to apply the app theme to all previews.
- ❌ Dependencies on `dart:io`, native plugins or a Cubit inside the preview: the previewer runs on the web. One more reason for components to receive data through parameters.

Run it from the IDE or with:

```bash
flutter widget-preview start
```

---

## External design system

If the organization maintains a design system as a package:

- ✅ Depend on it by version tag (`ref: v1.4.0`), never by branch.
- ✅ A missing component that is generic → contribute it to the package; in the meantime, create it at level 2 with the same name it will have there.
- ❌ Overriding design system styles inside a feature. If the variation is needed, it becomes an official variant of the component.
