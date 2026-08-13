# Implementation Plan: MLBEntidades — Plataforma de Gestão MCMV Entidades

**Branch**: `main` (submodule branch `001-mlbentidades-gestao` foi renomeado para `main`
durante a configuração inicial do repositório — ver nota em Project Structure) | **Date**: 2026-08-13 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-mlbentidades-gestao/spec.md`

## Summary

Construir o MLBEntidades: uma área pública (landing page sem autenticação com status
da obra, medições aprovadas e calendário de mutirões) e uma área administrativa
autenticada com RBAC de três perfis (AdminGeral, AssistenteSocial, TecnicoObra)
cobrindo gestão de famílias beneficiárias (cadastro, documentação, workflow de status),
unidades habitacionais (cadastro e atribuição), controle de obra (etapas, medições,
ocorrências), mutirões (escalas, presença, pontuação) e um dashboard agregado filtrado
por perfil. Backend em .NET 8 / ASP.NET Core Web API com Clean Architecture
(Api → Application → Domain ← Infrastructure), PostgreSQL 16 com schema
gerenciado exclusivamente via Liquibase, EF Core como ORM de escrita/mapeamento e
Dapper para leituras agregadas. Frontend em Vue 3 + Vite + TypeScript + Vuetify 3, com
Pinia para estado global e composables por domínio para chamadas HTTP.

## Technical Context

**Language/Version**: C# / .NET 8 (backend); TypeScript 5 com Vue 3 (frontend)

**Primary Dependencies**:
- Backend: ASP.NET Core Web API, EF Core 8 (Npgsql), Dapper, ASP.NET Core Identity,
  JWT Bearer, FluentValidation, Swashbuckle (Swagger/OpenAPI), Liquibase (CLI/Docker,
  fora do processo da aplicação)
- Frontend: Vue 3, Vite, Vuetify 3, @mdi/font, Pinia, Vue Router 4, Axios

**Storage**: PostgreSQL 16, schema versionado por Liquibase (`servico/db/`); upload de
documentos em disco local via `IDocumentoStorageService` (research.md §5)

**Testing**: xUnit + FluentAssertions + Moq (`ServicoMLBEntidades.UnitTests`,
research.md §3); Vitest + @vue/test-utils no frontend (research.md §4)

**Target Platform**: Web — API HTTP servida via ASP.NET Core (Linux/Windows server) e
SPA servida como estático/Vite build, consumida em navegador

**Project Type**: Web (monorepo com dois submodules Git: `servico/` backend,
`frontend/` frontend)

**Performance Goals**: Sem SLA formal; dimensionado para uso de um único
empreendimento habitacional (dezenas a poucas centenas de famílias, <50 usuários
administrativos simultâneos) com respostas interativas padrão de aplicação web
(research.md §9)

**Constraints**: EF Core MUST NOT executar migrations (`Database.Migrate()`/
`EnsureCreated()` proibidos); schema exclusivamente via Liquibase; todos os IDs UUID;
CORS restrito a origins explícitas (dev `http://localhost:5173` + origin de produção
via configuração); área pública sem autenticação, área administrativa sempre protegida
por JWT + role

**Scale/Scope**: 6 user stories (P1–P6), ~14 grupos de endpoints REST, 13 entidades de
domínio (research.md, data-model.md)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Princípio | Status | Nota |
|---|---|---|
| I. Clean Architecture no Backend | PASS | Solution com projetos Api/Application/Domain/Infrastructure/UnitTests (ver Project Structure); Controllers apenas fazem binding/delegação |
| II. Contrato-First e Versionamento de API | PASS | `contracts/openapi.yaml` definido nesta fase, antes de qualquer implementação; todas as rotas sob `/api/v1/`; erros em `ProblemDetails` (RFC 7807) em todo o contrato |
| III. Identidade e Persistência (UUID, EF Core, Liquibase) | PASS | Todas as PKs/FKs em `uuid` (data-model.md); EF Core apenas ORM (sem migrations automáticas); schema 100% em `servico/db/` via Liquibase |
| IV. Segurança e Autorização (RBAC) | PASS | Três roles via ASP.NET Core Identity + `[Authorize(Roles = "...")]`; área pública (`/public/*`) sem autenticação, separada da área administrativa por layout/guard no frontend |
| V. Arquitetura de Componentes Frontend (Vue 3) | PASS | Composition API + `<script setup>`, Pinia por domínio, composables isolando chamadas HTTP (estrutura em Project Structure) |
| VI. Qualidade, Validação e Histórico de Commits | PASS | FluentValidation em todos os Commands/DTOs de entrada (contracts/openapi.yaml lista os Commands); commits atômicos é responsabilidade de processo, não de arquitetura — sem violação |

Nenhuma violação identificada — **Complexity Tracking não se aplica** (seção omitida
por não haver desvios a justificar).

## Project Structure

### Documentation (this feature)

```text
specs/001-mlbentidades-gestao/
├── plan.md              # Este arquivo
├── research.md          # Fase 0 — decisões técnicas
├── data-model.md         # Fase 1 — entidades e regras
├── quickstart.md         # Fase 1 — como rodar e validar
├── contracts/
│   └── openapi.yaml      # Fase 1 — contrato REST completo
└── tasks.md              # Fase 2 — gerado por /speckit-tasks (não criado aqui)
```

### Source Code (repository root)

Monorepo com dois submodules Git independentes (`.gitmodules`), cada um com seu próprio
histórico e repositório remoto. **Nota sobre branch**: a branch de feature
`001-mlbentidades-gestao` criada para este trabalho foi renomeada para `main` no
repositório principal durante a configuração inicial (ver `git reflog`); os artefatos
desta feature permanecem em `specs/001-mlbentidades-gestao/` independentemente do nome
da branch, conforme `.specify/feature.json`.

```text
GestaoEntidades/                          # raiz do monorepo (workspace do spec-kit)
├── .specify/
├── specs/001-mlbentidades-gestao/        # esta feature
├── servico/                              # submodule — backend (.NET 8)
│   ├── ServicoMLBEntidades.sln
│   ├── ServicoMLBEntidades/              # camada API (já scaffolded — reaproveitada)
│   │   ├── Controllers/                  # Auth, Familias, Membros, Documentos,
│   │   │                                 # Unidades, Obra, Mutirao, Dashboard, Public
│   │   ├── Program.cs                    # DI, middlewares, CORS, JWT, Swagger
│   │   └── appsettings*.json
│   ├── ServicoMLBEntidades.Application/  # Services, Commands/Responses, Validators
│   │   ├── Familias/
│   │   ├── Membros/
│   │   ├── Documentos/
│   │   ├── Unidades/
│   │   ├── Obra/
│   │   ├── Mutirao/
│   │   ├── Dashboard/
│   │   └── Public/
│   ├── ServicoMLBEntidades.Domain/       # Entidades, Enums, interfaces de repositório
│   ├── ServicoMLBEntidades.Infrastructure/ # DbContext (EF Core), Repositories, Dapper
│   │   │                                  # queries, Identity/JWT, IDocumentoStorageService
│   ├── ServicoMLBEntidades.UnitTests/    # xUnit — testes dos Services
│   └── db/                               # projeto Liquibase (research.md §2)
│       └── changelog/
│           ├── db.changelog-master.xml
│           └── migrations/v1.0.0/*.sql
└── frontend/                              # submodule — frontend (Vue 3)
    ├── src/
    │   ├── assets/
    │   ├── components/
    │   │   ├── common/
    │   │   ├── admin/
    │   │   └── public/
    │   ├── composables/    # useAuth, useFamilias, useObra, useMutirao, useUnidades, useDashboard
    │   ├── layouts/
    │   │   ├── AdminLayout.vue
    │   │   └── PublicLayout.vue
    │   ├── pages/
    │   │   ├── public/
    │   │   └── admin/       # familias, obra, unidades, mutirao, dashboard
    │   ├── router/          # rotas + navigation guards
    │   ├── services/        # familiaService.ts, authService.ts, obraService.ts, ...
    │   ├── stores/           # authStore, familiasStore, obraStore, mutiraoStore (Pinia)
    │   └── types/            # interfaces TS por domínio (geradas a partir do contrato)
    └── vite.config.ts
```

**Structure Decision**: Web application com backend e frontend em submodules Git
separados (`servico/`, `frontend/`), cada um mantendo sua própria solution/projeto já
inicializado. O backend adota Clean Architecture com 5 projetos na mesma solution
(`ServicoMLBEntidades.sln`); o projeto de API pré-existente (`ServicoMLBEntidades`) é
reaproveitado como a camada Api em vez de recriado (research.md §1), e o Liquibase vive
em `servico/db/` em vez de `/backend/db/`, refletindo que `servico/` é a raiz real do
backend neste monorepo (research.md §2). O frontend segue a estrutura de pastas
solicitada, adicionando ao scaffold padrão do Vite as dependências ainda não instaladas
(Vuetify, Pinia, Vue Router, Axios, @mdi/font) na fase de implementação.

## Complexity Tracking

*Não aplicável — nenhuma violação da constituição identificada no Constitution Check.*
