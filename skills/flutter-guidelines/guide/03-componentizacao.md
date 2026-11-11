# 03 · Componentização

## Objetivo

Definir onde cada widget mora, como ele recebe dados e como manter a interface consistente entre telas e entre projetos.

---

## Três níveis de componente

| Nível | Onde | O que é | Exemplo |
|---|---|---|---|
| **1. Design system** | Pacote externo (Git) ou `lib/core/theme/` | Tokens (cores, tipografia, espaçamento, raio) e componentes básicos sem regra de app | `DsButton`, `DsTextField`, `DsCard` |
| **2. Compartilhado do app** | `lib/shared/ui/` | Componentes do app usados por 2+ features | `ErrorView`, `EmptyView`, `AppScaffold`, `LoginRequiredView` |
| **3. Local da feature** | `features/<f>/ui/widgets/` | Tudo que só faz sentido em uma feature | `OrderCard`, `OrderStatusChip` |

### Regra de promoção

1. Todo widget nasce no **nível 3**.
2. Quando uma **segunda** feature precisar dele, ele sobe para o **nível 2** (e perde qualquer referência à feature original).
3. Quando ele for útil para **outros apps**, propõe-se a inclusão no **nível 1**.

Nunca copie um widget de uma feature para outra. Promova.

---

## Componentes de apresentação

Widgets abaixo da Screen não leem o Cubit: recebem dados e callbacks. É o que o guia oficial chama de manter as Views "burras".

```dart
// ✅ Recebe dados prontos e devolve intenções
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
// ❌ Componente que conhece o Cubit: só funciona dentro desta tela
class OrderCard extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final order = context.watch<OrdersCubit>().state.orders[index];
    ...
  }
}
```

### Por quê

Um componente que recebe dados pode ser reutilizado, testado com `pumpWidget` sem montar Cubit e visualizado isoladamente no Widget Previewer (ver abaixo). Um componente que lê o Cubit fica preso à tela onde nasceu.

### Quem pode ler o Cubit

- A **Screen**.
- **Seções** grandes da Screen (`OrdersHeaderSection`, `OrdersListSection`) quando isso evita passar dez parâmetros adiante. A seção mora na mesma feature e o nome termina em `Section`.

---

## Regras de construção

- ✅ Extraia widgets em **classes**, não em métodos `Widget _buildHeader()`. Classes podem ser `const`, têm `build` próprio (o Flutter redesenha só elas) e aparecem com nome no DevTools.
- ✅ `const` em todo construtor e instância possível.
- ✅ Cores, fontes, espaçamentos e raios vêm do tema/tokens: `Theme.of(context)`, `AppSpacing.md`, `AppColors.primary`.
- ✅ Textos fixos de uma feature ficam em uma classe de constantes da feature (`OrdersStrings`), não espalhados nos widgets.
- ✅ Um widget por arquivo, salvo widgets privados pequenos usados só ali.
- ✅ Imagens e ícones com `semanticLabel`; áreas de toque com no mínimo 48×48.
- ❌ `Color(0xFF...)`, `fontSize: 14`, `EdgeInsets.all(13)` soltos no código.
- ✅ Decisões de layout pelo espaço disponível: `LayoutBuilder` (espaço do pai) e `MediaQuery.sizeOf(context)` (tamanho da janela), com `Expanded`/`Flexible` para distribuir espaço.
- ✅ `ListView.builder`/`GridView.builder` para listas longas ou de tamanho desconhecido.
- ❌ `MediaQuery.of(context).size` (redesenha em qualquer mudança do `MediaQuery`; use `sizeOf`).
- ❌ Decidir layout por tipo de aparelho ("é tablet?") ou por orientação no topo da árvore. Apps rodam em janelas redimensionáveis, multi-janela e dobráveis.
- ❌ Widgets com mais de ~150 linhas de `build`. Divida.

---

## Estados obrigatórios de uma tela

Toda tela que carrega dados trata os quatro estados, usando os componentes compartilhados:

| Estado | Componente | Observação |
|---|---|---|
| Carregando | Skeleton da feature (`OrdersSkeleton`) | Skeleton com o formato do conteúdo, não um spinner no meio da tela |
| Erro | `ErrorView(failure:, onRetry:)` | Mostra `failure.userMessage` e botão "Tentar novamente" |
| Vazio | `EmptyView(message:, action:)` | Texto que explica o porquê e, se houver, a ação para sair do vazio |
| Sucesso | Conteúdo | Com `RefreshIndicator` quando os dados podem mudar |

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
                label: 'Tentar novamente',
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

`failure.icon` pode ser uma extension em `Failure` dentro de `shared/ui/` (ícone de "sem conexão" para `NetworkFailure`, por exemplo), mantendo `core/errors/` livre de Flutter.

---

## Feedback ao usuário

Centralize toast/snackbar e diálogos em `shared/ui/` (`AppToast.success`, `AppToast.error`, `ConfirmDialog.show`). As features chamam esses helpers a partir do `BlocListener`, nunca constroem `SnackBar` à mão.

---

## Pré-visualização de widgets

Use o [Widget Previewer](https://docs.flutter.dev/tools/widget-previewer) oficial para desenvolver e revisar componentes isolados, sem navegar até a tela.

```dart
// features/orders/ui/widgets/order_card.dart
import 'package:flutter/widget_previews.dart';

@Preview(name: 'Pedido aberto', group: 'Orders')
Widget orderCardOpenPreview() => OrderCard(order: Order.sample(), onTap: () {});

@Preview(name: 'Pedido concluído', group: 'Orders', brightness: Brightness.dark)
Widget orderCardDonePreview() =>
    OrderCard(order: Order.sample(status: OrderStatus.done), onTap: () {});
```

- ✅ Previews para componentes dos níveis 2 e 3 com mais de um estado visual.
- ✅ Crie anotações próprias (estendendo `Preview`) para aplicar o tema do app a todos os previews.
- ❌ Dependências de `dart:io`, plugins nativos ou Cubit dentro do preview: o previewer roda na web. Mais um motivo para componentes receberem dados por parâmetro.

Rode pela IDE ou com:

```bash
flutter widget-preview start
```

---

## Design system externo

Se a organização mantém um design system em pacote:

- ✅ Dependa dele por tag de versão (`ref: v1.4.0`), nunca por branch.
- ✅ Componente que falta e é genérico → contribua com o pacote; enquanto isso, crie no nível 2 com o mesmo nome que terá lá.
- ❌ Sobrescrever estilos do design system dentro de uma feature. Se a variação é necessária, ela vira uma variante oficial do componente.
