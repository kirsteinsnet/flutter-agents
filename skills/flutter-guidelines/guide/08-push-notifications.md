# 08 · Push notifications

## Objetivo

Receber notificações via Firebase Cloud Messaging (FCM) nos três estados do app, exibir corretamente em primeiro plano e levar o usuário à tela certa ao tocar, com toda a lógica de roteamento em um único lugar.

---

## Peças

```
lib/core/push/
├── push_service.dart          # permissão, token, listeners do FCM
├── push_router.dart           # payload → rota
├── local_notifications.dart   # exibição em foreground + canal Android
└── push_background.dart       # handler top-level de background
```

| Peça | Responsabilidade |
|---|---|
| `PushService` | Pedir permissão, obter e renovar o token, registrar os listeners, enviar o token ao backend |
| `LocalNotifications` | Mostrar a notificação quando o app está aberto; criar o canal Android |
| `PushRouter` | Converter o payload em navegação. **Único** lugar que sabe disso |
| Handler de background | Processar mensagens *data-only* com o app em segundo plano/fechado |

---

## Os três estados do app

| Estado | O sistema mostra a notificação? | Como o app fica sabendo do toque |
|---|---|---|
| **Foreground** (aberto e visível) | ❌ Não (no Android; no iOS depende da configuração) | `FirebaseMessaging.onMessage` → exibir com `flutter_local_notifications` → callback de toque local |
| **Background** (aberto, mas em segundo plano) | ✅ Sim | `FirebaseMessaging.onMessageOpenedApp` |
| **Terminated** (fechado) | ✅ Sim | `FirebaseMessaging.instance.getInitialMessage()` na inicialização |

Esquecer qualquer um dos três é a causa mais comum de "a notificação não abre a tela certa".

---

## Payload padronizado

Combine com o backend um formato único de `data`:

```json
{
  "notification": { "title": "Pedido atualizado", "body": "Seu pedido #123 foi concluído." },
  "data": {
    "route": "/orders/123",
    "notification_id": "a1b2c3"
  }
}
```

- ✅ `route` com um **path interno** do app (o mesmo do GoRouter). Assim, push e deep link usam a mesma tabela de rotas.
- ✅ Se o backend não puder mandar paths, use `type` + `id` (`{"type": "order", "id": "123"}`) e faça a conversão **só** no `PushRouter`.
- ✅ `notification_id` para marcar como lida e para analytics.
- ❌ Dados pessoais no payload: ele passa por servidores de terceiros e aparece em logs do sistema.

---

## PushRouter

```dart
// lib/core/push/push_router.dart
class PushRouter {
  PushRouter({required AppNavigator navigator, required AnalyticsService analytics})
      : _navigator = navigator,
        _analytics = analytics;

  final AppNavigator _navigator;
  final AnalyticsService _analytics;

  void open(Map<String, dynamic> data) {
    final location = _resolve(data);
    _analytics.logEvent('push_open', {
      'notification_id': data['notification_id'],
      'has_route': location != null,
    });
    if (location != null) _navigator.go(location);
  }

  String? _resolve(Map<String, dynamic> data) {
    final route = data['route'];
    if (route is String && route.startsWith('/')) return route;

    // Formato alternativo type + id
    return switch ((data['type'], data['id'])) {
      ('order', final String id) => '/orders/$id',
      ('profile', _) => '/profile',
      _ => null,
    };
  }
}
```

Rotas protegidas continuam protegidas: se o usuário não estiver logado, o `redirect` do router (ver [07](07-navegacao.md)) leva para o login e volta ao destino depois.

---

## PushService

```dart
// lib/core/push/push_service.dart
class PushService {
  PushService({
    required PushRouter router,
    required LocalNotifications local,
    required PushTokenRepository tokenRepository,
    FirebaseMessaging? messaging,
  })  : _router = router,
        _local = local,
        _tokenRepository = tokenRepository,
        _messaging = messaging ?? FirebaseMessaging.instance;

  final PushRouter _router;
  final LocalNotifications _local;
  final PushTokenRepository _tokenRepository;
  final FirebaseMessaging _messaging;

  /// Chamar depois do primeiro frame (ver bootstrap em 01-arquitetura.md).
  Future<void> initialize() async {
    await _local.initialize(onTap: _router.open);

    // 1. App foi aberto por uma notificação com ele fechado
    final initial = await _messaging.getInitialMessage();
    if (initial != null) _router.open(initial.data);

    // 2. Toque com o app em background
    FirebaseMessaging.onMessageOpenedApp.listen((m) => _router.open(m.data));

    // 3. Mensagem com o app em foreground: o sistema não mostra, nós mostramos
    FirebaseMessaging.onMessage.listen(_local.show);

    // Token
    _messaging.onTokenRefresh.listen(_tokenRepository.register);
  }

  /// Pedir permissão em um momento com contexto (ex.: após o onboarding,
  /// ou quando o usuário ativa "receber avisos"), nunca na primeira abertura.
  Future<bool> requestPermission() async {
    final settings = await _messaging.requestPermission();
    final granted = settings.authorizationStatus == AuthorizationStatus.authorized ||
        settings.authorizationStatus == AuthorizationStatus.provisional;
    if (granted) await _registerToken();
    return granted;
  }

  Future<void> _registerToken() async {
    if (Platform.isIOS) {
      // No iOS o token FCM só existe depois do token APNs
      for (var i = 0; i < 10 && await _messaging.getAPNSToken() == null; i++) {
        await Future<void>.delayed(const Duration(milliseconds: 500));
      }
    }
    final token = await _messaging.getToken();
    if (token != null) await _tokenRepository.register(token);
  }

  /// No logout: desvincula o token do usuário no backend.
  Future<void> unregister() async {
    final token = await _messaging.getToken();
    if (token != null) await _tokenRepository.unregister(token);
  }
}
```

### Regras

- ✅ `initialize()` depois do primeiro frame: o router já está montado quando `getInitialMessage` navegar.
- ✅ Token enviado ao backend no login, no `onTokenRefresh` e removido no logout.
- ✅ Permissão pedida com contexto, explicando o benefício antes (tela própria ou bottom sheet).
- ❌ Navegar ou decidir rota dentro do `PushService` ou do handler de foreground. Tudo passa pelo `PushRouter`.

---

## Notificações locais (foreground)

As assinaturas de `flutter_local_notifications` mudam entre versões maiores. Confira a versão usada no projeto ao adaptar o código.

```dart
// lib/core/push/local_notifications.dart
class LocalNotifications {
  final _plugin = FlutterLocalNotificationsPlugin();

  static const _channel = AndroidNotificationChannel(
    'default_channel',
    'Avisos',
    description: 'Avisos gerais do aplicativo',
    importance: Importance.high,
  );

  Future<void> initialize({required void Function(Map<String, dynamic>) onTap}) async {
    await _plugin.initialize(
      const InitializationSettings(
        android: AndroidInitializationSettings('@drawable/ic_notification'),
        iOS: DarwinInitializationSettings(
          requestAlertPermission: false, // a permissão é pedida pelo PushService
          requestBadgePermission: false,
          requestSoundPermission: false,
        ),
      ),
      onDidReceiveNotificationResponse: (response) {
        final payload = response.payload;
        if (payload != null) onTap(jsonDecode(payload) as Map<String, dynamic>);
      },
    );

    await _plugin
        .resolvePlatformSpecificImplementation<AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(_channel);
  }

  Future<void> show(RemoteMessage message) async {
    final n = message.notification;
    if (n == null) return;
    await _plugin.show(
      message.hashCode,
      n.title,
      n.body,
      NotificationDetails(
        android: AndroidNotificationDetails(
          _channel.id,
          _channel.name,
          channelDescription: _channel.description,
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: const DarwinNotificationDetails(),
      ),
      payload: jsonEncode(message.data),
    );
  }
}
```

- ✅ Canal Android criado na inicialização e declarado no `AndroidManifest.xml` como padrão (`com.google.firebase.messaging.default_notification_channel_id`).
- ✅ Ícone de notificação monocromático (branco com transparência) em `res/drawable`.

---

## Handler de background

Só necessário para mensagens **data-only** que precisam de processamento (atualizar badge, salvar algo localmente). Mensagens com `notification` já são exibidas pelo sistema.

```dart
// lib/core/push/push_background.dart
@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  // Roda em um isolate separado: nenhum objeto criado no bootstrap está disponível.
  // Sem options, o Firebase usa o google-services.json / GoogleService-Info.plist
  // do flavor que foi compilado.
  await Firebase.initializeApp();
  // Processamento leve e rápido; sem UI, sem navegação.
}
```

```dart
// bootstrap.dart, antes do runApp
FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler);
```

- ✅ Função **top-level** (fora de classe) com `@pragma('vm:entry-point')`. Sem isso, o tree-shaking remove o handler no build de release.
- ❌ Acessar objetos do app (repositories, Cubits, navegação) dentro dele.

---

## Configuração nativa (checklist)

**Android**
- [ ] `google-services.json` por flavor (ver [09](09-flavors-ambientes.md))
- [ ] Permissão `POST_NOTIFICATIONS` (Android 13+) no manifest
- [ ] Canal padrão e ícone declarados no manifest

**iOS**
- [ ] Capability **Push Notifications** e **Background Modes → Remote notifications**
- [ ] Chave APNs (`.p8`) enviada ao projeto Firebase de cada ambiente
- [ ] `GoogleService-Info.plist` por flavor

**Teste**
- [ ] Foreground, background e terminated, em Android e iOS
- [ ] Toque com usuário deslogado em rota protegida (deve passar pelo login e voltar)
- [ ] Payload com rota inexistente (deve abrir o app sem quebrar)
