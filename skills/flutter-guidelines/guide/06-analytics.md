# 06 · Analytics

## Objetivo

Registrar o uso do app de forma consistente, com nomes previsíveis, sem código repetido em cada feature e sem dados pessoais.

---

## Estrutura

```mermaid
flowchart LR
    Screen -->|cliques| FA["OrdersAnalytics\n(nomes + parâmetros)"]
    Cubit -->|resultados| FA
    FA --> AS["AnalyticsService\n(interface)"]
    AS --> FB["FirebaseAnalyticsService"]
    Router["Observer do GoRouter"] -->|screen_view| AS
    Session["SessionCubit"] -->|setUserId| AS
```

| Peça | Responsabilidade |
|---|---|
| `AnalyticsService` | Única porta para o provedor. Verifica se está habilitado, normaliza parâmetros, engole os próprios erros. |
| `<Feature>Analytics` | Catálogo de eventos de uma feature: só monta nome e parâmetros. |
| Observer de navegação | `screen_view` automático. |
| `SessionCubit` | `setUserId` no login e logout. |

---

## AnalyticsService

```dart
// lib/core/analytics/analytics_service.dart
abstract interface class AnalyticsService {
  Future<void> logEvent(String name, [Map<String, Object?> params = const {}]);
  Future<void> logScreenView(String screenName);
  Future<void> setUserId(String? id);
  Future<void> setUserProperty(String name, String? value);
}

class FirebaseAnalyticsService implements AnalyticsService {
  FirebaseAnalyticsService([FirebaseAnalytics? analytics])
      : _analytics = analytics ?? FirebaseAnalytics.instance;

  final FirebaseAnalytics _analytics;

  static final _validName = RegExp(r'^[a-z][a-z0-9_]{0,39}$');

  @override
  Future<void> logEvent(String name, [Map<String, Object?> params = const {}]) async {
    assert(_validName.hasMatch(name), 'Nome de evento inválido: $name');
    try {
      await _analytics.logEvent(name: name, parameters: _normalize(params));
      if (kDebugMode) debugPrint('📊 $name $params');
    } catch (e) {
      if (kDebugMode) debugPrint('⚠️ analytics falhou em $name: $e');
    }
  }

  @override
  Future<void> logScreenView(String screenName) =>
      _safe(() => _analytics.logScreenView(screenName: screenName));

  @override
  Future<void> setUserId(String? id) => _safe(() => _analytics.setUserId(id: id));

  @override
  Future<void> setUserProperty(String name, String? value) =>
      _safe(() => _analytics.setUserProperty(name: name, value: value));

  /// Firebase aceita só String e num; remove nulos, converte bool e corta textos.
  Map<String, Object> _normalize(Map<String, Object?> params) => {
        for (final e in params.entries.take(25))
          if (e.value != null)
            e.key: switch (e.value) {
              final bool b => b ? 'true' : 'false',
              final num n => n,
              final Object o => _truncate(o.toString()),
            },
      };

  String _truncate(String v) => v.length > 100 ? v.substring(0, 100) : v;

  Future<void> _safe(Future<void> Function() call) async {
    try {
      await call();
    } catch (_) {}
  }
}
```

Em ambientes com analytics desligado (`analyticsEnabled: false` no `AppConfig`), o `bootstrap.dart` usa um `NoopAnalyticsService`. Nos testes, use um `FakeAnalyticsService` (em `testing/fakes/`) que guarda os eventos numa lista.

```dart
class NoopAnalyticsService implements AnalyticsService {
  const NoopAnalyticsService();
  @override
  Future<void> logEvent(String name, [Map<String, Object?> params = const {}]) async {}
  @override
  Future<void> logScreenView(String screenName) async {}
  @override
  Future<void> setUserId(String? id) async {}
  @override
  Future<void> setUserProperty(String name, String? value) async {}
}
```

---

## Eventos por feature

```dart
// features/orders/analytics/orders_analytics.dart
class OrdersAnalytics {
  const OrdersAnalytics(this._analytics);
  final AnalyticsService _analytics;

  void listLoaded({required int count}) =>
      _analytics.logEvent('orders_list_loaded', {'count': count});

  void itemTap({required String orderId, required int position}) =>
      _analytics.logEvent('orders_item_tap', {
        'order_id': orderId,
        'position': position,
      });

  void createSubmit({required bool success, String? failureType}) =>
      _analytics.logEvent('orders_create_submit', {
        'success': success,
        'failure_type': failureType,
      });
}
```

### Regras

- ✅ Uma classe por feature, sem estado e sem singleton próprio, criada na rota junto com o Cubit (`OrdersAnalytics(context.read())`) e recebida pelo construtor.
- ✅ Métodos com parâmetros nomeados e tipados: quem chama não monta `Map` à mão.
- ✅ Não espere (`await`) o analytics em fluxo de UI: dispare e siga.
- ❌ Chamar `FirebaseAnalytics.instance` fora do `FirebaseAnalyticsService`.
- ❌ `try/catch` e checagem de "está inicializado?" nas classes de feature. Isso é responsabilidade do serviço central.
- ❌ Adicionar `user_id` em cada evento. Ele é definido uma vez com `setUserId`.

---

## Nomenclatura

**Evento:** `<feature>_<objeto>_<ação>`, em `snake_case`, inglês ou português (escolha um por projeto e não misture).

| ✅ Bom | ❌ Ruim | Problema |
|---|---|---|
| `orders_item_tap` | `clickOrder` | camelCase, sem feature |
| `orders_filter_apply` | `orders_filter` | sem ação |
| `profile_photo_update_success` | `profile_success` | não diz o que deu certo |
| `orders_create_submit` + `success: false` | `orders_create_error_404` | detalhe técnico no nome; vai para parâmetro |

**Ações comuns:** `view`, `tap`, `submit`, `success`, `failure`, `open`, `close`, `share`, `search`, `filter_apply`, `loaded`.

**Limites do Firebase Analytics:**

| Item | Limite |
|---|---|
| Nome do evento | 40 caracteres, começa com letra, só `[a-z0-9_]` |
| Parâmetros por evento | 25 |
| Nome do parâmetro | 40 caracteres |
| Valor de texto | 100 caracteres |
| Nomes reservados | Não use prefixos `firebase_`, `google_`, `ga_` nem eventos reservados (`session_start`, `app_open`...) |

---

## Screen view automático

O observer da navegação registra `screen_view` com o **nome da rota**, não o path (para não gerar uma tela diferente por ID).

```dart
// lib/core/navigation/analytics_route_observer.dart
class AnalyticsRouteObserver extends NavigatorObserver {
  AnalyticsRouteObserver(this._analytics);
  final AnalyticsService _analytics;

  @override
  void didPush(Route<dynamic> route, Route<dynamic>? previousRoute) => _log(route);

  @override
  void didReplace({Route<dynamic>? newRoute, Route<dynamic>? oldRoute}) {
    if (newRoute != null) _log(newRoute);
  }

  @override
  void didPop(Route<dynamic> route, Route<dynamic>? previousRoute) {
    if (previousRoute != null) _log(previousRoute);
  }

  void _log(Route<dynamic> route) {
    final name = route.settings.name;
    if (name != null && route is PageRoute) _analytics.logScreenView(name);
  }
}
```

Registre em `GoRouter(observers: [AnalyticsRouteObserver(analytics)])`, no `createRouter` (ver [07](07-navegacao.md#router)), e garanta que toda `GoRoute` tenha `name` (ver [07](07-navegacao.md)).

Os `observers` do `GoRouter` só enxergam o navigator raiz. Rotas dentro de `ShellRoute`/`StatefulShellBranch` precisam do observer também no parâmetro `observers` do shell ou do branch.

---

## Privacidade

As mesmas regras do Crashlytics valem aqui (ver [05](05-erros-crashlytics.md#privacidade-lgpd)): nada de nome, e-mail, documento, telefone, endereço ou texto livre digitado em parâmetros. Para buscas, registre `query_length` ou a categoria, não o termo.

---

## Catálogo de eventos

Cada projeto mantém `docs/analytics-events.md` com uma tabela por feature:

| Evento | Quando dispara | Parâmetros | Origem |
|---|---|---|---|
| `orders_list_loaded` | Lista carregada com sucesso | `count` (int) | `OrdersCubit.load` |
| `orders_item_tap` | Toque em um pedido | `order_id` (string), `position` (int) | `OrdersScreen` |

Evento novo ou alterado só entra com a tabela atualizada no mesmo merge request.
