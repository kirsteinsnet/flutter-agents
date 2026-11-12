# Flutter Agents

Plugin do Claude Code com as **diretrizes de arquitetura Flutter da equipe** e os **agentes** que as aplicam. Um único repositório serve a todos os projetos Flutter: quando o guia ou um agente muda aqui, todos os projetos recebem a atualização.

Ele trabalha junto com o plugin oficial [`dart-flutter`](https://github.com/flutter/agent-plugins) da equipe do Flutter, que traz as skills oficiais Flutter/Dart e o Dart MCP server. Ele é declarado como dependência e instalado junto.

## O que tem aqui

| Componente | Tipo | Papel |
|---|---|---|
| [`flutter-guidelines`](skills/flutter-guidelines/SKILL.md) | Skill | Índice do guia: precedência, divergências intencionais em relação ao Flutter oficial, regras de bloqueio, mapa "tarefa → documento → skill oficial" e política de migração de legado. Usada por todos os agentes. |
| [Guia completo](skills/flutter-guidelines/guide/README.md) | Documentação | 9 documentos: arquitetura, estado, componentização, rede, erros/Crashlytics, analytics, navegação, push, flavors. |
| [`flutter-spec`](skills/flutter-spec/SKILL.md) | Skill | Escreve a spec de uma feature (produto + desenho técnico pelo guia) a partir de texto, link do Figma e contrato da API, **conversando** para fechar as lacunas. Commita a spec em `docs/specs/` na branch da feature. |
| [`flutter-spec`](agents/flutter-spec.md) | Agente | Mesma spec, em modo **não interativo**: para entrada já completa ou para encadear com o `flutter-dev`. Lacunas viram "Perguntas em aberto" em vez de perguntas. |
| [`flutter-dev`](agents/flutter-dev.md) | Agente | Implementa features, bugfixes e refatorações seguindo o guia, valida (format, analyze, testes) e faz commits atômicos numa branch de feature, sem push. |

Próximos agentes: code review e distribuição (Firebase App Distribution e lojas).

## Pré-requisitos

- Claude Code instalado e autenticado.
- `dart` e `flutter` no `PATH` do shell que abre o Claude Code, para o Dart MCP server subir. Com FVM, adicione ao `~/.zshrc`:

  ```bash
  export PATH="$HOME/fvm/default/bin:$PATH"
  ```

- MCP do Figma configurado, para gerar specs (ver abaixo).

### MCP do Figma

A skill e o agente `flutter-spec` leem as telas pelo link do Figma, usando o MCP oficial do Figma. O plugin **não** o declara: cada dev configura uma vez, no escopo de usuário. Sem ele, o `flutter-spec` para e pede a configuração (prints não substituem o link).

Servidor remoto (não precisa do app desktop aberto):

```bash
claude mcp add --scope user --transport http figma https://mcp.figma.com/mcp
```

Depois, numa sessão do Claude Code no terminal, rode `/mcp`, escolha `figma` e autentique com a sua conta do Figma.

Alternativa, servidor do app desktop (exige o Figma desktop aberto com o Dev Mode MCP server habilitado nas preferências):

```bash
claude mcp add --scope user --transport http figma-desktop http://127.0.0.1:3845/mcp
```

Confira com `claude mcp list` e abra uma sessão nova.

## Adicionar a um projeto

1. No `.claude/settings.json` do projeto (versionado), declare os dois marketplaces e habilite os plugins:

   ```json
   {
     "extraKnownMarketplaces": {
       "flutter-agents": {
         "source": {
           "source": "github",
           "repo": "kirsteinsnet/flutter-agents"
         }
       },
       "dart-flutter": {
         "source": {
           "source": "github",
           "repo": "flutter/agent-plugins"
         }
       }
     },
     "enabledPlugins": {
       "flutter-agents@flutter-agents": true,
       "dart-flutter@dart-flutter": true
     }
   }
   ```

2. Cada dev, uma vez por máquina e por projeto:

   ```bash
   claude plugin install flutter-agents@flutter-agents --scope project
   ```

   O `dart-flutter` é instalado como dependência. Se não for, rode `claude plugin install dart-flutter@dart-flutter --scope project`.

3. No `CLAUDE.md` do projeto, descreva o que é específico dele (o guia é genérico):
   - flavors, ambientes e comandos de execução, build e teste;
   - branches de integração (ex.: `main`, `develop`, `hom`) e padrão de nomes de branch;
   - design system e pacotes internos;
   - opcional: uma seção **"Exceções ao guia"** com desvios aprovados para aquele projeto.

   Recomendado: uma seção curta dizendo que o código fora do guia é legado em migração, para que o agente não o copie como padrão.

4. Abra uma **sessão nova** do Claude Code (plugins e agentes carregam no início da sessão).

## Usar

### 1. Spec

Conversando (recomendado para feature nova ou requisito vago):

> /flutter-agents:flutter-spec tela de detalhe do pedido. Figma: <link>. API: `docs/api/order.json`

A skill lê o projeto, o guia, o Figma e o contrato da API e pergunta uma coisa por vez até fechar as lacunas (estados de tela, campos opcionais, regras, navegação, analytics). Depois escreve `docs/specs/<slug>.md`, mostra um resumo e commita na branch `feature/<slug>`, sem push.

Delegando (entrada já completa, ou encadeando com a implementação):

> use o agente flutter-spec para gerar a spec do detalhe do pedido; Figma: <link>; API em `docs/api/order.json`; cancelamento só em pending

O agente não pergunta: o que faltar vai para a seção "Perguntas em aberto" da spec, com a suposição adotada, e a spec fica como `rascunho`.

A spec tem IDs rastreáveis (`RF-xx`, `RN-xx`, `CA-xx`) e fica versionada para auditoria. O esqueleto está em [`skills/flutter-spec/template.md`](skills/flutter-spec/template.md).

### 2. Implementação

Na mesma branch da spec, peça no Claude Code:

> use o flutter-dev para implementar a tela de detalhe de pedidos conforme `docs/specs/order-detail.md`

O agente lê o `CLAUDE.md` do projeto, os documentos do guia que a tarefa exige e as skills oficiais correspondentes. Depois planeja, cria a branch, implementa em fatias validadas e entrega um relatório com commits, migrações e o que não foi verificado.

## Atualizar

```bash
claude plugin marketplace update flutter-agents
claude plugin update flutter-agents@flutter-agents
```

## Contribuir

- Mudança no guia: edite `skills/flutter-guidelines/guide/` e, se mudar uma regra, atualize também as regras de bloqueio e o mapa em `skills/flutter-guidelines/SKILL.md`.
- Mudança de regra (não só de texto) precisa de aprovação de pelo menos um tech lead de outro projeto que siga o guia.
- Suba a `version` em `.claude-plugin/plugin.json` (semver) e registre no [CHANGELOG](CHANGELOG.md).
- Antes do merge request, rode:

  ```bash
  claude plugin validate .
  ```

## Estrutura

```
.claude-plugin/
  plugin.json          # manifest do plugin (versão, dependência do dart-flutter)
  marketplace.json     # este repo também é o marketplace
agents/
  flutter-dev.md
  flutter-spec.md      # spec em modo não interativo
skills/
  flutter-guidelines/
    SKILL.md           # índice do guia
    guide/             # guia completo (01..09 + README)
  flutter-spec/
    SKILL.md           # fluxo e regras da spec
    template.md        # esqueleto da spec
```
