# Research & Technical Decisions: MLBEntidades

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Este documento resolve as lacunas técnicas do Technical Context do plano, reconciliando
o stack solicitado com o estado real do monorepo (submodules `servico/` e `frontend/`
já inicializados com scaffolds padrão) e com a [constitution.md](../../.specify/memory/constitution.md).

## 1. Nomenclatura dos projetos do backend

**Decision**: Manter o nome já existente `ServicoMLBEntidades` para a solution e para o
projeto de API (`servico/ServicoMLBEntidades.sln`, projeto `ServicoMLBEntidades` com
`Controllers/` e `Program.cs`). Os novos projetos de camada são adicionados como
irmãos na mesma solution: `ServicoMLBEntidades.Application`,
`ServicoMLBEntidades.Domain`, `ServicoMLBEntidades.Infrastructure`,
`ServicoMLBEntidades.UnitTests`.

**Rationale**: O submodule já está inicializado e versionado no GitHub como
`ServicoMLBEntidades` (`.gitmodules` → `url = .../ServicoMLBEntidades`), com um projeto
de API scaffolded (`dotnet new webapi`). Renomear tudo para o prefixo `MLBEntidades.*`
pedido originalmente exigiria mover/renomear arquivos, namespaces e o próprio repositório
remoto sem nenhum ganho funcional. Preservar o nome existente elimina esse retrabalho e
mantém consistência entre nome do repositório, da solution e dos assemblies.

**Alternatives considered**: Renomear todo o projeto para `MLBEntidades.*` conforme o
texto original do stack — rejeitado por causar renomeação destrutiva de um projeto já
versionado sem benefício técnico.

## 2. Caminho do projeto Liquibase

**Decision**: O projeto Liquibase vive em `servico/db/`, com
`servico/db/changelog/db.changelog-master.xml` e changelogs SQL organizados em
`servico/db/changelog/migrations/v1.0.0/`.

**Rationale**: O contexto de repositório desta feature define `servico/` como a raiz do
backend (é o submodule real, mapeado em `.gitmodules`). O caminho `/backend/db/` citado
no stack não existe neste monorepo — é reconciliado para `/servico/db/`.

**Alternatives considered**: Criar uma pasta `backend/` adicional na raiz do
monorepo apenas para hospedar `db/` — rejeitado por duplicar a raiz do backend em dois
lugares (`servico/` e `backend/`) e confundir onde o código-fonte da API realmente mora.

## 3. Framework de testes (backend)

**Decision**: xUnit + FluentAssertions + Moq para `ServicoMLBEntidades.UnitTests`.

**Rationale**: xUnit é o padrão de fato para soluções .NET 8 novas, integra nativamente
com `dotnet test` e com o SDK de testes do Visual Studio/CI. FluentAssertions melhora a
legibilidade das asserções sobre DTOs/Services; Moq cobre o mock de interfaces de
repositório (Domain) nos testes de Service (Application). A constituição exige apenas
que exista uma camada de testes unitários da Application — não prescreve framework.

**Alternatives considered**: NUnit (equivalente, porém menos comum em scaffolds .NET 8
recentes), MSTest (sintaxe de asserção menos expressiva).

## 4. Framework de testes (frontend)

**Decision**: Vitest + @vue/test-utils para testes de composables e componentes.

**Rationale**: O projeto já usa Vite; Vitest reaproveita a mesma configuração/transform
pipeline, é rápido e tem suporte de primeira classe para Vue 3 + `<script setup>` + TS.

**Alternatives considered**: Jest — exigiria configuração extra de transform para
ESM/Vite; Cypress component testing — mais pesado, reservado para uma futura suíte E2E
se necessário, fora do escopo desta fase.

## 5. Armazenamento de documentos (upload de RG/CPF/comprovantes/certidões)

**Decision**: Armazenamento em disco local, em um diretório configurável via
`appsettings` (ex.: `Storage:DocumentosPath`), com o caminho relativo/identificador do
arquivo persistido no campo `ArquivoPath` da entidade `Documento`. Validação de tipo
MIME e tamanho máximo via FluentValidation antes da gravação.

**Rationale**: Nenhum provedor de armazenamento em nuvem foi especificado pelo usuário
ou pela constituição; disco local mantém o MVP simples para um sistema autogerido por um
movimento social sem orçamento de infraestrutura cloud assumido. O acesso a arquivos é
isolado detrás de uma interface `IDocumentoStorageService` na camada Infrastructure,
permitindo trocar para blob storage (Azure/S3-compatible) depois sem alterar Domain ou
Application.

**Alternatives considered**: Azure Blob Storage / S3-compatible desde o início —
adiado por não haver requisito ou orçamento declarado; BLOB no PostgreSQL — rejeitado
por inflar o banco de dados e conflitar com o gerenciamento de schema exclusivo via
Liquibase para dados estruturados.

## 6. Papel do Dapper ao lado do EF Core

**Decision**: Dapper é usado exclusivamente para consultas de leitura otimizadas e
agregadas: dashboard (FR-029/FR-030), relatório de pontuação e participação (FR-027) e
mapa de ocupação de unidades (FR-019). Todas as escritas e leituras simples de entidade
usam EF Core.

**Rationale**: A constituição (Princípio III) exige que o EF Core seja usado
exclusivamente como ORM (nunca para gerar/aplicar migrations) — isso não impede uma
segunda ferramenta de leitura. Delimitar o uso do Dapper a consultas agregadas evita
ambiguidade sobre "quando usar qual ferramenta" e aproveita sua performance em queries
com múltiplos `JOIN`/agregações sem duplicar lógica de mapeamento de entidades.

**Alternatives considered**: `FromSqlRaw` do próprio EF Core para as mesmas consultas —
rejeitado porque o stack do usuário já nomeia Dapper explicitamente como dependência.

## 7. Rotação de refresh token

**Decision**: Cada uso do endpoint `/api/v1/auth/refresh-token` invalida o refresh token
anterior e emite um novo (rotação), mantendo a expiração de 7 dias por token emitido. O
access token JWT expira em 8 horas, conforme especificado.

**Rationale**: Dados sensíveis de famílias (renda, vulnerabilidade, documentos) ficam
acessíveis a qualquer perfil autenticado (FR-031); rotação de refresh token é prática
padrão para reduzir o risco de replay caso um refresh token seja vazado, alinhado ao
Princípio IV da constituição (segurança e autorização).

**Alternatives considered**: Refresh token estático reutilizável até expirar — rejeitado
por oferecer postura de segurança mais fraca sem ganho de simplicidade relevante.

## 8. CORS

**Decision**: Política de CORS nomeada permitindo apenas as origins explícitas
`http://localhost:5173` (desenvolvimento) e uma origin de produção lida de
configuração (`Cors:ProductionOrigin` em `appsettings`/variável de ambiente), nunca
wildcard `*` combinado com credenciais.

**Rationale**: O endpoint expõe cookies/headers de autorização; a origin de produção
ainda não está definida (domínio final do movimento social), então fica em configuração
em vez de hardcoded.

## 9. Escopo de performance e escala

**Decision**: Sem SLA formal de performance; dimensionar para o uso real esperado — um
único empreendimento habitacional, com dezenas a poucas centenas de famílias
cadastradas e menos de 50 usuários administrativos simultâneos. Respostas interativas
padrão de aplicação web (sem processamento em lote pesado).

**Rationale**: O spec não define metas de performance (ver Assumptions em spec.md); o
domínio (autogestão de um único bairro) não sugere escala além disso. Documentado como
suposição, não como requisito rígido — pode ser revisitado se o movimento social
replicar o sistema para múltiplos empreendimentos no futuro (fora do escopo atual).

## 10. Lacuna nos endpoints públicos

**Decision**: Além de `/api/v1/public/status` e `/api/v1/public/etapas` (já listados no
stack), adicionar `/api/v1/public/medicoes` (histórico de medições aprovadas, FR-003) e
`/api/v1/public/mutiroes` (calendário de próximas atividades/mutirões, FR-004).

**Rationale**: A lista de endpoints fornecida pelo usuário não cobria explicitamente
FR-003 e FR-004 da especificação (histórico de medições aprovadas e calendário de
mutirões na área pública) — sem esses dois endpoints, User Story 3 (página pública) não
poderia ser implementada por completo. Adicionados para fechar essa lacuna sem alterar
nenhum dos endpoints já especificados.
