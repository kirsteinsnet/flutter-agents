# 07 · Navigation

## Goal

Centralize routes in a single place, protect screens that require login, allow navigation from services without a `BuildContext` (push, expired session) and support deep links without extra code.

---

## Structure

```
lib/core/navigation/
├── app_routes.dart              # names and paths
├── app_router.dart              # GoRouter: routes, redirect, observers
├── app_navigator.dart           # navigation without context
└── analytics_route_observer.dart
```

---

## Names and paths

```dart
// lib/core/navigation/app_routes.dart
abstract final class AppRoutes {
  static const home = 'home';
  static const login = 'login';
  static const orders = 'orders';
  static const orderDetail = 'orderDetail';
  static const profile = 'profile';
}

abstract final class AppPaths {
  static const home = '/';
  static const login = '/login';
  static const orders = '/orders';
  static const orderDetail = ':id'; // child of /orders → /orders/:id
  static const profile = '/profile';
}
```

---

## Router

```dart
// lib/core/navigation/app_router.dart
final rootNavigatorKey = GlobalKey<NavigatorState>();

GoRouter createRouter({
  required SessionCubit session,
  required AnalyticsService analytics,
}) {
  return GoRouter(
    navigatorKey: rootNavigatorKey,
    initialLocation: AppPaths.home,
    observers: [AnalyticsRouteObserver(analytics)],
    refreshListenable: GoRouterRefreshStream(session.stream),
    redirect: (context, state) {
      final loggedIn = session.state.isAuthenticated;
      final goingToLogin = state.matchedLocation == AppPaths.login;
      final isProtected = _protectedPaths.any(state.matchedLocation.startsWith);

      if (!loggedIn && isProtected) {
        // Keep the destination to return to after login
        return Uri(
          path: AppPaths.login,
          queryParameters: {'from': state.uri.toString()},
        ).toString();
      }
      if (loggedIn && goingToLogin) {
        return state.uri.queryParameters['from'] ?? AppPaths.home;
      }
      return null;
    },
    routes: [
      GoRoute(
        path: AppPaths.home,
        name: AppRoutes.home,
        builder: (_, __) => const HomeScreen(),
      ),
      GoRoute(
        path: AppPaths.login,
        name: AppRoutes.login,
        builder: (_, state) => LoginScreen(from: state.uri.queryParameters['from']),
      ),
      GoRoute(
        path: AppPaths.orders,
        name: AppRoutes.orders,
        builder: (_, __) => BlocProvider(
          create: (context) => OrdersCubit(
            repository: context.read(),
            analytics: OrdersAnalytics(context.read()),
            crashReporter: context.read(),
          )..load(),
          child: const OrdersScreen(),
        ),
        routes: [
          GoRoute(
            path: AppPaths.orderDetail,
            name: AppRoutes.orderDetail,
            builder: (_, state) {
              final id = state.pathParameters['id']!;
              return BlocProvider(
                create: (context) => OrderDetailCubit(
                  repository: context.read(),
                  crashReporter: context.read(),
                )..load(id),
                child: OrderDetailScreen(orderId: id),
              );
            },
          ),
        ],
      ),
    ],
    errorBuilder: (_, state) => NotFoundScreen(location: state.uri.toString()),
  );
}

const _protectedPaths = [AppPaths.orders, AppPaths.profile];
```

`GoRouterRefreshStream` is a simple `ChangeNotifier` that calls `notifyListeners()` on every stream event. This way, `redirect` runs again when the session changes (login, logout or expired session).

### Rules

- ✅ Every `GoRoute` has a `name`: navigation, analytics and deep links depend on it.
- ✅ Child routes for real hierarchy (`/orders/:id`). The back button and deep links work effortlessly.
- ✅ Login protection in `redirect`, never with `if (!loggedIn) Navigator.push(login)` inside the screen.
- ✅ The screen's `BlocProvider` lives in the route (see [02](02-state-management.md#provider-scope)).
- ✅ `errorBuilder` with a friendly "not found" screen.
- ❌ Direct `Navigator.push(MaterialPageRoute(...))`. Every navigable screen is a route.

---

## Tabs with preserved state

For bottom navigation (bottom bar), use `StatefulShellRoute.indexedStack`: each tab has its own stack and keeps its state when switching.

```dart
StatefulShellRoute.indexedStack(
  builder: (context, state, shell) => MainShell(navigationShell: shell),
  branches: [
    StatefulShellBranch(routes: [/* Home tab routes */]),
    StatefulShellBranch(routes: [/* Orders tab routes */]),
    StatefulShellBranch(routes: [/* Profile tab routes */]),
  ],
)
```

---

## Navigating from the UI

```dart
// Switch context (replaces the stack up to the common ancestor)
context.goNamed(AppRoutes.orders);

// Push and allow going back
context.pushNamed(
  AppRoutes.orderDetail,
  pathParameters: {'id': order.id},
);

// Wait for a result
final selected = await context.pushNamed<Address>(AppRoutes.addressPicker);
if (selected != null) cubit.setAddress(selected);
```

| Use | When |
|---|---|
| `go` / `goNamed` | Go to another area of the app: menu, tab, after login, deep link |
| `push` / `pushNamed` | Open a detail or flow the user comes back from |
| `pushReplacement` | Replace the current screen (e.g. a completed step of a flow) |
| `pop(result)` | Go back returning a value |

### Parameters

- ✅ Data required to open the screen goes in the **path** (`/orders/:id`) or in the **query** (`?tab=history`). These are the only ones that survive deep links, push and state restoration.
- ✅ `extra` only as an optional optimization (e.g. passing the already-loaded `Order` to display before the fetch). The screen must work without it.
- ❌ `extra` as the only way to receive data. The screen breaks when opened from a link.

---

## Navigating without context

Push notifications, the expired-session event and other services don't have a `BuildContext`. Use `AppNavigator`, which wraps the router:

```dart
// lib/core/navigation/app_navigator.dart
class AppNavigator {
  AppNavigator(this._router);
  final GoRouter _router;

  void go(String location) => _router.go(location);

  void goNamed(
    String name, {
    Map<String, String> pathParameters = const {},
    Map<String, dynamic> queryParameters = const {},
  }) =>
      _router.goNamed(
        name,
        pathParameters: pathParameters,
        queryParameters: queryParameters,
      );

  Future<T?> pushNamed<T>(
    String name, {
    Map<String, String> pathParameters = const {},
  }) =>
      _router.pushNamed<T>(name, pathParameters: pathParameters);

  String get currentLocation =>
      _router.routerDelegate.currentConfiguration.uri.toString();
}
```

- ✅ Created in `bootstrap.dart` and passed through the constructor only to services (push, deep link).
- ❌ Using `AppNavigator` in widgets. In the UI, use `context.goNamed`.
- ❌ A Cubit navigating, even via `AppNavigator`. The Cubit emits state; the Screen navigates in the `BlocListener`.

---

## Deep links

With named routes and path parameters, deep links work with GoRouter itself, following the [official declarative routing skill](https://github.com/flutter/agent-plugins/tree/main/skills/flutter-setup-declarative-routing):

**Android**
1. `intent-filter` with `android:autoVerify="true"` on `MainActivity` (`AndroidManifest.xml`):

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="app.example.gov.br" />
</intent-filter>
```

2. `assetlinks.json` published at `https://app.example.gov.br/.well-known/assetlinks.json`, with the SHA-256 of the certificate of **each** flavor that should open links.

**iOS**
1. Enable Flutter's deep link handler in `Info.plist`:

```xml
<key>FlutterDeepLinkingEnabled</key>
<true/>
```

If the project uses its own deep link plugin (e.g. `app_links`), set this flag to `false` to avoid conflicts.

2. Associated Domains in `Runner.entitlements` (`applinks:app.example.gov.br`).
3. `apple-app-site-association` (no extension) at `https://app.example.gov.br/.well-known/`.

**Mapping:** the public URL points to the same internal path: `https://app.example.gov.br/orders/123` → `/orders/123`. If they differ, handle the conversion in the top-level `redirect`, not in a parallel service.

**Testing:**

```bash
adb shell 'am start -a android.intent.action.VIEW -c android.intent.category.BROWSABLE -d "https://app.example.gov.br/orders/123"' br.com.example.app.dev
```

```bash
xcrun simctl openurl booted "https://app.example.gov.br/orders/123"
```
