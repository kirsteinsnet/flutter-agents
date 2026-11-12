# Diretrizes de Arquitetura para Projetos Flutter

Guia de referência para desenvolvedores que criam ou mantêm aplicativos Flutter seguindo a mesma linha de desenvolvimento. Ele nasceu da experiência acumulada em projetos reais, mas é **genérico**: os exemplos usam um domínio fictício (`orders`, `Order`) e o pacote `my_app`.

O guia segue as recomendações oficiais da equipe do Flutter (veja [Alinhamento com o Flutter oficial](#alinhamento-com-o-flutter-oficial)).

---

## Stack de referência

| Preocupação | Escolha |
|---|---|
| Arquitetura | MVVM do guia oficial (View + ViewModel, Repository + Service) |
| Gerenciamento de estado | `flutter_bloc` (Cubit no papel de ViewModel) + `equatable` |
| Injeção de dependência | `provider`, via `RepositoryProvider`/`BlocProvider` do `flutter_bloc` |
| Navegação | `go_router` |
| HTTP | `dio` |
| Observabilidade | Firebase Crashlytics + Firebase Analytics |
| Push | Firebase Cloud Messaging + `flutter_local_notifications` |
| Configuração remota | Firebase Remote Config (opcional) |
| Testes | `bloc_test`, fakes, `mockito` (`@GenerateNiceMocks`) para interações, `http_mock_adapter` |
| Pré-visualização | Widget Previewer (`@Preview`) |

Trocar uma peça dessa stack é uma decisão de arquitetura: registre o motivo no repositório do projeto (ADR ou seção no README).

---

## Índice

| # | Documento | Assunto |
|---|---|---|
| 01 | [Arquitetura](01-arquitetura.md) | Camadas MVVM, organização de pastas, injeção de dependência, bootstrap |
| 02 | [Gerenciamento de estado](02-gerenciamento-estado.md) | Cubit como ViewModel, State, efeitos colaterais, escopo, testes com fakes |
| 03 | [Componentização](03-componentizacao.md) | Design system, widgets compartilhados e locais, estados de tela, previews |
| 04 | [Requests e rede](04-requests-rede.md) | ApiClient, interceptors, `safeCall`, repositories, models estritos |
| 05 | [Erros e Crashlytics](05-erros-crashlytics.md) | `Failure` tipada, `ErrorHandlerMixin`, erros globais, privacidade |
| 06 | [Analytics](06-analytics.md) | `AnalyticsService`, eventos por feature, nomenclatura |
| 07 | [Navegação](07-navegacao.md) | GoRouter, rotas protegidas, navegação sem `context`, deep links |
| 08 | [Push notifications](08-push-notifications.md) | FCM, notificações locais, roteamento por payload |
| 09 | [Flavors e ambientes](09-flavors-ambientes.md) | `appFlavor`, `AppConfig`, Firebase por ambiente |

Leia na ordem na primeira vez. Depois, use como consulta.

---

## Princípios

1. **Dependência em uma direção.** A UI conhece o Cubit, o Cubit conhece o Repository, o Repository conhece o ApiClient. Nunca o contrário.
2. **Feature autocontida.** Tudo que pertence a uma funcionalidade mora na pasta dela. Só sobe para `shared/` o que é usado por duas ou mais features.
3. **Dependências pelo construtor.** Nenhuma classe de negócio busca suas dependências sozinha, e não há objetos globais. Isso torna tudo testável com fakes.
4. **Erros tipados, registrados uma vez.** Toda falha vira uma `Failure` com mensagem para o usuário e é reportada em um único ponto.
5. **Privacidade por padrão.** Nenhum dado pessoal (nome, e-mail, documento) vai para Crashlytics ou Analytics.
6. **Simplicidade.** Não crie camada, abstração ou pacote que o problema atual não exige.

---

## Alinhamento com o Flutter oficial

Fontes de referência, em ordem de prioridade:

1. [docs.flutter.dev](https://docs.flutter.dev/), em especial o [guia de arquitetura](https://docs.flutter.dev/app-architecture/guide), as [recomendações](https://docs.flutter.dev/app-architecture/recommendations) e o [case study](https://docs.flutter.dev/app-architecture/case-study).
2. Skills mantidas pela equipe do Flutter: [flutter/agent-plugins](https://github.com/flutter/agent-plugins) e [dart-lang/skills](https://github.com/dart-lang/skills). Use-as também com agentes de IA no dia a dia.

Divergências **intencionais**, cada uma explicada no próprio documento:

| Tema | Oficial | Este guia | Onde |
|---|---|---|---|
| Pastas | Dados por tipo (`lib/data`), UI por feature (`lib/ui`) | Dados e UI dentro da feature | [01](01-arquitetura.md#diferença-em-relação-ao-guia-oficial) |
| Erros entre camadas | Retornar `Result<T>` | Lançar `Failure` e capturar no Cubit | [04](04-requests-rede.md#diferença-em-relação-ao-guia-oficial) |
| Ferramenta de estado | `ChangeNotifier` + `Command` (opcional) | `Cubit` como ViewModel | [02](02-gerenciamento-estado.md#o-cubit-é-o-viewmodel) |
| Models | `freezed`/`built_value` (recomendado) | Parsing manual e estrito, sem codegen | [04](04-requests-rede.md#models) |

Qualquer outra diferença em relação às fontes oficiais é um erro do guia: abra um merge request.

---

## Uso com agentes de IA (Claude Code)

Este guia é distribuído como parte do plugin `flutter-agents`, junto com a skill `flutter-guidelines` (índice do guia) e o agente `flutter-dev`. Instalação e uso estão no [README do repositório](../../../README.md).

---|---|
| `.claude/skills/flutter-guidelines/SKILL.md` | Índice do guia: precedência, regras de bloqueio e mapa "tarefa → documento → skill oficial". Usado por todos os agentes. |
| `.claude/skills/flutter-spec/` + `.claude/agents/flutter-spec.md` | Spec de feature (produto + desenho técnico pelo guia) a partir de texto, Figma e contrato da API, commitada em `docs/specs/`. |
| `.claude/agents/flutter-dev.md` | Agente de desenvolvimento: implementa seguindo o guia, valida e faz commits atômicos numa branch de feature (sem push). |
| `.claude/settings.json` | Declara o plugin oficial `dart-flutter` (skills Flutter/Dart + Dart MCP server). |

Na primeira vez, cada dev instala o plugin oficial:

```bash
claude plugin install dart-flutter@dart-flutter --scope project
```

Para usar o agente, peça no Claude Code algo como "use o flutter-dev para implementar ...", ou passe o caminho de uma spec.

Para levar a outro projeto, copie `docs/flutter-guidelines/` e a pasta `.claude/` (`settings.json`, `skills/flutter-guidelines/`, `agents/`).

---

## Como cada documento está organizado

- **Objetivo**: o que o tema resolve.
- **Regras**: lista de ✅ FAÇA / ❌ NÃO FAÇA.
- **Por quê**: o raciocínio por trás das regras, para decidir os casos que o guia não cobre.
- **Código de referência**: implementação pronta para copiar e adaptar.

## Como propor mudanças

Abra um merge request alterando o documento do tema. Mudanças de regra (não só de texto) precisam de aprovação de pelo menos um tech lead de outro projeto que siga o guia.
