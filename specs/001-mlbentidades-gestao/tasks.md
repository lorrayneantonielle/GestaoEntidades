---

description: "Task list for MLBEntidades — Plataforma de Gestão MCMV Entidades"
---

# Tasks: MLBEntidades — Plataforma de Gestão MCMV Entidades

**Input**: Design documents from `specs/001-mlbentidades-gestao/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml, quickstart.md

**Tests**: Não solicitados explicitamente na especificação (spec.md não pede TDD) — nenhuma
tarefa de teste de contrato/integração por história é gerada. `research.md` §3/§4 já
decide os frameworks de teste (xUnit/FluentAssertions/Moq no backend, Vitest/@vue-test-utils
no frontend); a configuração desses frameworks está no Setup (Phase 1) como infraestrutura
compartilhada, não como testes por história.

**Organization**: Tarefas agrupadas por user story (spec.md, priorizadas P1→P6) para permitir
implementação e teste independentes de cada uma.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Pode rodar em paralelo (arquivos diferentes, sem dependência de outra tarefa incompleta)
- **[Story]**: A qual user story a tarefa pertence (US1–US6)
- Caminhos de arquivo exatos incluídos em cada descrição

## Path Conventions

Monorepo com dois submodules (plan.md → Project Structure):

- **Backend**: `servico/ServicoMLBEntidades.sln` com 5 projetos —
  `ServicoMLBEntidades` (Api, já scaffolded), `ServicoMLBEntidades.Application`,
  `ServicoMLBEntidades.Domain`, `ServicoMLBEntidades.Infrastructure`,
  `ServicoMLBEntidades.UnitTests`; schema em `servico/db/changelog/` (Liquibase)
- **Frontend**: `frontend/src/` (Vue 3 + Vite + TS, já scaffolded com App.vue/main.ts)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Preparar solution, projetos, dependências e configuração base antes de qualquer
código de domínio.

- [X] T001 Adicionar os projetos `ServicoMLBEntidades.Domain`, `ServicoMLBEntidades.Application`,
  `ServicoMLBEntidades.Infrastructure` e `ServicoMLBEntidades.UnitTests` à
  `servico/ServicoMLBEntidades.sln`, com referências Api→Application→Domain,
  Infrastructure→Domain, Api→Infrastructure (Clean Architecture, plan.md Project Structure)
- [X] T002 [P] Adicionar pacotes NuGet por camada: EF Core (Npgsql) + Dapper em
  `servico/ServicoMLBEntidades.Infrastructure/ServicoMLBEntidades.Infrastructure.csproj`;
  FluentValidation em `servico/ServicoMLBEntidades.Application/ServicoMLBEntidades.Application.csproj`;
  ASP.NET Core Identity + JWT Bearer + Swashbuckle em
  `servico/ServicoMLBEntidades/ServicoMLBEntidades.csproj`
- [X] T003 [P] Adicionar xUnit, FluentAssertions e Moq em
  `servico/ServicoMLBEntidades.UnitTests/ServicoMLBEntidades.UnitTests.csproj` com referência a
  `ServicoMLBEntidades.Application` (research.md §3)
- [X] T004 [P] Instalar dependências do frontend ainda não presentes (`vuetify`, `@mdi/font`,
  `pinia`, `vue-router@4`, `axios`) em `frontend/package.json`
- [X] T005 [P] Configurar Vitest + `@vue/test-utils` no frontend: `frontend/vitest.config.ts` e
  script `test:unit` em `frontend/package.json` (research.md §4)
- [X] T006 [P] Criar esqueleto do projeto Liquibase em
  `servico/db/changelog/db.changelog-master.xml` referenciando a pasta
  `servico/db/changelog/migrations/v1.0.0/` (research.md §2)
- [X] T007 [P] Configurar `servico/ServicoMLBEntidades/appsettings.Development.json` com
  `ConnectionStrings:Default` (PostgreSQL 16), `Jwt:Secret`/`Jwt:Issuer`,
  `Storage:DocumentosPath` e `Cors:ProductionOrigin`
- [X] T008 [P] Configurar `frontend/.env.local` com `VITE_API_BASE_URL` e ajustar
  `frontend/vite.config.ts` se necessário para proxy de desenvolvimento

**Checkpoint**: Solution e projetos compiláveis, dependências instaladas, sem lógica de negócio ainda.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Autenticação/RBAC, schema base (Identity + ConfiguracaoSistema), tratamento de
erro padronizado e esqueleto de frontend (layouts, router, store de auth, cliente HTTP) —
tudo isso é pré-requisito de **todas** as user stories.

**⚠️ CRITICAL**: Nenhuma user story pode começar antes desta fase estar completa.

- [X] T009 Criar `ApplicationDbContext` (EF Core) com integração ao ASP.NET Core Identity em
  `servico/ServicoMLBEntidades.Infrastructure/Persistence/ApplicationDbContext.cs`
- [X] T010 Changelog Liquibase para tabelas do Identity (`AspNetUsers`, `AspNetRoles`,
  `AspNetUserRoles` etc.) e seed dos 3 roles (`AdminGeral`, `AssistenteSocial`,
  `TecnicoObra`) em `servico/db/changelog/migrations/v1.0.0/001-identity.sql`
  (data-model.md → Usuario/Role/RefreshToken)
- [X] T011 [P] Changelog Liquibase para tabela `refresh_tokens` (Id, UsuarioId, TokenHash,
  ExpiresAt, RevokedAt, CreatedAt) em
  `servico/db/changelog/migrations/v1.0.0/002-refresh-tokens.sql` (research.md §7)
- [X] T012 [P] Changelog Liquibase para tabela singleton `configuracao_sistema` com seed de
  `LimiteMinimoPontuacaoMutirao` padrão em
  `servico/db/changelog/migrations/v1.0.0/003-configuracao-sistema.sql` (data-model.md →
  ConfiguracaoSistema)
- [X] T013 [P] Criar entidade `RefreshToken` (Domain) e mapeamento EF Core (Infrastructure) em
  `servico/ServicoMLBEntidades.Domain/Entities/RefreshToken.cs` e
  `servico/ServicoMLBEntidades.Infrastructure/Persistence/Configurations/RefreshTokenConfiguration.cs`
- [X] T014 Implementar serviço de geração/validação de JWT (access token 8h) e rotação de
  refresh token (7 dias, invalida o anterior a cada uso) em
  `servico/ServicoMLBEntidades.Infrastructure/Auth/JwtTokenService.cs` (research.md §7,
  depende de T013)
- [X] T015 Implementar `AuthService` (login, refresh-token, logout) em
  `servico/ServicoMLBEntidades.Application/Auth/AuthService.cs` com
  `LoginCommand`/`RefreshTokenCommand` e validadores FluentValidation em
  `servico/ServicoMLBEntidades.Application/Auth/Validators/` (contracts/openapi.yaml
  `/auth/*`, depende de T014)
- [X] T016 Implementar `AuthController` (`POST /auth/login`, `POST /auth/refresh-token`,
  `POST /auth/logout`) em
  `servico/ServicoMLBEntidades/Controllers/AuthController.cs` (depende de T015)
- [X] T017 Configurar ASP.NET Core Identity + autenticação JWT Bearer + políticas de
  autorização por role (`AdminGeral`, `AssistenteSocial`, `TecnicoObra`) em
  `servico/ServicoMLBEntidades/Program.cs` (constitution Princípio IV)
- [X] T018 [P] Configurar política de CORS nomeada (`http://localhost:5173` +
  `Cors:ProductionOrigin`, sem wildcard) em `servico/ServicoMLBEntidades/Program.cs`
  (research.md §8)
- [X] T019 [P] Implementar middleware global de exceção mapeando erros para `ProblemDetails`
  (RFC 7807) em `servico/ServicoMLBEntidades/Middlewares/ProblemDetailsExceptionMiddleware.cs`
  (constitution Princípio II)
- [X] T020 [P] Configurar Swashbuckle/Swagger com suporte a JWT Bearer ("Authorize") em
  `servico/ServicoMLBEntidades/Program.cs` (quickstart.md §3)
- [X] T021 [P] Implementar `IDocumentoStorageService` (Domain) e implementação em disco local
  lendo `Storage:DocumentosPath` (Infrastructure) em
  `servico/ServicoMLBEntidades.Domain/Services/IDocumentoStorageService.cs` e
  `servico/ServicoMLBEntidades.Infrastructure/Storage/LocalDocumentoStorageService.cs`
  (research.md §5)
- [X] T022 [P] Configurar plugin Vuetify (tema, ícones `@mdi/font`) em
  `frontend/src/plugins/vuetify.ts` e registrar em `frontend/src/main.ts`
- [X] T023 [P] Configurar Vue Router com `AdminLayout.vue`/`PublicLayout.vue` e guards de
  navegação (rotas `/admin/*` exigem autenticação; rotas públicas livres) em
  `frontend/src/router/index.ts`, `frontend/src/layouts/AdminLayout.vue`,
  `frontend/src/layouts/PublicLayout.vue` (constitution Princípio IV)
- [X] T024 [P] Configurar cliente HTTP Axios base (baseURL de `VITE_API_BASE_URL`,
  interceptor de JWT, retry de refresh em 401) em
  `frontend/src/services/httpClient.ts`
- [X] T025 [P] Configurar Pinia e `authStore` (login/logout/refresh, persistência do token)
  em `frontend/src/main.ts` e `frontend/src/stores/authStore.ts` (depende de T024)
- [X] T026 [P] Implementar `authService.ts`, composable `useAuth` e página de login em
  `frontend/src/services/authService.ts`, `frontend/src/composables/useAuth.ts`,
  `frontend/src/pages/admin/LoginPage.vue` (depende de T025)

**Checkpoint**: Login funcional, RBAC aplicado, layouts/rotas base, tratamento de erro
padronizado — todas as user stories podem começar.

---

## Phase 3: User Story 1 - Cadastro e Acompanhamento de Famílias Beneficiárias (Priority: P1) 🎯 MVP

**Goal**: Assistente Social cadastra famílias com composição familiar, renda e
vulnerabilidade, controla documentação obrigatória e avança/reverte o status da família
pelo workflow completo, com busca/filtro da listagem.

**Independent Test**: Cadastrar uma família com membros, renda e documentos; avançar e
reverter seu status pelo workflow completo (registrando motivo na reversão); buscar/filtrar
a listagem por status, nome e número de membros — sem depender de nenhum outro módulo.

### Implementation for User Story 1

- [X] T027 [P] [US1] Changelog Liquibase para tabelas `familias`, `membros`, `documentos`,
  `familia_status_historico` — incluindo índice único condicional de `cpf` entre membros de
  famílias não excluídas (FR-033) em
  `servico/db/changelog/migrations/v1.0.0/004-familias.sql` (data-model.md → Membro)
- [X] T028 [P] [US1] Criar entidade `Familia` + enum `FamiliaStatus` em
  `servico/ServicoMLBEntidades.Domain/Entities/Familia.cs` e
  `servico/ServicoMLBEntidades.Domain/Enums/FamiliaStatus.cs`
- [X] T029 [P] [US1] Criar entidade `Membro` em
  `servico/ServicoMLBEntidades.Domain/Entities/Membro.cs`
- [X] T030 [P] [US1] Criar entidade `Documento` + enums `DocumentoTipo`/`DocumentoStatus` em
  `servico/ServicoMLBEntidades.Domain/Entities/Documento.cs`,
  `servico/ServicoMLBEntidades.Domain/Enums/DocumentoTipo.cs`,
  `servico/ServicoMLBEntidades.Domain/Enums/DocumentoStatus.cs`
- [X] T031 [P] [US1] Criar entidade `FamiliaStatusHistorico` em
  `servico/ServicoMLBEntidades.Domain/Entities/FamiliaStatusHistorico.cs`
- [X] T032 [US1] Configurar mapeamentos EF Core (`IEntityTypeConfiguration`) de
  Familia/Membro/Documento/FamiliaStatusHistorico e registrar no `ApplicationDbContext` em
  `servico/ServicoMLBEntidades.Infrastructure/Persistence/Configurations/` (depende de
  T028-T031, T009)
- [X] T033 [P] [US1] Implementar `IFamiliaRepository` (Domain) e `FamiliaRepository` (EF Core,
  Infrastructure) em
  `servico/ServicoMLBEntidades.Domain/Repositories/IFamiliaRepository.cs` e
  `servico/ServicoMLBEntidades.Infrastructure/Repositories/FamiliaRepository.cs` (depende de
  T032)
- [X] T034 [US1] Implementar `FamiliaService` (`CreateFamilia`, `UpdateFamilia`,
  `ListFamilias` com filtros status/nome/numeroMembros, `GetFamilia`) em
  `servico/ServicoMLBEntidades.Application/Familias/FamiliaService.cs` (FR-008, FR-014,
  depende de T033)
- [X] T035 [US1] Implementar `FamiliaStatusService.UpdateStatus` (avanço bloqueado por
  documentação pendente FR-011, reversão exigindo motivo FR-012, liberação de unidade ao
  reverter FR-013, marcação manual `EmConstrucao→Finalizada` FR-032) em
  `servico/ServicoMLBEntidades.Application/Familias/FamiliaStatusService.cs` (depende de
  T033)
- [X] T036 [P] [US1] Implementar `FamiliaCreateCommand`/`FamiliaUpdateCommand`/
  `FamiliaStatusUpdateCommand` + validadores FluentValidation em
  `servico/ServicoMLBEntidades.Application/Familias/Commands/` e
  `servico/ServicoMLBEntidades.Application/Familias/Validators/`
- [X] T037 [P] [US1] Implementar `MembroService` (criar/atualizar/excluir + bloqueio de CPF
  duplicado entre famílias ativas, FR-033) em
  `servico/ServicoMLBEntidades.Application/Membros/MembroService.cs` (depende de T033)
- [X] T038 [P] [US1] Implementar `MembroCommand` + validador FluentValidation (formato de CPF)
  em `servico/ServicoMLBEntidades.Application/Membros/Validators/MembroCommandValidator.cs`
- [X] T039 [P] [US1] Implementar `DocumentoService` (upload via `IDocumentoStorageService`,
  listagem por família, cálculo de completude documental) em
  `servico/ServicoMLBEntidades.Application/Documentos/DocumentoService.cs` (FR-009, depende
  de T021, T033)
- [X] T040 [P] [US1] Implementar validação de upload (tipo MIME, tamanho máximo) para
  `DocumentoCommand` em
  `servico/ServicoMLBEntidades.Application/Documentos/Validators/DocumentoCommandValidator.cs`
- [X] T041 [US1] Implementar `FamiliasController` (`GET/POST /familias`,
  `GET/PUT /familias/{id}`, `PATCH /familias/{id}/status`) com
  `[Authorize(Roles = "AdminGeral,AssistenteSocial")]` em cadastro/edição e leitura liberada
  a qualquer perfil autenticado (FR-031) em
  `servico/ServicoMLBEntidades/Controllers/FamiliasController.cs` (depende de T034-T036)
- [X] T042 [US1] Implementar `MembrosController` (`GET/POST /membros`,
  `PUT/DELETE /membros/{id}`) em
  `servico/ServicoMLBEntidades/Controllers/MembrosController.cs` (depende de T037, T038)
- [X] T043 [US1] Implementar `DocumentosController` (`GET/POST /documentos`, multipart/form-data)
  em `servico/ServicoMLBEntidades/Controllers/DocumentosController.cs` (depende de T039, T040)
- [X] T044 [P] [US1] Criar tipos TS `Familia`, `Membro`, `Documento` em
  `frontend/src/types/familia.ts`
- [X] T045 [P] [US1] Implementar `familiaService.ts` (chamadas Axios) em
  `frontend/src/services/familiaService.ts` (depende de T024, T044)
- [X] T046 [P] [US1] Implementar composable `useFamilias` em
  `frontend/src/composables/useFamilias.ts` (depende de T045)
- [X] T047 [P] [US1] Implementar `familiasStore` (Pinia) em
  `frontend/src/stores/familiasStore.ts` (depende de T045)
- [X] T048 [US1] Implementar página de listagem de famílias com filtros (status, nome,
  número de membros) em `frontend/src/pages/admin/familias/FamiliasListPage.vue` (depende
  de T046, T047)
- [X] T049 [US1] Implementar formulário de cadastro/edição de família (composição familiar,
  renda, vulnerabilidade) em `frontend/src/pages/admin/familias/FamiliaFormPage.vue`
  (depende de T046)
- [X] T050 [US1] Implementar componente de workflow de status (avançar/reverter com motivo
  obrigatório na reversão) em `frontend/src/components/admin/FamiliaStatusWorkflow.vue`
  (depende de T046)
- [X] T051 [US1] Implementar componente de checklist/upload de documentos obrigatórios em
  `frontend/src/components/admin/FamiliaDocumentosChecklist.vue` (depende de T046)

**Checkpoint**: User Story 1 completa e testável de forma independente (cadastro, documentos,
workflow de status, busca/filtro).

---

## Phase 4: User Story 2 - Controle da Obra por Etapas e Medições (Priority: P2)

**Goal**: Técnico de Obra cadastra etapas da construção, atualiza percentual de conclusão,
registra medições do governo vinculadas a etapas e registra ocorrências técnicas.

**Independent Test**: Cadastrar etapas da obra (fundação, estrutura, alvenaria, cobertura,
acabamento), atualizar o percentual de conclusão de cada uma, registrar uma medição
vinculada a uma etapa e adicionar uma ocorrência técnica — isoladamente dos demais módulos.

### Implementation for User Story 2

- [X] T052 [P] [US2] Changelog Liquibase para tabelas `etapas_obra`, `medicoes`,
  `ocorrencias` em `servico/db/changelog/migrations/v1.0.0/005-obra.sql`
- [X] T053 [P] [US2] Criar entidade `EtapaObra` em
  `servico/ServicoMLBEntidades.Domain/Entities/EtapaObra.cs`
- [X] T054 [P] [US2] Criar entidade `Medicao` + enum `StatusAprovacao` em
  `servico/ServicoMLBEntidades.Domain/Entities/Medicao.cs` e
  `servico/ServicoMLBEntidades.Domain/Enums/StatusAprovacao.cs`
- [X] T055 [P] [US2] Criar entidade `Ocorrencia` em
  `servico/ServicoMLBEntidades.Domain/Entities/Ocorrencia.cs`
- [X] T056 [US2] Configurar mapeamentos EF Core de EtapaObra/Medicao/Ocorrencia e registrar
  no `ApplicationDbContext` em
  `servico/ServicoMLBEntidades.Infrastructure/Persistence/Configurations/` (depende de
  T053-T055, T009)
- [X] T057 [P] [US2] Implementar `IEtapaObraRepository`/`IMedicaoRepository`/
  `IOcorrenciaRepository` (Domain) e repositórios EF Core (Infrastructure) em
  `servico/ServicoMLBEntidades.Domain/Repositories/` e
  `servico/ServicoMLBEntidades.Infrastructure/Repositories/` (depende de T056)
- [X] T058 [US2] Implementar `EtapaObraService` (`CreateEtapa`, `UpdatePercentual`,
  `ListEtapas`) em `servico/ServicoMLBEntidades.Application/Obra/EtapaObraService.cs`
  (FR-020, FR-021, depende de T057)
- [X] T059 [P] [US2] Implementar `EtapaObraCommand` + validador FluentValidation em
  `servico/ServicoMLBEntidades.Application/Obra/Validators/EtapaObraCommandValidator.cs`
- [X] T060 [US2] Implementar `MedicaoService` (registro manual, sinaliza divergência entre
  percentual da etapa e medição aprovada) em
  `servico/ServicoMLBEntidades.Application/Obra/MedicaoService.cs` (FR-022, depende de T057)
- [X] T061 [P] [US2] Implementar `MedicaoCommand` + validador FluentValidation em
  `servico/ServicoMLBEntidades.Application/Obra/Validators/MedicaoCommandValidator.cs`
- [X] T062 [US2] Implementar `OcorrenciaService` em
  `servico/ServicoMLBEntidades.Application/Obra/OcorrenciaService.cs` (FR-023, depende de
  T057)
- [X] T063 [P] [US2] Implementar `OcorrenciaCommand` + validador FluentValidation em
  `servico/ServicoMLBEntidades.Application/Obra/Validators/OcorrenciaCommandValidator.cs`
- [X] T064 [US2] Implementar `ObraController` (`GET/POST /obra/etapas`,
  `PATCH /obra/etapas/{id}`, `GET/POST /obra/medicoes`, `GET/POST /obra/ocorrencias`) com
  `[Authorize(Roles = "AdminGeral,TecnicoObra")]` em
  `servico/ServicoMLBEntidades/Controllers/ObraController.cs` (depende de T058-T063)
- [X] T065 [P] [US2] Criar tipos TS `EtapaObra`, `Medicao`, `Ocorrencia` em
  `frontend/src/types/obra.ts`
- [X] T066 [P] [US2] Implementar `obraService.ts` em `frontend/src/services/obraService.ts`
  (depende de T024, T065)
- [X] T067 [P] [US2] Implementar composable `useObra` em `frontend/src/composables/useObra.ts`
  (depende de T066)
- [X] T068 [P] [US2] Implementar `obraStore` (Pinia) em `frontend/src/stores/obraStore.ts`
  (depende de T066)
- [X] T069 [US2] Implementar página de etapas da obra (listagem, cadastro, atualização de
  percentual) em `frontend/src/pages/admin/obra/EtapasPage.vue` (depende de T067, T068)
- [X] T070 [US2] Implementar página de medições (registro manual, listagem por etapa) em
  `frontend/src/pages/admin/obra/MedicoesPage.vue` (depende de T067)
- [X] T071 [US2] Implementar componente/página de ocorrências técnicas em
  `frontend/src/pages/admin/obra/OcorrenciasPage.vue` (depende de T067)

**Checkpoint**: User Stories 1 e 2 funcionam de forma independente.

---

## Phase 5: User Story 3 - Página Pública de Acompanhamento do Empreendimento (Priority: P3)

**Goal**: Visitante sem autenticação acessa informações gerais, status da obra por etapa,
histórico de medições aprovadas e calendário de mutirões futuros.

**Independent Test**: Acessar a landing page sem autenticação e verificar que informações
gerais, progresso por etapa, histórico de medições aprovadas e calendário de mutirões
futuros são exibidos corretamente com os dados publicados pela área administrativa.

### Implementation for User Story 3

- [X] T072 [P] [US3] Implementar `PublicService` (`GetStatus`, `GetEtapas`,
  `GetMedicoesAprovadas`, `GetProximosMutiroes` — leituras agregadas via Dapper) em
  `servico/ServicoMLBEntidades.Application/Public/PublicService.cs` (FR-001 a FR-004,
  research.md §6/§10; depende das entidades de US2 (T053-T055) e US5 (T093-T094) já
  existirem no schema)
- [X] T073 [US3] Implementar `PublicController` (`GET /public/status`, `GET /public/etapas`,
  `GET /public/medicoes`, `GET /public/mutiroes`) com `[AllowAnonymous]` em
  `servico/ServicoMLBEntidades/Controllers/PublicController.cs` (depende de T072)
- [X] T074 [P] [US3] Criar tipos TS `PublicStatus`, `PublicEtapa`, `PublicMedicao`,
  `PublicMutirao` em `frontend/src/types/public.ts`
- [X] T075 [P] [US3] Implementar `publicService.ts` em
  `frontend/src/services/publicService.ts` (depende de T024, T074)
- [X] T076 [P] [US3] Implementar composable `usePublic` em
  `frontend/src/composables/usePublic.ts` (depende de T075)
- [X] T077 [US3] Implementar landing page com informações gerais do empreendimento em
  `frontend/src/pages/public/LandingPage.vue` (depende de T076, T023)
- [X] T078 [US3] Implementar seção de status da obra por etapa (com estado vazio informativo
  quando sem dados) na landing page em
  `frontend/src/components/public/StatusObraSection.vue` (depende de T076)
- [X] T079 [US3] Implementar seção de histórico de medições aprovadas (ordem cronológica, com
  estado vazio) em `frontend/src/components/public/MedicoesSection.vue` (depende de T076)
- [X] T080 [US3] Implementar seção de calendário de próximos mutirões (com estado vazio) em
  `frontend/src/components/public/MutiroesCalendarSection.vue` (depende de T076)

**Checkpoint**: Página pública funcional e consistente com os dados administrativos
publicados por US2 e US5.

---

## Phase 6: User Story 4 - Gestão de Unidades Habitacionais e Atribuição a Famílias (Priority: P4)

**Goal**: Administrador Geral/Técnico de Obra cadastram unidades/lotes, atribuem unidades a
famílias aprovadas e visualizam o mapa de ocupação do terreno.

**Independent Test**: Cadastrar unidades com identificador, metragem e localização; atribuir
uma unidade a uma família com status "Aprovada"; visualizar o mapa de ocupação mostrando
unidades livres, reservadas e ocupadas.

### Implementation for User Story 4

- [X] T081 [P] [US4] Changelog Liquibase para tabela `unidades_habitacionais` (`identificador`
  único, índice único parcial em `familia_id` para status != Livre) em
  `servico/db/changelog/migrations/v1.0.0/006-unidades.sql` (FR-015, FR-017)
- [X] T082 [P] [US4] Criar entidade `UnidadeHabitacional` + enum `UnidadeStatus` em
  `servico/ServicoMLBEntidades.Domain/Entities/UnidadeHabitacional.cs` e
  `servico/ServicoMLBEntidades.Domain/Enums/UnidadeStatus.cs`
- [X] T083 [US4] Configurar mapeamento EF Core de `UnidadeHabitacional` e registrar no
  `ApplicationDbContext` em
  `servico/ServicoMLBEntidades.Infrastructure/Persistence/Configurations/UnidadeHabitacionalConfiguration.cs`
  (depende de T082, T009)
- [X] T084 [P] [US4] Implementar `IUnidadeRepository` (Domain) e `UnidadeRepository` (EF Core,
  Infrastructure) em
  `servico/ServicoMLBEntidades.Domain/Repositories/IUnidadeRepository.cs` e
  `servico/ServicoMLBEntidades.Infrastructure/Repositories/UnidadeRepository.cs` (depende de
  T083)
- [X] T085 [US4] Implementar `UnidadeService` (`CreateUnidade`, `ListUnidades` para mapa de
  ocupação, `AtribuirUnidade` validando `Familia.Status ∈ {Aprovada+}` FR-016, bloqueio de
  unidade não-Livre FR-017, avanço de `Familia.Status` para `UnidadeAtribuida` FR-018, em
  transação) em `servico/ServicoMLBEntidades.Application/Unidades/UnidadeService.cs`
  (depende de T033, T084)
- [X] T086 [P] [US4] Implementar `UnidadeCommand` + validador FluentValidation em
  `servico/ServicoMLBEntidades.Application/Unidades/Validators/UnidadeCommandValidator.cs`
- [X] T087 [US4] Implementar `UnidadesController` (`GET/POST /unidades`,
  `PUT /unidades/{id}/atribuicao`) com `[Authorize(Roles = "AdminGeral,TecnicoObra")]` em
  `servico/ServicoMLBEntidades/Controllers/UnidadesController.cs` (depende de T085, T086)
- [X] T088 [P] [US4] Criar tipos TS `UnidadeHabitacional` em
  `frontend/src/types/unidade.ts`
- [X] T089 [P] [US4] Implementar `unidadeService.ts` em
  `frontend/src/services/unidadeService.ts` (depende de T024, T088)
- [X] T090 [P] [US4] Implementar composable `useUnidades` em
  `frontend/src/composables/useUnidades.ts` (depende de T089)
- [X] T091 [P] [US4] Implementar `unidadesStore` (Pinia) em
  `frontend/src/stores/unidadesStore.ts` (depende de T089)
- [X] T092 [US4] Implementar mapa de ocupação (unidades livres/reservadas/ocupadas
  visualmente diferenciadas) em `frontend/src/pages/admin/unidades/MapaOcupacaoPage.vue`
  (depende de T090, T091)
- [X] T093 [US4] Implementar fluxo de cadastro e atribuição de unidade a família aprovada em
  `frontend/src/pages/admin/unidades/UnidadeFormPage.vue` (depende de T090)

**Checkpoint**: User Story 4 funcional; unidades cadastráveis isoladamente e atribuíveis a
famílias aprovadas por US1.

---

## Phase 7: User Story 5 - Mutirão e Sistema de Presença e Pontuação (Priority: P5)

**Goal**: Técnico de Obra cria escalas de mutirão com vagas e pontuação, registra presença de
famílias, concede pontuação e sinaliza famílias com baixa participação.

**Independent Test**: Criar uma escala de mutirão com vagas definidas, registrar presença de
famílias participantes, verificar a pontuação concedida conforme o valor configurado para
aquele mutirão, e confirmar que uma família cuja pontuação cai abaixo do limite mínimo é
sinalizada para o Assistente Social.

### Implementation for User Story 5

- [X] T094 [P] [US5] Changelog Liquibase para tabelas `mutirao_escalas` e `presencas` (par
  `mutirao_escala_id`+`familia_id` único) em
  `servico/db/changelog/migrations/v1.0.0/007-mutirao.sql` (FR-024, FR-025)
- [X] T095 [P] [US5] Criar entidade `MutiraoEscala` + enum `Turno` em
  `servico/ServicoMLBEntidades.Domain/Entities/MutiraoEscala.cs` e
  `servico/ServicoMLBEntidades.Domain/Enums/Turno.cs`
- [X] T096 [P] [US5] Criar entidade `Presenca` em
  `servico/ServicoMLBEntidades.Domain/Entities/Presenca.cs`
- [X] T097 [US5] Configurar mapeamentos EF Core de `MutiraoEscala`/`Presenca` e registrar no
  `ApplicationDbContext` em
  `servico/ServicoMLBEntidades.Infrastructure/Persistence/Configurations/` (depende de
  T095, T096, T009)
- [X] T098 [P] [US5] Implementar `IMutiraoEscalaRepository`/`IPresencaRepository` (Domain) e
  repositórios EF Core (Infrastructure) em
  `servico/ServicoMLBEntidades.Domain/Repositories/` e
  `servico/ServicoMLBEntidades.Infrastructure/Repositories/` (depende de T097)
- [X] T099 [US5] Implementar `MutiraoService` (`CreateEscala`, `ListEscalas` com
  `vagasDisponiveis`) em `servico/ServicoMLBEntidades.Application/Mutirao/MutiraoService.cs`
  (FR-024, depende de T098)
- [X] T100 [US5] Implementar `PresencaService.RegistrarPresenca` (bloqueio ao exceder vagas
  FR-025, bloqueio de presença duplicada, crédito de pontuação copiando
  `MutiraoEscala.PontuacaoPorPresenca` para `Presenca.PontuacaoConcedida` e somando em
  `Familia.PontuacaoAcumulada` FR-026) em
  `servico/ServicoMLBEntidades.Application/Mutirao/PresencaService.cs` (depende de T033,
  T098)
- [X] T101 [P] [US5] Implementar `MutiraoEscalaCommand` + `PresencaCommand` e validadores
  FluentValidation em `servico/ServicoMLBEntidades.Application/Mutirao/Validators/`
- [X] T102 [P] [US5] Implementar `PontuacaoService` (Dapper) — relatório por família com
  histórico de presenças, pontuação acumulada e flag `baixaParticipacao` a partir de
  `ConfiguracaoSistema.LimiteMinimoPontuacaoMutirao` (FR-027, FR-028) em
  `servico/ServicoMLBEntidades.Infrastructure/Queries/PontuacaoQueryService.cs` (depende de
  T012, T094)
- [X] T103 [US5] Implementar `MutiraoController` (`GET/POST /mutirao/escalas`,
  `POST /mutirao/presencas`, `GET /mutirao/pontuacao`) com
  `[Authorize(Roles = "AdminGeral,TecnicoObra")]` para escritas em
  `servico/ServicoMLBEntidades/Controllers/MutiraoController.cs` (depende de T099-T102)
- [X] T104 [P] [US5] Criar tipos TS `MutiraoEscala`, `Presenca`, `PontuacaoFamilia` em
  `frontend/src/types/mutirao.ts`
- [X] T105 [P] [US5] Implementar `mutiraoService.ts` em
  `frontend/src/services/mutiraoService.ts` (depende de T024, T104)
- [X] T106 [P] [US5] Implementar composable `useMutirao` em
  `frontend/src/composables/useMutirao.ts` (depende de T105)
- [X] T107 [P] [US5] Implementar `mutiraoStore` (Pinia) em
  `frontend/src/stores/mutiraoStore.ts` (depende de T105)
- [X] T108 [US5] Implementar página de escalas de mutirão (criação, listagem, vagas
  disponíveis) em `frontend/src/pages/admin/mutirao/EscalasPage.vue` (depende de T106, T107)
- [X] T109 [US5] Implementar registro de presença e relatório de pontuação/participação (com
  destaque de baixa participação) em
  `frontend/src/pages/admin/mutirao/PresencaPontuacaoPage.vue` (depende de T106)

**Checkpoint**: User Story 5 funcional de forma independente, alimentando o calendário
público (US3) e o dashboard (US6).

---

## Phase 8: User Story 6 - Dashboard Administrativo por Perfil (Priority: P6)

**Goal**: Qualquer usuário autenticado visualiza um dashboard agregado (famílias por status,
percentual de conclusão da obra, próximos mutirões) filtrado conforme seu perfil.

**Independent Test**: Autenticar com cada um dos três perfis e verificar que o conteúdo do
dashboard corresponde às responsabilidades daquele perfil (AssistenteSocial vê indicadores de
famílias; TecnicoObra vê indicadores de obra e mutirão; AdminGeral vê todos).

### Implementation for User Story 6

- [ ] T110 [US6] Implementar `DashboardService` (Dapper) — agrega total de famílias por
  status, percentual de conclusão da obra, próximos mutirões e famílias com baixa
  participação, montando o payload conforme o role do usuário autenticado (FR-029, FR-030)
  em `servico/ServicoMLBEntidades.Application/Dashboard/DashboardService.cs` (depende das
  entidades de US1, US2 e US5 já existirem no schema)
- [ ] T111 [US6] Implementar `DashboardController` (`GET /dashboard`) com `[Authorize]` (todos
  os perfis autenticados, filtragem interna por role) em
  `servico/ServicoMLBEntidades/Controllers/DashboardController.cs` (depende de T110)
- [ ] T112 [P] [US6] Criar tipo TS `DashboardResponse` em
  `frontend/src/types/dashboard.ts`
- [ ] T113 [P] [US6] Implementar `dashboardService.ts` em
  `frontend/src/services/dashboardService.ts` (depende de T024, T112)
- [ ] T114 [P] [US6] Implementar composable `useDashboard` em
  `frontend/src/composables/useDashboard.ts` (depende de T113)
- [ ] T115 [US6] Implementar página de dashboard com indicadores condicionais por perfil
  (AdminGeral vê todos; AssistenteSocial vê famílias/baixa participação; TecnicoObra vê
  obra/mutirão) em `frontend/src/pages/admin/dashboard/DashboardPage.vue` (depende de T114)

**Checkpoint**: Todas as 6 user stories funcionam de forma independente e integrada.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Consistência e robustez transversal após todas as histórias priorizadas
estarem implementadas.

- [ ] T116 [P] Implementar tratamento global de erros/snackbar no frontend a partir das
  respostas `ProblemDetails` da API em `frontend/src/composables/useApiError.ts`
- [ ] T117 [P] Adicionar guards de rota por perfil em cada rota administrativa (bloqueio de
  acesso fora do escopo do perfil com mensagem clara, FR-006/FR-007/SC-007) em
  `frontend/src/router/index.ts`
- [ ] T118 [P] Revisar cobertura de FluentValidation em todos os Commands de entrada
  (constitution Princípio VI) em `servico/ServicoMLBEntidades.Application/**/Validators/`
- [ ] T119 [P] Revisar consistência de `ProblemDetails` (type/title/status/detail) em todos os
  controllers e no middleware de exceção em `servico/ServicoMLBEntidades/Controllers/` e
  `servico/ServicoMLBEntidades/Middlewares/ProblemDetailsExceptionMiddleware.cs`
- [ ] T120 Executar o smoke test do fluxo mínimo (User Story 1) descrito em quickstart.md §5
  ponta a ponta (login → criar família → upload de documentos → avançar status)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sem dependências — pode começar imediatamente
- **Foundational (Phase 2)**: Depende de Setup — BLOQUEIA todas as user stories
- **User Stories (Phase 3–8)**: Todas dependem de Foundational completo
  - US1 (P1) e US2 (P2) são totalmente independentes entre si e podem rodar em paralelo
  - US3 (P3) consome dados publicados por US2 (obra) e US5 (mutirão) — implementar depois
    dessas duas para ter conteúdo real, embora o endpoint em si não tenha dependência de
    código
  - US4 (P4) depende de US1 (famílias aprovadas) para a atribuição ter sentido de negócio;
    o cadastro de unidades em si é independente
  - US5 (P5) depende de US1 (famílias existentes) para registrar presença; a criação de
    escalas é independente
  - US6 (P6) agrega dados de US1, US2 e US5 — implementar por último
- **Polish (Phase 9)**: Depende de todas as user stories desejadas estarem completas

### User Story Dependencies

- **US1 (P1)**: Após Foundational — sem dependência de outras stories
- **US2 (P2)**: Após Foundational — sem dependência de outras stories
- **US3 (P3)**: Após Foundational; conteúdo relevante requer US2 e US5 com dados publicados
- **US4 (P4)**: Após Foundational; atribuição de unidade requer famílias de US1 com status
  Aprovada+
- **US5 (P5)**: Após Foundational; registro de presença requer famílias de US1
- **US6 (P6)**: Após Foundational; dashboard completo requer US1, US2 e US5

### Within Each User Story

- Liquibase changelog → entidades (Domain) → mapeamentos EF Core/DbContext → repositórios →
  services → commands/validators → controllers → tipos TS → services/composables/stores
  frontend → páginas/componentes
- Cada história deve ficar completa e testável antes de prosseguir para a próxima em ordem
  de prioridade

### Parallel Opportunities

- Todas as tarefas [P] do Setup podem rodar em paralelo
- Todas as tarefas [P] do Foundational podem rodar em paralelo (dentro da Phase 2)
- Após o Foundational, **US1 e US2 podem ser desenvolvidas em paralelo** por equipes
  diferentes (nenhuma depende da outra)
- Dentro de cada história, entidades de Domain marcadas [P] (arquivos diferentes) podem
  rodar em paralelo; o mesmo vale para tipos TS/services/composables/stores do frontend

---

## Parallel Example: User Story 1

```bash
# Lançar as entidades de Domain de US1 em paralelo:
Task: "Criar entidade Familia + enum FamiliaStatus em servico/ServicoMLBEntidades.Domain/Entities/Familia.cs"
Task: "Criar entidade Membro em servico/ServicoMLBEntidades.Domain/Entities/Membro.cs"
Task: "Criar entidade Documento + enums em servico/ServicoMLBEntidades.Domain/Entities/Documento.cs"
Task: "Criar entidade FamiliaStatusHistorico em servico/ServicoMLBEntidades.Domain/Entities/FamiliaStatusHistorico.cs"

# Lançar a camada de frontend de US1 em paralelo (após services prontos):
Task: "Criar tipos TS Familia/Membro/Documento em frontend/src/types/familia.ts"
Task: "Implementar familiaService.ts em frontend/src/services/familiaService.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 apenas)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (CRÍTICO — bloqueia todas as histórias)
3. Completar Phase 3: User Story 1
4. **PARAR e VALIDAR**: testar User Story 1 de forma independente (quickstart.md §5)
5. Demonstrar/entregar se pronto

### Incremental Delivery

1. Setup + Foundational → base pronta
2. US1 (Famílias) → testar independentemente → MVP
3. US2 (Obra) → testar independentemente (pode ser feito em paralelo com US1)
4. US3 (Página Pública) → testar independentemente (com dados de US2/US5 já existentes)
5. US4 (Unidades) → testar independentemente (com famílias Aprovadas de US1)
6. US5 (Mutirão) → testar independentemente (com famílias de US1)
7. US6 (Dashboard) → testar independentemente (agrega US1/US2/US5)
8. Cada história agrega valor sem quebrar as anteriores

### Parallel Team Strategy

Com múltiplos desenvolvedores, após Setup + Foundational:

- Desenvolvedor A: US1 (Famílias) → depois US4 (Unidades, depende de US1)
- Desenvolvedor B: US2 (Obra) → depois US3 (Página Pública, consome US2/US5)
- Desenvolvedor C: US5 (Mutirão, depende de famílias de US1 para presença) → depois US6
  (Dashboard, agrega tudo)

---

## Notes

- [P] = arquivos diferentes, sem dependência entre as tarefas
- [Story] mapeia a tarefa à user story correspondente para rastreabilidade
- Nenhuma tarefa de teste automatizado foi gerada por história — não solicitado em spec.md;
  a infraestrutura de testes (xUnit/Vitest) é configurada no Setup para uso futuro
- EF Core MUST NOT executar migrations — todo changelog de schema é Liquibase (constitution
  Princípio III)
- Todas as PKs/FKs são UUID (`Guid`/`gen_random_uuid()`)
- Commit após cada tarefa ou grupo lógico de tarefas
- Parar em qualquer checkpoint para validar a história isoladamente
- Evitar: tarefas vagas, conflito no mesmo arquivo, dependências entre histórias que quebrem
  a independência
