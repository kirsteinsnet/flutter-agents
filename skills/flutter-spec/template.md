# <Nome da feature>

> **Status:** rascunho | pronta
> **Figma:** <link do arquivo/frame>
> **Contrato da API:** <caminho do OpenAPI/JSON ou "colado na conversa">
> **Data:** <AAAA-MM-DD> · **Autor:** <nome do git user.name>

<!--
Regras deste template:
- Mantenha os IDs (RF-xx, RN-xx, CA-xx) estáveis entre versões. Ao remover um item, risque-o em vez de renumerar.
- A spec só fica "pronta" com a seção 11 vazia.
- Nada de dados pessoais reais em exemplos (nomes, e-mails, CPF, telefones, endereços). Use valores fictícios.
- A seção 7 descreve estrutura e nomes, não código.
- Apague este comentário na spec final.
-->

## 1. Contexto e objetivo

<Por que a feature existe, para quem e qual problema resolve. 2 a 5 frases.>

## 2. Fora do escopo

- <O que explicitamente não entra nesta entrega.>

## 3. Requisitos funcionais

| ID | Requisito |
|---|---|
| RF-01 | <O app deve ...> |

## 4. Regras de negócio

| ID | Regra |
|---|---|
| RN-01 | <Ex.: o botão "Cancelar" só aparece quando `status == pending`.> |

## 5. Telas e estados

### <Nome da tela> — [Figma](<link do node>)

| Estado | O que o usuário vê | Origem |
|---|---|---|
| Carregando | <skeleton/spinner> | <node do Figma ou "padrão do design system"> |
| Sucesso | <conteúdo> | |
| Vazio | <mensagem + ação> | |
| Erro | <`failure.userMessage` + botão "Tentar novamente"> | |
| <Variação> | <ex.: pedido de retirada na loja> | |

**Componentes do design system:** <componentes e tokens usados, mapeados das variáveis do Figma.>

## 6. Navegação

| Rota (`name`) | Path | Parâmetros obrigatórios (no path) | `extra` opcional |
|---|---|---|---|
| `<orderDetail>` | `<:id>` (filha de `</orders>`) | `<id>` | <—> |

- **Entrada:** <de onde se chega a esta tela>
- **Saída:** <para onde se vai e o que acontece ao voltar>
- **Deep link / push:** <tipo do payload e destino, ou "não se aplica">

## 7. Desenho técnico

Segue o guia `flutter-guidelines`. Pasta: `lib/features/<feature>/`.

### Models (`data/models/`)

| Model | Campo | Tipo Dart | Obrigatório | Campo na API | Observação |
|---|---|---|---|---|---|
| `<Order>` | `<id>` | `String` | sim | `<id>` | |

### Repository (`data/repositories/`)

| Método | Endpoint | Retorno | `Failure`s esperadas |
|---|---|---|---|
| `<getOrder(String id)>` | `<GET /orders/{id}>` | `<Future<Order>>` | `<NotFoundFailure, NetworkFailure, ...>` |

### Cubit e State (`ui/view_models/`)

- **State:** `status` (`initial`, `loading`, `success`, `failure`), <campos>, `Failure? failure`.
- **Ações do Cubit:** `<load()>`, `<cancel()>` — todas via `handle(...)`.

### Widgets (`ui/widgets/`)

- `<XxxScreen>` (lê o Cubit), `<XxxSection>` (lê o Cubit, mesma feature), <componentes puros>.

## 8. Analytics

| Evento | Quando | Parâmetros |
|---|---|---|
| `<order_detail_cancel_tap>` | <toque em "Cancelar"> | `<order_status>` |

> `screen_view` é automático pelo observer de navegação; não liste aqui.

## 9. Critérios de aceite

| ID | Cenário | Referência |
|---|---|---|
| CA-01 | **Dado** <contexto> **quando** <ação> **então** <resultado> | RF-01 |

## 10. Testes esperados

- **Models:** `fromJson` com JSON válido e com JSON fora do contrato (<campos>).
- **Repository:** sucesso e cada `Failure` da seção 7.
- **Cubit (`bloc_test`):** <sequências de estado por ação>.
- **Widget:** <estados da seção 5 que precisam de teste>.

## 11. Perguntas em aberto

| # | Pergunta | Suposição adotada |
|---|---|---|
| 1 | <pergunta> | <suposição, ou "nenhuma — bloqueia"> |
