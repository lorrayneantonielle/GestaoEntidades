# Quickstart: MLBEntidades

**Plano**: [plan.md](./plan.md) | **Contrato**: [contracts/openapi.yaml](./contracts/openapi.yaml)

## 1. Clonar com submodules

```bash
git clone --recurse-submodules <url-do-monorepo>
# ou, se já clonado sem submodules:
git submodule update --init --recursive
```

## 2. Banco de dados e schema (Liquibase)

```bash
# Subir PostgreSQL 16 localmente (exemplo via Docker)
docker run --name mlbentidades-db -e POSTGRES_PASSWORD=devpassword \
  -e POSTGRES_DB=mlbentidades -p 5432:5432 -d postgres:16

# Aplicar changelogs (a partir de servico/db/)
cd servico/db
liquibase --changelog-file=changelog/db.changelog-master.xml \
  --url="jdbc:postgresql://localhost:5432/mlbentidades" \
  --username=postgres --password=devpassword update
```

A aplicação .NET **nunca** chama `Database.Migrate()` ou `EnsureCreated()` — o schema
já deve existir antes do primeiro `dotnet run` (Princípio III da constituição).

## 3. Backend (`servico/`)

```bash
cd servico
dotnet restore ServicoMLBEntidades.sln
dotnet build ServicoMLBEntidades.sln
dotnet run --project ServicoMLBEntidades
```

- Swagger/OpenAPI disponível em `https://localhost:<porta>/swagger`, com suporte a
  autenticação JWT (botão "Authorize").
- Configurar `appsettings.Development.json` com a connection string do PostgreSQL,
  segredo JWT e `Storage:DocumentosPath` (research.md §5).

## 4. Frontend (`frontend/`)

```bash
cd frontend
npm install
npm run dev
```

- Aplicação disponível em `http://localhost:5173`.
- Configurar a baseURL da API (ex.: `.env.local` com `VITE_API_BASE_URL=https://localhost:<porta>/api/v1`).

## 5. Smoke test — fluxo mínimo (User Story 1)

1. `POST /api/v1/auth/login` com um usuário seed `AssistenteSocial` → obter `accessToken`.
2. `POST /api/v1/familias` com `rendaFamiliar`, `situacaoVulnerabilidade` e ao menos um
   membro em `membros[]` → família criada com `status = PreCadastro`.
3. `POST /api/v1/documentos` (multipart) para cada tipo obrigatório (`RG`, `CPF`,
   `ComprovanteRenda`, `Certidao`) da família criada, cada um com `status = Validado`.
4. `PATCH /api/v1/familias/{id}/status` com `novoStatus = EmAnalise` → deve ter sucesso
   somente após o passo 3 (FR-011).
5. Repetir o PATCH para `Aprovada`.
6. Acessar `GET /api/v1/public/status` sem token → deve responder 200 sem autenticação.

Esse fluxo cobre o teste independente descrito na User Story 1 do spec e valida a
cadeia contrato → autorização → workflow de status ponta a ponta.

## 6. Comandos úteis

| Comando | Descrição |
|---|---|
| `dotnet test servico/ServicoMLBEntidades.sln` | Executa `ServicoMLBEntidades.UnitTests` (xUnit) |
| `npm run type-check` (em `frontend/`) | Checagem de tipos TypeScript |
| `npm run test:unit` (em `frontend/`, a adicionar) | Executa testes Vitest |
| `liquibase status` (em `servico/db/`) | Lista changelogs pendentes |
