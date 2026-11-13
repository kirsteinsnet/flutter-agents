---
name: flutter-spec
description: Escreve a spec de uma feature Flutter da equipe (produto + desenho técnico pelo guia) a partir de texto livre, link do Figma e contrato da API, conversando para fechar as lacunas, e commita a spec em docs/specs/ na branch da feature. Use quando o usuário pedir para especificar, detalhar ou escrever a spec de uma feature, tela ou fluxo antes de implementar, ou mencionar "spec", "especificação", "documentar a feature", mesmo sem citar a skill.
---

# Spec de feature Flutter

Você escreve a spec que o agente `flutter-dev` vai implementar. A spec tem duas metades: **produto** (o que e por quê, com critérios de aceite rastreáveis) e **desenho técnico** (estrutura e nomes seguindo o guia da equipe, sem código). Responda em português do Brasil.

O esqueleto obrigatório está em [`template.md`](template.md) (caminho absoluto: `${CLAUDE_SKILL_DIR}/template.md`). Leia-o antes de escrever.

A seção técnica segue a skill `flutter-guidelines` deste plugin. Se ela não estiver no seu contexto, invoque-a pela ferramenta Skill antes de começar.

## Modo de execução

- **Interativo** (esta skill rodando na conversa principal): você **pergunta** ao usuário para fechar as lacunas, uma pergunta por vez.
- **Não interativo** (rodando dentro do agente `flutter-spec`): você **não pode perguntar**. Cada lacuna vira um item na seção 11 (Perguntas em aberto), com a suposição adotada quando houver uma razoável. As instruções do agente indicam quando você está nesse modo.

Todo o resto do fluxo é igual nos dois modos.

## Entrada

- **Texto livre** descrevendo a feature (obrigatório, pode ser vago).
- **Link do Figma** das telas (esperado para qualquer feature com UI).
- **Contrato da API**: OpenAPI/Swagger ou JSON de exemplo, por caminho de arquivo ou colado. Opcional, mas sem ele os models ficam como pergunta em aberto.

## Fluxo

### 1. Pré-checagem do Figma

Se houver link do Figma, confirme que as ferramentas do MCP do Figma estão disponíveis: procure ferramentas como `get_design_context`, `get_metadata`, `get_screenshot` e `get_variable_defs` de um servidor do Figma (o nome do servidor varia conforme a configuração de cada dev; se as ferramentas estiverem diferidas, busque com ToolSearch pelo termo `figma`).

Se não estiverem disponíveis, **pare sem escrever nada** e responda:

> O MCP do Figma não está configurado nesta sessão, então não consigo ler as telas. Configure-o conforme a seção "MCP do Figma" do README do plugin `flutter-agents`, abra uma sessão nova e peça a spec de novo.

Não aceite prints como substituto nem siga sem o design: toda spec com UI precisa do design lido de fato.

Se o link estiver inacessível (sem permissão, node inexistente), pare e diga qual link falhou e o erro retornado.

### 2. Contexto do projeto

- Leia o `CLAUDE.md` e o `README.md` do projeto: branches de integração e padrão de nomes, design system e tokens, idioma de código e commits, pasta de specs (padrão `docs/specs/`) e a seção "Exceções ao guia".
- Leia, com a ferramenta Read, os documentos do guia em `${CLAUDE_PLUGIN_ROOT}/skills/flutter-guidelines/guide/` exigidos pela feature: sempre 01, 02, 03, 04 e 06; 07 se houver rota nova ou deep link; 08 se houver push; 05 para listar as `Failure`s.
- Se a feature altera algo existente, explore `lib/features/<feature>/` e as rotas atuais para não especificar o que já existe. Código fora do guia é legado: descreva o desenho técnico **no padrão do guia**, não no padrão do código atual.
- Liste `docs/specs/` para ver se já existe spec da feature (ver "Spec existente").

### 3. Ler o design

Para cada tela/frame do link:
- `get_metadata` para a estrutura e os nodes de cada tela e variação;
- `get_screenshot` para ver layout e estados;
- `get_design_context` para hierarquia e componentes;
- `get_variable_defs` para cores, tipografia e espaçamentos, **mapeando para os tokens do design system do projeto**. Variável sem correspondente vira pergunta.

Registre o link de cada node usado na seção 5. Identifique quais estados (loading, sucesso, vazio, erro, variações) o design cobre e quais faltam.

### 4. Ler o contrato da API

Mapeie cada campo usado na tela para o campo da API: tipo Dart, obrigatório ou opcional, nome no JSON. Identifique os endpoints do repository e as `Failure`s possíveis (guia 05).

### 5. Fechar as lacunas

Percorra o checklist abaixo. No **modo interativo**, pergunte **uma coisa por vez**, de múltipla escolha quando der, com a sua recomendação primeiro. Pergunte só o que não dá para deduzir da entrada, do design, da API ou do guia. Pare quando o checklist zerar ou quando o usuário pedir para deixar o resto em aberto.

**Checklist de lacunas:**
- [ ] Objetivo e fora do escopo claros.
- [ ] Cada tela tem os estados **carregando, sucesso, vazio e erro com retry**. Estado que o design não mostra: perguntar se usa o padrão do design system (`ErrorView`, `EmptyView`) ou se falta design.
- [ ] Cada campo da tela existe no contrato da API. **Campo na tela sem origem na API, ou campo opcional na API mostrado como obrigatório, é sempre pergunta, nunca suposição silenciosa.**
- [ ] Campos opcionais da API têm comportamento definido na tela (oculta, placeholder, texto alternativo).
- [ ] Regras de negócio explícitas: quem pode fazer o quê e em qual status.
- [ ] Ações com efeito (enviar, cancelar, excluir): confirmação, feedback de sucesso e de erro, comportamento ao voltar.
- [ ] Navegação: de onde se entra, para onde se sai, parâmetros obrigatórios (no path, nunca só em `extra`), deep link ou push.
- [ ] Analytics: eventos de interação relevantes, nomeados `<feature>_<objeto>_<ação>`, sem dados pessoais nem `user_id` como parâmetro.
- [ ] Paginação, pull-to-refresh, cache ou atualização em tempo real, se a tela listar dados.
- [ ] Permissões, login obrigatório e feature flags, se aplicável.

### 6. Escrever a spec

- Caminho: `docs/specs/<slug>.md` (ou a pasta definida no `CLAUDE.md`). `<slug>` em `kebab-case`, curto, em inglês se o código do projeto for em inglês (ex.: `order-detail`).
- Siga o `template.md` **seção por seção, sem pular nenhuma**. Seção sem conteúdo recebe "Não se aplica" com o motivo.
- IDs: `RF-01`, `RN-01`, `CA-01`. Todo `CA` referencia ao menos um `RF` ou `RN`; todo `RF` é coberto por ao menos um `CA`.
- Desenho técnico (seção 7): nomes e estrutura pelo guia (`XxxScreen`, `XxxCubit`, `XxxState` com `status`, `XxxRepository` + `Impl`, pastas em `lib/features/<feature>/`). Sem código Dart.
- **Status:** `pronta` só se a seção 11 estiver vazia; senão, `rascunho`.
- **Privacidade:** a spec fica versionada. Nunca copie dados pessoais reais do Figma, da API ou da conversa para exemplos; troque por valores fictícios.
- Remova o comentário de instruções do template.

No **modo interativo**, mostre ao usuário um resumo (requisitos, telas e estados, desenho técnico, perguntas em aberto) e aplique os ajustes pedidos antes de commitar.

### 7. Branch e commit

- `git status` e `git branch --show-current`.
- Se a branch atual for de integração (as do `CLAUDE.md`; na falta, `main`, `master`, `develop`, `release/*` ou equivalente), crie `feature/<slug>` a partir dela, seguindo o padrão de nomes do `CLAUDE.md` se houver. Se já estiver numa branch de feature, use-a.
- `git add` **somente** do arquivo da spec. Nunca `git add -A` ou `git add .`. Alterações alheias não commitadas: não inclua nem descarte; avise no relatório.
- Mensagem em Conventional Commits, no idioma e estilo do histórico (`git log --oneline -15`): `docs: spec de <feature>` para spec nova, `docs: atualiza spec de <feature>` para atualização. Corpo curto com o status e o número de perguntas em aberto.
- Nunca faça `push`, `rebase` de branch compartilhada, `reset --hard` ou `force`.

### 8. Relatório

Entregue:
- caminho da spec, branch e hash do commit;
- status (`pronta` ou `rascunho`) e a lista de perguntas em aberto, se houver;
- estados ou telas que o design não cobre;
- próximo passo: "use o flutter-dev para implementar `docs/specs/<slug>.md`" (se `pronta`) ou fechar as perguntas antes (se `rascunho`).

## Spec existente

Se `docs/specs/<slug>.md` já existir:
- **Interativo:** pergunte se é para atualizar a spec existente ou criar uma nova com outro slug.
- **Não interativo:** atualize a existente.

Ao atualizar, preserve os IDs existentes (não renumere; item removido fica riscado), atualize a data e use a mensagem `docs: atualiza spec de <feature>`. O histórico do git é a trilha de auditoria.

## Limites

- Não escreva nem altere código, testes, `pubspec.yaml` ou configuração do projeto. A spec é o único arquivo que você cria ou altera.
- Não invente regra de negócio, campo de API ou estado de tela: o que não está na entrada nem no design vira pergunta.
- Não leia nem copie para a spec segredos, chaves ou URLs com tokens.
