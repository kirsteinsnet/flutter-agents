# 02 · State management

## Goal

Standardize how each screen's state is modeled, changed and consumed, using `flutter_bloc` in Cubit mode.

## The Cubit is the ViewModel

The [official guide](https://docs.flutter.dev/app-architecture/guide) uses MVVM and treats the state tool as a matter of preference. In this guide, the **Cubit plays the ViewModel role**:

| Official concept (MVVM) | Here |
|---|---|
| ViewModel (`ChangeNotifier`) | `XxxCubit`, in `ui/view_models/` |
| UI state exposed by the ViewModel | Immutable `XxxState` |
| Commands | Public Cubit methods (`load`, `refresh`, `submit`) |
| `ListenableBuilder` | `BlocBuilder` / `BlocSelector` / `BlocListener` |
| 1 View : 1 ViewModel relationship | 1 `XxxScreen` : 1 `XxxCubit` |

The command state (running, completed, failed), which the official guide keeps in the `Command` class, lives here in the State itself (`status`, `isRefreshing`, `isSubmitting`).

---

## Cubit or Bloc?

| Use | When |
|---|---|
| **Cubit** (default) | Almost always. Public methods (`load()`, `refresh()`, `select(id)`) call `emit`. |
| **Bloc** (with events) | When you need to transform the event stream: search *debounce*, cancel the previous request (`restartable`), queue (`sequential`). |

If you choose `Bloc`, leave a comment at the top of the class explaining which transformation justified the choice.

---

## Modeling the State

### Default: single class with `status`

```dart
// features/orders/ui/view_models/orders_state.dart
enum OrdersStatus { initial, loading, success, failure }

final class OrdersState extends Equatable {
  const OrdersState({
    this.status = OrdersStatus.initial,
    this.orders = const [],
    this.failure,
    this.isRefreshing = false,
  });

  final OrdersStatus status;
  final List<Order> orders;
  final Failure? failure;
  final bool isRefreshing;

  bool get isEmpty => status == OrdersStatus.success && orders.isEmpty;

  OrdersState copyWith({
    OrdersStatus? status,
    List<Order>? orders,
    Failure? failure,
    bool clearFailure = false,
    bool? isRefreshing,
  }) {
    return OrdersState(
      status: status ?? this.status,
      orders: orders ?? this.orders,
      failure: clearFailure ? null : (failure ?? this.failure),
      isRefreshing: isRefreshing ?? this.isRefreshing,
    );
  }

  @override
  List<Object?> get props => [status, orders, failure, isRefreshing];
}
```

### Why

With subclasses (`OrdersLoading`, `OrdersLoaded`, `OrdersError`), switching state **discards the data**. Reloading the list wipes the list from the screen, and an error in "load more" makes the already loaded items disappear. With a single class, the `status` changes and the data stays there.

### When to use `sealed class`

Use `sealed` subclasses when the states are **truly disjoint** and carry different data, such as the steps of a wizard (`StepPersonalData`, `StepAddress`, `StepConfirmation`). Dart's exhaustive `switch` guarantees the UI handles all of them.

### Rules

- ✅ All fields `final`; the State is immutable.
- ✅ `Equatable` (or manual `==`/`hashCode`) on every State and every model that goes into the State. Without it, `BlocBuilder` cannot tell that nothing changed and redraws for nothing.
- ✅ Manual `copyWith`. For nullable fields that need to go back to `null`, use an explicit `clearX` parameter.
- ✅ Default values in the constructor: the initial state is `const OrdersState()`.
- ✅ Derived getters (`isEmpty`, `canSubmit`) in the State, not in the UI.
- ❌ Exposed mutable lists: never do `state.orders.add(...)`. Create a new list.
- ❌ `String errorMessage` in the State. Use `Failure` (see [05](05-errors-crashlytics.md)).

---

## Anatomy of the Cubit

```dart
// features/orders/ui/view_models/orders_cubit.dart
class OrdersCubit extends Cubit<OrdersState> with ErrorHandlerMixin {
  OrdersCubit({
    required OrdersRepository repository,
    required OrdersAnalytics analytics,
    required this.crashReporter,
  })  : _repository = repository,
        _analytics = analytics,
        super(const OrdersState());

  final OrdersRepository _repository;
  final OrdersAnalytics _analytics;

  @override
  final CrashReporter crashReporter;

  Future<void> load() async {
    emit(state.copyWith(status: OrdersStatus.loading, clearFailure: true));

    await handle(
      operation: 'orders.load',
      action: () async {
        final orders = await _repository.fetchOrders();
        emit(state.copyWith(status: OrdersStatus.success, orders: orders));
        _analytics.listLoaded(count: orders.length);
      },
      onFailure: (failure) => emit(
        state.copyWith(status: OrdersStatus.failure, failure: failure),
      ),
    );
  }

  Future<void> refresh() async {
    emit(state.copyWith(isRefreshing: true));
    await handle(
      operation: 'orders.refresh',
      action: () async {
        final orders = await _repository.fetchOrders();
        emit(state.copyWith(orders: orders, isRefreshing: false));
      },
      // On refresh, keep the list and only signal the failure
      onFailure: (failure) =>
          emit(state.copyWith(isRefreshing: false, failure: failure)),
    );
  }
}
```

### Rules

- ✅ Dependencies through the constructor, in private fields (see [01](01-architecture.md#dependency-injection)).
- ✅ Every async operation goes through `handle(...)` from `ErrorHandlerMixin`. It converts exceptions into `Failure`, reports to Crashlytics and ignores `emit` after `close()`.
- ✅ Public methods named by intent: `load`, `refresh`, `submit`, `toggleFavorite`.
- ✅ Analytics fired by the Cubit when the event depends on the result (success/failure). Pure click events may come from the Screen.
- ❌ `import 'package:flutter/material.dart'` in the Cubit. If you need it, something UI-related is in the wrong place.
- ❌ `BuildContext` as a parameter, `Navigator`, `showDialog` or toast inside the Cubit.
- ❌ Manual `try/catch` scattered around. Use `handle`.
- ❌ Display formatting logic (dates, currency). That belongs in extensions/formatters used by the UI.

---

## Consuming in the UI

### Separate drawing from side effects

```dart
class OrdersScreen extends StatelessWidget {
  const OrdersScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocListener<OrdersCubit, OrdersState>(
      // Side effects: run once per change
      listenWhen: (prev, curr) =>
          prev.failure != curr.failure &&
          curr.failure != null &&
          curr.status == OrdersStatus.success,
      listener: (context, state) =>
          AppToast.error(context, state.failure!.userMessage),
      child: Scaffold(
        appBar: AppBar(title: const Text('Orders')),
        body: BlocBuilder<OrdersCubit, OrdersState>(
          builder: (context, state) => switch (state.status) {
            OrdersStatus.initial ||
            OrdersStatus.loading => const OrdersSkeleton(),
            OrdersStatus.failure => ErrorView(
                failure: state.failure!,
                onRetry: context.read<OrdersCubit>().load,
              ),
            OrdersStatus.success when state.isEmpty =>
              const EmptyView(message: "You don't have any orders yet."),
            OrdersStatus.success => OrdersList(
                orders: state.orders,
                onRefresh: context.read<OrdersCubit>().refresh,
              ),
          },
        ),
      ),
    );
  }
}
```

### Rules

- ✅ `BlocBuilder` only draws. Toasts, dialogs and navigation go in `BlocListener`.
- ✅ `listenWhen` / `buildWhen` to avoid repeated side effects and unnecessary rebuilds.
- ✅ `BlocSelector` when a widget depends on a single State field.
- ✅ `context.read<C>()` to call methods; `context.watch<C>()` / `BlocBuilder` to react to changes.
- ❌ `context.watch` inside callbacks (`onPressed`).
- ❌ Calling `cubit.load()` inside `build`. Trigger it on creation (`create: (context) => OrdersCubit(...)..load()`) or in `initState`.

---

## Provider scope

| Scope | Where the `BlocProvider` lives | Example |
|---|---|---|
| Screen | In the route (`GoRoute.builder`) | `OrdersCubit` |
| Multi-screen flow | In the `ShellRoute` that groups the flow | Cubit for a 3-step checkout |
| Whole app | Above `MaterialApp.router` | `SessionCubit`, `ThemeCubit` |

- ✅ `BlocProvider` creates **and closes** the Cubit. Use `BlocProvider(create: ...)`.
- ❌ `BlocProvider.value` with a Cubit created right there. `.value` is only for passing along a Cubit that already exists, and in that case it is not closed.
- ❌ Global state for data from a single screen. If nobody outside the flow uses it, it is not global.

---

## Testing

Following the [official recommendation](https://docs.flutter.dev/app-architecture/recommendations#testing), the default is **fakes**: simple, working implementations of the interface, reusable across tests, in `testing/fakes/`.

```dart
// testing/fakes/fake_orders_repository.dart
class FakeOrdersRepository implements OrdersRepository {
  List<Order> orders = [];
  Failure? failure;

  @override
  Future<List<Order>> fetchOrders({int page = 1}) async {
    if (failure case final f?) throw f;
    return orders;
  }

  @override
  Future<Order> fetchOrder(String id) async =>
      orders.firstWhere((o) => o.id == id, orElse: () => throw const NotFoundFailure());

  @override
  Future<Order> createOrder(CreateOrderRequest request) async =>
      throw UnimplementedError();
}
```

Every Cubit has a `bloc_test` test covering at least the success path and the failure path:

```dart
late FakeOrdersRepository repository;

setUp(() => repository = FakeOrdersRepository());

OrdersCubit buildCubit() => OrdersCubit(
      repository: repository,
      analytics: OrdersAnalytics(FakeAnalyticsService()),
      crashReporter: FakeCrashReporter(),
    );

blocTest<OrdersCubit, OrdersState>(
  'emits loading and then success with the orders',
  setUp: () => repository.orders = [fakeOrder],
  build: buildCubit,
  act: (cubit) => cubit.load(),
  expect: () => [
    const OrdersState(status: OrdersStatus.loading),
    OrdersState(status: OrdersStatus.success, orders: [fakeOrder]),
  ],
);

blocTest<OrdersCubit, OrdersState>(
  'emits failure when the repository fails',
  setUp: () => repository.failure = const NetworkFailure(),
  build: buildCubit,
  act: (cubit) => cubit.load(),
  expect: () => [
    const OrdersState(status: OrdersStatus.loading),
    const OrdersState(status: OrdersStatus.failure, failure: NetworkFailure()),
  ],
);
```

Use **mocks** (`mockito` with `@GenerateNiceMocks`, as the [official Dart skill](https://github.com/dart-lang/skills) recommends) only when the goal of the test is to verify an **interaction**, for example "the analytics event was fired once":

```dart
@GenerateNiceMocks([MockSpec<AnalyticsService>()])
import 'orders_cubit_test.mocks.dart';

// ...
verify(analytics.logEvent('orders_list_loaded', any)).called(1);
```

- ✅ When stubbing async methods, always `thenAnswer((_) async => value)`, never `thenReturn`.
- ✅ Infrastructure fakes (`FakeAnalyticsService`, `FakeCrashReporter`) also live in `testing/fakes/` and record the calls they receive in a list, which usually makes the mock unnecessary.
