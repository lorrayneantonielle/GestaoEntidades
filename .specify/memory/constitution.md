<!--
Sync Impact Report
==================
Version change: [TEMPLATE] → 1.0.0 (initial ratification)
Modified principles: N/A (first fill of template placeholders)
Added sections:
  - I. Clean Architecture no Backend
  - II. Contrato-First e Versionamento de API
  - III. Identidade e Persistência (UUID, EF Core, Liquibase)
  - IV. Segurança e Autorização (RBAC)
  - V. Arquitetura de Componentes Frontend (Vue 3)
  - VI. Qualidade, Validação e Histórico de Commits
  - Governance
Removed sections: none (template placeholders only)
Templates requiring updates:
  - .specify/templates/plan-template.md ⚠ pending manual review (verify Constitution Check gates reference these principles)
  - .specify/templates/spec-template.md ⚠ pending manual review (no conflicting requirements found)
  - .specify/templates/tasks-template.md ⚠ pending manual review (ensure task categories cover backend/frontend/quality gates above)
  - .claude/skills/speckit-constitution (this file) ✅ no outdated agent-specific references found
Follow-up TODOs: none — all placeholders resolved from user-supplied principles.
-->

# MLBEntidades Constitution

## Core Principles

### I. Clean Architecture no Backend
O backend (.NET 8) MUST seguir a separação em camadas Controllers → Services →
Repositories, com dependências fluindo em uma única direção (Controllers
dependem de Services, Services dependem de Repositories; nunca o inverso).
Controllers MUST conter apenas orquestração HTTP (binding, status codes,
delegação), sem lógica de negócio. Regras de negócio MUST residir em Services.
Acesso a dados MUST ser isolado em Repositories, nunca invocado diretamente por
Controllers.

**Rationale**: A separação de camadas mantém o sistema testável e substituível
em partes isoladas, essencial para uma plataforma de gestão que evoluirá com
múltiplos módulos (famílias, obras, entidades) ao longo do tempo.

### II. Contrato-First e Versionamento de API
Todo endpoint MUST ter seu contrato OpenAPI definido antes do início da
implementação. Endpoints RESTful MUST ser expostos sob `/api/v1/` (ou versão
subsequente explícita), nunca sem prefixo de versão. Toda resposta de erro
MUST seguir o padrão ProblemDetails (RFC 7807), incluindo `type`, `title`,
`status`, `detail` e extensões relevantes ao domínio.

**Rationale**: Definir o contrato antes do código evita retrabalho entre
frontend e backend e permite geração de clientes/documentação consistente.
Versionamento explícito protege consumidores existentes de breaking changes;
ProblemDetails padroniza o tratamento de erro em toda a API.

### III. Identidade e Persistência (UUID, EF Core, Liquibase)
Todo identificador de entidade MUST ser UUID (`Guid` no .NET,
`gen_random_uuid()` no PostgreSQL) — chaves sequenciais/inteiras autoincrementais
NÃO SÃO permitidas para chaves primárias de domínio. EF Core MUST ser usado
exclusivamente como ORM (mapeamento e consultas); MUST NOT ser usado para gerar
ou aplicar migrations automáticas. O schema do banco de dados MUST ser
gerenciado exclusivamente via Liquibase (changelogs versionados); qualquer
alteração de schema fora do Liquibase é proibida.

**Rationale**: UUIDs evitam colisão entre ambientes e facilitam integração
futura entre serviços/entidades distribuídas do movimento social. Separar
ORM de gerenciamento de schema (Liquibase) garante trilha de auditoria e
controle de rollback independente do ciclo de vida da aplicação.

### IV. Segurança e Autorização (RBAC)
Toda operação sensível MUST ser protegida por autorização baseada em perfis
(RBAC), com os perfis mínimos `AdminGeral`, `AssistenteSocial` e
`TecnicoObra`. Novas operações MUST declarar explicitamente quais perfis têm
acesso; não é permitido endpoint autenticado sem policy de autorização
associada. A área administrativa autenticada MUST ser mantida separada
(rotas, layout e guards de navegação) da landing page pública, que MUST
permanecer acessível sem autenticação.

**Rationale**: O sistema lida com dados sensíveis de famílias e entidades
sociais; RBAC explícito reduz risco de exposição indevida. Separar área
pública da administrativa evita acoplamento entre necessidades de SEO/acesso
público e requisitos de segurança da área logada.

### V. Arquitetura de Componentes Frontend (Vue 3)
Componentes MUST ser escritos com Composition API usando `<script setup>` —
Options API não é permitida em código novo. Estado global MUST ser
gerenciado via Pinia; estado local de componente permanece em `ref`/`reactive`
dentro do próprio componente. Chamadas HTTP MUST ser isoladas em composables
organizados por domínio (ex.: `useFamilias`, `useObra`), nunca feitas
diretamente dentro de componentes de UI. Cada componente MUST ter
responsabilidade única — componentes que acumulam múltiplas preocupações
(dados + apresentação + navegação) MUST ser decompostos.

**Rationale**: Composition API com composables por domínio isola regras de
integração da camada de apresentação, tornando os componentes reutilizáveis e
testáveis independentemente da origem dos dados.

### VI. Qualidade, Validação e Histórico de Commits
Todo command/DTO de entrada MUST ter validação declarada via FluentValidation
antes de alcançar a lógica de negócio — validação manual ad-hoc dentro de
Services não substitui essa camada. Commits MUST ser atômicos por
funcionalidade (um commit não deve misturar funcionalidades não relacionadas),
permitindo reversão isolada sem efeitos colaterais em outras features.

**Rationale**: Validação centralizada em FluentValidation evita duplicação de
regras de entrada entre endpoints. Commits atômicos tornam o histórico
auditável e o rollback seguro, importante em um sistema que sofre auditoria
por lidar com benefícios sociais.

## Governance

Esta constituição prevalece sobre qualquer prática, convenção de equipe ou
preferência individual em conflito. Toda alteração de código revisada em PR
MUST ser verificada quanto à conformidade com os princípios acima; violações
MUST ser justificadas explicitamente no PR ou corrigidas antes do merge.

Emendas a esta constituição MUST ser propostas por escrito (PR alterando este
arquivo), MUST descrever o racional da mudança e MUST atualizar o campo
"Last Amended" e a versão semântica:
- **MAJOR**: remoção ou redefinição incompatível de um princípio existente.
- **MINOR**: adição de novo princípio ou expansão material de um princípio.
- **PATCH**: correções de redação, esclarecimentos, ajustes não semânticos.

Após qualquer emenda, os templates dependentes (`plan-template.md`,
`spec-template.md`, `tasks-template.md`) MUST ser revisados quanto à
consistência com os princípios atualizados antes da emenda ser considerada
concluída.

**Version**: 1.0.0 | **Ratified**: 2026-08-13 | **Last Amended**: 2026-08-13
