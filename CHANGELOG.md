# Changelog

Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/); versões em [SemVer](https://semver.org/lang/pt-BR/).

## [0.2.0] - 2026-11-12

### Adicionado

- Skill `flutter-spec`: escreve a spec de uma feature (produto + desenho técnico pelo guia) a partir de texto, link do Figma (via MCP do Figma) e contrato da API, perguntando uma lacuna por vez, e commita em `docs/specs/<slug>.md` na branch da feature.
- Template da spec com IDs rastreáveis (`RF`, `RN`, `CA`), estados de tela, desenho técnico, analytics, testes esperados e perguntas em aberto.
- Agente `flutter-spec`: a mesma spec em modo não interativo; lacunas viram "Perguntas em aberto" e a spec fica como `rascunho`.
- README: configuração do MCP do Figma e fluxo spec → implementação.

### Alterado

- `flutter-dev`: o relatório final cita os IDs de requisitos e critérios de aceite quando a entrada é uma spec.

## [0.1.0] - 2026-11-11

### Adicionado

- Guia de arquitetura Flutter (01 a 09), alinhado ao Flutter oficial, com 4 divergências intencionais documentadas.
- Skill `flutter-guidelines`: índice do guia, regras de bloqueio, mapa "tarefa → documento → skill oficial", política de migração de legado e particularidades por projeto via `CLAUDE.md`.
- Agente `flutter-dev`: desenvolvimento seguindo o guia, com validação por fatia e commits atômicos.
- Dependência do plugin oficial `dart-flutter`.
