# 07 · Navegação

## Objetivo

Centralizar rotas em um só lugar, proteger telas que exigem login, permitir navegação a partir de serviços sem `BuildContext` (push, sessão expirada) e suportar deep links sem código extra.

---

## Estrutura

```
lib/core/navigation/
├── app_routes.dart              # nomes e paths
├── app_router.dart              # GoRouter: rotas, redirect, observers
├── app_navigator.dart           # navegação sem context
└── analytics_route_observer.dart
```

---

## Nomes e paths

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
  static const orderDetail = ':id'; // filha de /orders → /orders/:id
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
        // Guarda o destino para voltar depois do login
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

`GoRouterRefreshStream` é um `ChangeNotifier` simples que chama `notifyListeners()` a cada evento do stream. Assim, o `redirect` roda de novo quando a sessão muda (login, logout ou sessão expirada).

### Regras

- ✅ Toda `GoRoute` tem `name`: navegação, analytics e deep link dependem dele.
- ✅ Rotas filhas para hierarquia real (`/orders/:id`). O botão voltar e o deep link funcionam sem esforço.
- ✅ Proteção de login no `redirect`, nunca com `if (!logado) Navigator.push(login)` dentro da tela.
- ✅ O `BlocProvider` da tela fica na rota (ver [02](02-gerenciamento-estado.md#escopo-dos-providers)).
- ✅ `errorBuilder` com uma tela de "não encontrado" amigável.
- ❌ `Navigator.push(MaterialPageRoute(...))` direto. Toda tela navegável é uma rota.

---

## Abas com estado preservado

Para navegação inferior (bottom bar), use `StatefulShellRoute.indexedStack`: cada aba tem sua pilha e mantém o estado ao trocar.

```dart
StatefulShellRoute.indexedStack(
  builder: (context, state, shell) => MainShell(navigationShell: shell),
  branches: [
    StatefulShellBranch(routes: [/* rotas da aba Início */]),
    StatefulShellBranch(routes: [/* rotas da aba Pedidos */]),
    StatefulShellBranch(routes: [/* rotas da aba Perfil */]),
  ],
)
```

---

## Navegando a partir da UI

```dart
// Trocar de contexto (substitui a pilha até o ancestral comum)
context.goNamed(AppRoutes.orders);

// Empilhar e permitir voltar
context.pushNamed(
  AppRoutes.orderDetail,
  pathParameters: {'id': order.id},
);

// Esperar um resultado
final selected = await context.pushNamed<Address>(AppRoutes.addressPicker);
if (selected != null) cubit.setAddress(selected);
```

| Use | Quando |
|---|---|
| `go` / `goNamed` | Ir para outra área do app: menu, aba, após login, deep link |
| `push` / `pushNamed` | Abrir um detalhe ou fluxo do qual o usuário volta |
| `pushReplacement` | Trocar a tela atual (ex.: passo concluído de um fluxo) |
| `pop(result)` | Voltar devolvendo um valor |

### Parâmetros

- ✅ Dados obrigatórios para abrir a tela vão no **path** (`/orders/:id`) ou na **query** (`?tab=history`). São os únicos que sobrevivem a deep link, push e restauração de estado.
- ✅ `extra` apenas como otimização opcional (ex.: passar o `Order` já carregado para exibir antes do fetch). A tela precisa funcionar sem ele.
- ❌ `extra` como única forma de receber dados. A tela quebra ao abrir por link.

---

## Navegando sem context

Push notifications, o evento de sessão expirada e outros serviços não têm `BuildContext`. Use o `AppNavigator`, que encapsula o router:

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

- ✅ Criado no `bootstrap.dart` e passado pelo construtor só a serviços (push, deep link).
- ❌ Usar `AppNavigator` em widgets. Na UI, use `context.goNamed`.
- ❌ Cubit navegando, mesmo via `AppNavigator`. O Cubit emite estado; a Screen navega no `BlocListener`.

---

## Deep links

Com rotas nomeadas e parâmetros no path, deep links funcionam com o próprio GoRouter, seguindo a [skill oficial de rotas declarativas](https://github.com/flutter/agent-plugins/tree/main/skills/flutter-setup-declarative-routing):

**Android**
1. `intent-filter` com `android:autoVerify="true"` na `MainActivity` (`AndroidManifest.xml`):

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="app.exemplo.gov.br" />
</intent-filter>
```

2. `assetlinks.json` publicado em `https://app.exemplo.gov.br/.well-known/assetlinks.json`, com o SHA-256 do certificado de **cada** flavor que deve abrir links.

**iOS**
1. Ative o handler de deep link do Flutter no `Info.plist`:

```xml
<key>FlutterDeepLinkingEnabled</key>
<true/>
```

Se o projeto usar um plugin de deep link próprio (ex.: `app_links`), deixe essa flag como `false` para não haver conflito.

2. Associated Domains no `Runner.entitlements` (`applinks:app.exemplo.gov.br`).
3. `apple-app-site-association` (sem extensão) em `https://app.exemplo.gov.br/.well-known/`.

**Mapeamento:** a URL pública aponta para o mesmo path interno: `https://app.exemplo.gov.br/orders/123` → `/orders/123`. Se forem diferentes, trate a conversão no `redirect` de topo, não em um serviço paralelo.

**Teste:**

```bash
adb shell 'am start -a android.intent.action.VIEW -c android.intent.category.BROWSABLE -d "https://app.exemplo.gov.br/orders/123"' br.com.exemplo.app.dev
```

```bash
xcrun simctl openurl booted "https://app.exemplo.gov.br/orders/123"
```
