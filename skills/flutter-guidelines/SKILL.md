---
name: flutter-guidelines
description: Diretrizes de arquitetura Flutter da equipe. Use antes de escrever, alterar, especificar ou revisar código Flutter/Dart em projetos da equipe, para saber as regras obrigatórias, qual documento do guia ler e qual skill oficial do Flutter/Dart usar junto.
---

# Diretrizes Flutter da equipe

O guia completo está na pasta [`guide/`](guide/README.md) desta skill (caminho absoluto: `${CLAUDE_SKILL_DIR}/guide/`). Esta skill é o **índice**: regras que nunca podem ser quebradas e o mapa de quais documentos ler. Antes de implementar ou revisar, **leia os documentos indicados no mapa** para o tipo de tarefa. Este resumo não substitui o guia.

## Precedência

1. **Guia da equipe** (`${CLAUDE_SKILL_DIR}/guide/`), sempre que ele trata do assunto. Exceções aprovadas para um projeto ficam no `CLAUDE.md` dele (ver "Particularidades do projeto").
2. **Skills oficiais** do plugin `dart-flutter` (flutter/agent-plugins) e de dart-lang/skills, para tudo que o guia não cobre e para o "como fazer" (comandos, APIs, configuração nativa).
3. **Código existente do projeto**, só como referência de domínio. Ele **não** é referência de padrão: código fora do guia é legado em migração.

Divergências **intencionais** do guia em relação ao Flutter oficial. Aqui o guia vence, mesmo que uma skill oficial diga o contrário:

| Tema | Oficial | Guia |
|---|---|---|
| Pastas | `lib/data` + `lib/domain` por tipo, `lib/ui/<feature>` | Tudo dentro de `lib/features/<feature>/` (`data/`, `ui/view_models/`, `ui/widgets/`); compartilhado em `lib/shared/` |
| Erros entre camadas | Retornar `Result<T>` | Lançar `Failure` (sealed) e capturar com `ErrorHandlerMixin.handle` no Cubit |
| Estado | `ChangeNotifier` + `Command` | `Cubit` no papel de ViewModel, State imutável com `status` |
| Models | `freezed`/`built_value` | `fromJson`/`toJson` manuais e estritos (pattern matching + `FormatException`) |

## Particularidades do projeto

O guia é genérico. O que muda de projeto para projeto vem do `CLAUDE.md` (e do `README.md`) do repositório onde você está trabalhando:

- flavors, ambientes e comandos de execução, build e teste;
- branches de integração e padrão de nomes de branch;
- design system usado e pacotes internos;
- idioma do código e dos commits.

Se o `CLAUDE.md` tiver uma seção **"Exceções ao guia"**, as exceções listadas valem para aquele projeto. Qualquer outro trecho do `CLAUDE.md` que contradiga o guia descreve código legado, não um padrão a seguir.

## Regras de bloqueio

Nunca faça o que está abaixo. Se o código existente faz, migre (ver "Migração" abaixo).

**Arquitetura e DI** ([01](guide/01-arquitetura.md))
- Feature importando outra feature; `core/` importando `features/`.
- `static X get instance`, variáveis globais, `get_it` ou qualquer service locator. Dependências chegam pelo construtor, em campos privados; a composição usa `RepositoryProvider`/`BlocProvider` e `context.read()` só em rotas e providers.
- Repository dependendo de outro repository.
- Nomes fora do padrão: pastas em `snake_case`; `ui/view_models/`, `ui/widgets/`; classes `XxxScreen`, `XxxCubit`, `XxxState`, `XxxRepository`(+`Impl`).

**Estado** ([02](guide/02-gerenciamento-estado.md))
- `BuildContext`, `material.dart`, navegação, diálogo ou toast dentro do Cubit.
- Operação assíncrona do Cubit fora de `handle(...)`.
- State mutável, sem `Equatable`, ou com `String errorMessage` em vez de `Failure`.
- Efeito colateral (toast, navegação) dentro de `BlocBuilder`; deve ficar em `BlocListener`.

**Componentes** ([03](guide/03-componentizacao.md))
- Widget abaixo da Screen lendo o Cubit (exceto `XxxSection` da mesma feature).
- Métodos `Widget _buildX()` no lugar de classes; cores, fontes e espaçamentos fixos no código.
- Tela que carrega dados sem os estados de loading, erro (com retry) e vazio.

**Rede e models** ([04](guide/04-requests-rede.md))
- `Dio()` criado fora do `ApiClient`; requisição fora de `safeCall`.
- `throw Exception('...')` ou `throw 'texto'`. Sempre uma subclasse de `Failure`.
- Repository expondo `Response`, `Map` ou `DioException`.
- Valor padrão silencioso para campo obrigatório no `fromJson` (`?? ''`, `?? DateTime.now()`).

**Erros e privacidade** ([05](guide/05-erros-crashlytics.md))
- Mostrar `error.toString()` ao usuário. Use `failure.userMessage`.
- Reportar o mesmo erro em duas camadas.
- Nome, e-mail, CPF, telefone, endereço, corpo de request/response ou URL com query string no Crashlytics ou no Analytics.

**Analytics** ([06](guide/06-analytics.md))
- `FirebaseAnalytics.instance` fora do `FirebaseAnalyticsService`; `user_id` como parâmetro de evento.
- Nome de evento fora de `<feature>_<objeto>_<ação>` em `snake_case`.

**Navegação e push** ([07](guide/07-navegacao.md), [08](guide/08-push-notifications.md))
- `Navigator.push(MaterialPageRoute(...))`; `GoRoute` sem `name`; dado obrigatório só em `extra`.
- Lógica de rota de push fora do `PushRouter`.

**Ambiente** ([09](guide/09-flavors-ambientes.md))
- `if (environment == ...)` ou `kReleaseMode` para decidir ambiente; `appFlavor` lido fora do `main.dart`.
- Segredos no código ou em `dart-define` versionado.

## Mapa: tarefa → guia → skill oficial

| Tarefa | Leia no guia | Skill oficial complementar |
|---|---|---|
| Feature nova (ponta a ponta) | 01, 02, 03, 04, 06 | `flutter-apply-architecture-best-practices` (respeitando a precedência) |
| Tela, componente, layout | 03 | `flutter-build-responsive-layout`, `flutter-fix-layout-issues`, `flutter-add-widget-preview` |
| Estado / Cubit | 02 | `dart-use-pattern-matching` |
| Chamada de API, repository | 04, 05 | `flutter-use-http-package` (só conceitos; o cliente é Dio) |
| Model / JSON | 04 | `flutter-implement-json-serialization`, `dart-use-pattern-matching` |
| Tratamento de erro, Crashlytics | 05 | `dart-fix-runtime-errors` |
| Analytics | 06 | — |
| Rotas, deep link | 07 | `flutter-setup-declarative-routing` |
| Push notification | 08, 07 | — |
| Flavors, ambientes | 09 | — |
| Testes | 02 (seção Testando) | `dart-add-unit-test`, `flutter-add-widget-test`, `dart-generate-test-mocks`, `flutter-add-integration-test` |
| Lint, análise, formatação | — | `dart-run-static-analysis` |
| Conflito de dependências | — | `dart-resolve-package-conflicts` |
| Documentação de API (`///`) | — | `dart-write-documentation` |

## Migração de código legado

Todo código escrito **ou alterado** segue o guia. Ao tocar código fora do padrão:

- **Dentro da feature tocada:** migre o que for necessário para a mudança ficar no padrão (ex.: Cubit que usa `XService.instance` passa a receber a dependência pelo construtor). Faça isso num commit `refactor:` separado, **antes** da mudança de comportamento, sem alterar comportamento.
- **Fora da feature tocada** (infraestrutura em `core/`, DI do app inteiro, `ApiClient`, outra feature): **pare e peça confirmação**, descrevendo o que precisa mudar e quantos arquivos são afetados.
- Divergências vistas e não migradas entram no relatório final.

## Validação mínima

Antes de considerar qualquer mudança pronta:
1. `dart format` nos arquivos alterados.
2. `dart analyze` sem erros nem warnings nos arquivos tocados (via ferramenta de análise do Dart MCP server, ou CLI).
3. Testes novos para o que foi criado (Cubit com `bloc_test` e fakes de `testing/fakes/`; `fromJson` com JSON válido e fora do contrato) e testes existentes afetados passando.
