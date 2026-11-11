# 02 · Gerenciamento de estado

## Objetivo

Padronizar como o estado de cada tela é modelado, alterado e consumido, usando `flutter_bloc` no modo Cubit.

## O Cubit é o ViewModel

O [guia oficial](https://docs.flutter.dev/app-architecture/guide) usa MVVM e trata a ferramenta de estado como questão de preferência. Neste guia, o **Cubit cumpre o papel de ViewModel**:

| Conceito oficial (MVVM) | Aqui |
|---|---|
| ViewModel (`ChangeNotifier`) | `XxxCubit`, em `ui/view_models/` |
| UI state exposto pelo ViewModel | `XxxState` imutável |
| Commands | Métodos públicos do Cubit (`load`, `refresh`, `submit`) |
| `ListenableBuilder` | `BlocBuilder` / `BlocSelector` / `BlocListener` |
| Relação 1 View : 1 ViewModel | 1 `XxxScreen` : 1 `XxxCubit` |

O estado do comando (executando, concluído, com erro), que no oficial fica na classe `Command`, aqui fica no próprio State (`status`, `isRefreshing`, `isSubmitting`).

---

## Cubit ou Bloc?

| Use | Quando |
|---|---|
| **Cubit** (padrão) | Quase sempre. Métodos públicos (`load()`, `refresh()`, `select(id)`) chamam `emit`. |
| **Bloc** (com eventos) | Quando precisa transformar o fluxo de eventos: *debounce* de busca, cancelar a requisição anterior (`restartable`), fila (`sequential`). |

Se escolher `Bloc`, deixe um comentário no topo da classe explicando qual transformação justificou a escolha.

---

## Modelando o State

### Padrão: classe única com `status`

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

### Por quê

Com subclasses (`OrdersLoading`, `OrdersLoaded`, `OrdersError`), trocar de estado **descarta os dados**. Recarregar a lista apaga a lista da tela, e um erro no "carregar mais" some com os itens já carregados. Na classe única, o `status` muda e os dados continuam lá.

### Quando usar `sealed class`

Use subclasses `sealed` quando os estados são **realmente disjuntos** e carregam dados diferentes, como as etapas de um wizard (`StepPersonalData`, `StepAddress`, `StepConfirmation`). O `switch` exaustivo do Dart garante que a UI trata todos.

### Regras

- ✅ Todos os campos `final`; o State é imutável.
- ✅ `Equatable` (ou `==`/`hashCode` manuais) em todo State e em todo model que entra no State. Sem isso, o `BlocBuilder` não sabe que nada mudou e redesenha à toa.
- ✅ `copyWith` manual. Para campos anuláveis que precisam voltar a `null`, use um parâmetro `clearX` explícito.
- ✅ Valores padrão no construtor: o estado inicial é `const OrdersState()`.
- ✅ Getters derivados (`isEmpty`, `canSubmit`) no State, não na UI.
- ❌ Listas mutáveis expostas: nunca faça `state.orders.add(...)`. Crie uma lista nova.
- ❌ `String errorMessage` no State. Use `Failure` (ver [05](05-erros-crashlytics.md)).

---

## Anatomia do Cubit

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
      // No refresh, mantém a lista e só sinaliza a falha
      onFailure: (failure) =>
          emit(state.copyWith(isRefreshing: false, failure: failure)),
    );
  }
}
```

### Regras

- ✅ Dependências pelo construtor, em campos privados (ver [01](01-arquitetura.md#injeção-de-dependência)).
- ✅ Toda operação assíncrona passa por `handle(...)` do `ErrorHandlerMixin`. Ele converte exceções em `Failure`, reporta no Crashlytics e ignora `emit` depois de `close()`.
- ✅ Métodos públicos com nome de intenção: `load`, `refresh`, `submit`, `toggleFavorite`.
- ✅ Analytics disparado pelo Cubit quando o evento depende do resultado (sucesso/falha). Eventos de clique puro podem sair da Screen.
- ❌ `import 'package:flutter/material.dart'` no Cubit. Se precisar, há algo de UI no lugar errado.
- ❌ `BuildContext` como parâmetro, `Navigator`, `showDialog` ou toast dentro do Cubit.
- ❌ `try/catch` manual espalhado. Use `handle`.
- ❌ Lógica de formatação para exibição (datas, moeda). Isso fica em extensions/formatters usados pela UI.

---

## Consumindo na UI

### Separar desenho de efeito

```dart
class OrdersScreen extends StatelessWidget {
  const OrdersScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocListener<OrdersCubit, OrdersState>(
      // Efeitos colaterais: rodam uma vez por mudança
      listenWhen: (prev, curr) =>
          prev.failure != curr.failure &&
          curr.failure != null &&
          curr.status == OrdersStatus.success,
      listener: (context, state) =>
          AppToast.error(context, state.failure!.userMessage),
      child: Scaffold(
        appBar: AppBar(title: const Text('Pedidos')),
        body: BlocBuilder<OrdersCubit, OrdersState>(
          builder: (context, state) => switch (state.status) {
            OrdersStatus.initial ||
            OrdersStatus.loading => const OrdersSkeleton(),
            OrdersStatus.failure => ErrorView(
                failure: state.failure!,
                onRetry: context.read<OrdersCubit>().load,
              ),
            OrdersStatus.success when state.isEmpty =>
              const EmptyView(message: 'Você ainda não tem pedidos.'),
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

### Regras

- ✅ `BlocBuilder` só desenha. Toast, diálogo e navegação ficam no `BlocListener`.
- ✅ `listenWhen` / `buildWhen` para evitar efeitos repetidos e rebuilds desnecessários.
- ✅ `BlocSelector` quando um widget depende de um único campo do State.
- ✅ `context.read<C>()` para chamar métodos; `context.watch<C>()` / `BlocBuilder` para reagir a mudanças.
- ❌ `context.watch` dentro de callbacks (`onPressed`).
- ❌ Chamar `cubit.load()` dentro do `build`. Dispare na criação (`create: (context) => OrdersCubit(...)..load()`) ou no `initState`.

---

## Escopo dos providers

| Escopo | Onde fica o `BlocProvider` | Exemplo |
|---|---|---|
| Tela | Na rota (`GoRoute.builder`) | `OrdersCubit` |
| Fluxo de várias telas | No `ShellRoute` que agrupa o fluxo | Cubit de um checkout em 3 passos |
| App inteiro | Acima do `MaterialApp.router` | `SessionCubit`, `ThemeCubit` |

- ✅ O `BlocProvider` cria **e fecha** o Cubit. Use `BlocProvider(create: ...)`.
- ❌ `BlocProvider.value` com um Cubit criado ali mesmo. `.value` serve só para repassar um Cubit que já existe, e nesse caso ele não é fechado.
- ❌ Estado global para dado de uma tela só. Se ninguém fora do fluxo usa, ele não é global.

---

## Testando

Seguindo a [recomendação oficial](https://docs.flutter.dev/app-architecture/recommendations#testing), o padrão são **fakes**: implementações simples e funcionais da interface, reutilizáveis entre testes, em `testing/fakes/`.

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

Todo Cubit tem teste com `bloc_test` cobrindo pelo menos o caminho de sucesso e o de falha:

```dart
late FakeOrdersRepository repository;

setUp(() => repository = FakeOrdersRepository());

OrdersCubit buildCubit() => OrdersCubit(
      repository: repository,
      analytics: OrdersAnalytics(FakeAnalyticsService()),
      crashReporter: FakeCrashReporter(),
    );

blocTest<OrdersCubit, OrdersState>(
  'emite loading e depois success com os pedidos',
  setUp: () => repository.orders = [fakeOrder],
  build: buildCubit,
  act: (cubit) => cubit.load(),
  expect: () => [
    const OrdersState(status: OrdersStatus.loading),
    OrdersState(status: OrdersStatus.success, orders: [fakeOrder]),
  ],
);

blocTest<OrdersCubit, OrdersState>(
  'emite failure quando o repository falha',
  setUp: () => repository.failure = const NetworkFailure(),
  build: buildCubit,
  act: (cubit) => cubit.load(),
  expect: () => [
    const OrdersState(status: OrdersStatus.loading),
    const OrdersState(status: OrdersStatus.failure, failure: NetworkFailure()),
  ],
);
```

Use **mocks** (`mockito` com `@GenerateNiceMocks`, como orienta a [skill oficial de Dart](https://github.com/dart-lang/skills)) só quando o objetivo do teste é verificar uma **interação**, por exemplo "o evento de analytics foi disparado uma vez":

```dart
@GenerateNiceMocks([MockSpec<AnalyticsService>()])
import 'orders_cubit_test.mocks.dart';

// ...
verify(analytics.logEvent('orders_list_loaded', any)).called(1);
```

- ✅ Em stubs de métodos assíncronos, sempre `thenAnswer((_) async => valor)`, nunca `thenReturn`.
- ✅ Fakes de infraestrutura (`FakeAnalyticsService`, `FakeCrashReporter`) também ficam em `testing/fakes/` e guardam as chamadas recebidas numa lista, o que costuma dispensar o mock.
