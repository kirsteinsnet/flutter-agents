# 08 · Push notifications

## Goal

Receive notifications via Firebase Cloud Messaging (FCM) in all three app states, display them correctly in the foreground and take the user to the right screen on tap, with all routing logic in a single place.

---

## Pieces

```
lib/core/push/
├── push_service.dart          # permission, token, FCM listeners
├── push_router.dart           # payload → route
├── local_notifications.dart   # foreground display + Android channel
└── push_background.dart       # top-level background handler
```

| Piece | Responsibility |
|---|---|
| `PushService` | Request permission, get and refresh the token, register the listeners, send the token to the backend |
| `LocalNotifications` | Show the notification when the app is open; create the Android channel |
| `PushRouter` | Convert the payload into navigation. The **only** place that knows about this |
| Background handler | Process *data-only* messages with the app in the background/closed |

---

## The three app states

| State | Does the system show the notification? | How the app learns about the tap |
|---|---|---|
| **Foreground** (open and visible) | ❌ No (on Android; on iOS it depends on configuration) | `FirebaseMessaging.onMessage` → display with `flutter_local_notifications` → local tap callback |
| **Background** (open, but in the background) | ✅ Yes | `FirebaseMessaging.onMessageOpenedApp` |
| **Terminated** (closed) | ✅ Yes | `FirebaseMessaging.instance.getInitialMessage()` at startup |

Forgetting any of the three is the most common cause of "the notification doesn't open the right screen".

---

## Standardized payload

Agree on a single `data` format with the backend:

```json
{
  "notification": { "title": "Order updated", "body": "Your order #123 has been completed." },
  "data": {
    "route": "/orders/123",
    "notification_id": "a1b2c3"
  }
}
```

- ✅ `route` with an **internal path** of the app (the same as in GoRouter). This way, push and deep links use the same route table.
- ✅ If the backend can't send paths, use `type` + `id` (`{"type": "order", "id": "123"}`) and do the conversion **only** in `PushRouter`.
- ✅ `notification_id` to mark as read and for analytics.
- ❌ Personal data in the payload: it goes through third-party servers and shows up in system logs.

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

    // Alternative type + id format
    return switch ((data['type'], data['id'])) {
      ('order', final String id) => '/orders/$id',
      ('profile', _) => '/profile',
      _ => null,
    };
  }
}
```

Protected routes stay protected: if the user isn't logged in, the router's `redirect` (see [07](07-navigation.md)) takes them to login and back to the destination afterwards.

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

  /// Call after the first frame (see bootstrap in 01-architecture.md).
  Future<void> initialize() async {
    await _local.initialize(onTap: _router.open);

    // 1. App was opened by a notification while closed
    final initial = await _messaging.getInitialMessage();
    if (initial != null) _router.open(initial.data);

    // 2. Tap with the app in the background
    FirebaseMessaging.onMessageOpenedApp.listen((m) => _router.open(m.data));

    // 3. Message with the app in the foreground: the system doesn't show it, we do
    FirebaseMessaging.onMessage.listen(_local.show);

    // Token
    _messaging.onTokenRefresh.listen(_tokenRepository.register);
  }

  /// Request permission at a moment with context (e.g. after onboarding,
  /// or when the user turns on "receive alerts"), never on first launch.
  Future<bool> requestPermission() async {
    final settings = await _messaging.requestPermission();
    final granted = settings.authorizationStatus == AuthorizationStatus.authorized ||
        settings.authorizationStatus == AuthorizationStatus.provisional;
    if (granted) await _registerToken();
    return granted;
  }

  Future<void> _registerToken() async {
    if (Platform.isIOS) {
      // On iOS the FCM token only exists after the APNs token
      for (var i = 0; i < 10 && await _messaging.getAPNSToken() == null; i++) {
        await Future<void>.delayed(const Duration(milliseconds: 500));
      }
    }
    final token = await _messaging.getToken();
    if (token != null) await _tokenRepository.register(token);
  }

  /// On logout: unlinks the token from the user on the backend.
  Future<void> unregister() async {
    final token = await _messaging.getToken();
    if (token != null) await _tokenRepository.unregister(token);
  }
}
```

### Rules

- ✅ `initialize()` after the first frame: the router is already built when `getInitialMessage` navigates.
- ✅ Token sent to the backend on login and on `onTokenRefresh`, and removed on logout.
- ✅ Permission requested with context, explaining the benefit first (dedicated screen or bottom sheet).
- ❌ Navigating or deciding the route inside `PushService` or the foreground handler. Everything goes through `PushRouter`.

---

## Local notifications (foreground)

The `flutter_local_notifications` signatures change between major versions. Check the version used in the project when adapting the code.

```dart
// lib/core/push/local_notifications.dart
class LocalNotifications {
  final _plugin = FlutterLocalNotificationsPlugin();

  static const _channel = AndroidNotificationChannel(
    'default_channel',
    'Alerts',
    description: 'General app alerts',
    importance: Importance.high,
  );

  Future<void> initialize({required void Function(Map<String, dynamic>) onTap}) async {
    await _plugin.initialize(
      const InitializationSettings(
        android: AndroidInitializationSettings('@drawable/ic_notification'),
        iOS: DarwinInitializationSettings(
          requestAlertPermission: false, // permission is requested by PushService
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

- ✅ Android channel created at startup and declared in `AndroidManifest.xml` as the default (`com.google.firebase.messaging.default_notification_channel_id`).
- ✅ Monochrome notification icon (white with transparency) in `res/drawable`.

---

## Background handler

Only needed for **data-only** messages that require processing (update badge, save something locally). Messages with `notification` are already displayed by the system.

```dart
// lib/core/push/push_background.dart
@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  // Runs in a separate isolate: no object created in bootstrap is available.
  // Without options, Firebase uses the google-services.json / GoogleService-Info.plist
  // of the flavor that was built.
  await Firebase.initializeApp();
  // Light, fast processing; no UI, no navigation.
}
```

```dart
// bootstrap.dart, before runApp
FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler);
```

- ✅ **Top-level** function (outside any class) with `@pragma('vm:entry-point')`. Without it, tree-shaking removes the handler in the release build.
- ❌ Accessing app objects (repositories, Cubits, navigation) inside it.

---

## Native setup (checklist)

**Android**
- [ ] `google-services.json` per flavor (see [09](09-flavors-environments.md))
- [ ] `POST_NOTIFICATIONS` permission (Android 13+) in the manifest
- [ ] Default channel and icon declared in the manifest

**iOS**
- [ ] **Push Notifications** and **Background Modes → Remote notifications** capabilities
- [ ] APNs key (`.p8`) uploaded to each environment's Firebase project
- [ ] `GoogleService-Info.plist` per flavor

**Testing**
- [ ] Foreground, background and terminated, on Android and iOS
- [ ] Tap with a logged-out user on a protected route (must go through login and come back)
- [ ] Payload with a nonexistent route (must open the app without breaking)
