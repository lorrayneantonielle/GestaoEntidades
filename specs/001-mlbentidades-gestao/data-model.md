# Data Model: MLBEntidades

**Feature**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

Todas as chaves primárias e estrangeiras são `UUID` (`Guid` no EF Core,
`gen_random_uuid()` como default no Liquibase), conforme Princípio III da constituição.

## Familia

Representa um núcleo familiar beneficiário (spec: Key Entities → Família).

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK |
| RendaFamiliar | decimal(12,2) | obrigatório, ≥ 0 |
| SituacaoVulnerabilidade | text | obrigatório |
| Status | enum | `PreCadastro`, `EmAnalise`, `Aprovada`, `UnidadeAtribuida`, `EmConstrucao`, `Finalizada` (FR-010) |
| PontuacaoAcumulada | int | default 0, soma das `Presenca.PontuacaoConcedida` (FR-026) |
| CreatedAt / UpdatedAt | timestamptz | auditoria |

**Relacionamentos**: 1—N `Membro`, 1—N `Documento`, 1—N `FamiliaStatusHistorico`,
0—1 `UnidadeHabitacional` (unidade atribuída), 1—N `Presenca`.

**Transições de estado** (FR-010, FR-011, FR-012, FR-013):
- Avanço sequencial: `PreCadastro → EmAnalise → Aprovada → UnidadeAtribuida → EmConstrucao → Finalizada`.
- `PreCadastro → EmAnalise` bloqueado enquanto houver `Documento` obrigatório com
  `Status != Validado` (FR-011).
- Reversão/reprovação permitida a partir de qualquer status, para um status anterior
  válido, exigindo `Motivo` (FR-012) — gera um registro em `FamiliaStatusHistorico`.
- Se a família revertida possuía `UnidadeHabitacional` atribuída, a unidade volta para
  `Status = Livre` e seu `FamiliaId` é limpo (FR-013).
- `EmConstrucao → Finalizada` é uma marcação manual (Técnico de Obra ou Admin), sem
  regra automática vinculada a percentual de etapas (FR-032, clarificação).

## Membro

Pessoa que compõe o núcleo familiar (spec: atributo "composição familiar"; exposto como
recurso próprio via `/api/v1/membros`).

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK |
| FamiliaId | uuid | FK → Familia, obrigatório |
| Nome | text | obrigatório |
| DataNascimento | date | obrigatório |
| Vinculo | enum/text | parentesco com o responsável (ex.: Cônjuge, Filho(a), Dependente) |
| Cpf | text(11) | obrigatório, formato validado |

**Regra de unicidade** (FR-033): `Cpf` é único entre todos os `Membro` cujas famílias
não estejam com exclusão lógica (soft delete). Implementado como índice único
condicional (`WHERE familia_excluida = false`) no changelog Liquibase. Violação retorna
`ProblemDetails` 409 identificando a família com o CPF já cadastrado.

## Documento

Item de documentação obrigatória vinculado a uma família (spec: Key Entities →
Documento).

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK |
| FamiliaId | uuid | FK → Familia, obrigatório |
| Tipo | enum | `RG`, `CPF`, `ComprovanteRenda`, `Certidao` |
| Status | enum | `Pendente`, `Recebido`, `Validado` |
| ArquivoPath | text | nullable até o upload; caminho relativo no storage local (research.md §5) |
| ArquivoMimeType | text | nullable, preenchido no upload |
| UpdatedAt | timestamptz | auditoria |

## FamiliaStatusHistorico

Trilha de auditoria de transições de status (spec: Key Entities → Família, "histórico
de transições de status").

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK |
| FamiliaId | uuid | FK → Familia |
| StatusAnterior | enum | mesmo domínio de `Familia.Status` |
| StatusNovo | enum | mesmo domínio de `Familia.Status` |
| Motivo | text | obrigatório apenas quando `StatusNovo` representa reversão/reprovação (FR-012) |
| UsuarioId | uuid | FK → AspNetUsers, quem executou a transição |
| DataTransicao | timestamptz | obrigatório |

## UnidadeHabitacional

Lote/unidade do empreendimento (spec: Key Entities → Unidade Habitacional).

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK |
| Identificador | text | único (FR-015) |
| Metragem | decimal(8,2) | obrigatório, > 0 |
| LocalizacaoTerreno | text | coordenadas/descrição de posição no terreno |
| Status | enum | `Livre`, `Reservada`, `Ocupada` (FR-019) |
| FamiliaId | uuid | nullable, FK → Familia; obrigatório quando `Status != Livre` |

**Regras**:
- Atribuição só é permitida quando `Familia.Status ∈ {Aprovada, UnidadeAtribuida, EmConstrucao, Finalizada}` (FR-016).
- `UnidadeHabitacional` com `Status ∈ {Reservada, Ocupada}` não pode ser atribuída a
  outra família (FR-017) — enforced por transação + índice único parcial em
  `FamiliaId` para unidades não-`Livre`.
- Ao atribuir, `Familia.Status` MUST avançar para `UnidadeAtribuida` (FR-018).

## EtapaObra

Fase da construção (spec: Key Entities → Etapa da Obra).

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK |
| Nome | text | obrigatório (ex.: fundação, estrutura, alvenaria, cobertura, acabamento) |
| Ordem | int | único, define sequência de exibição |
| PercentualConclusao | decimal(5,2) | 0–100, default 0 (FR-021) |

**Relacionamentos**: 1—N `Medicao`, 1—N `Ocorrencia`.

## Medicao

Medição de progresso do governo, registrada manualmente (spec: Key Entities →
Medição; research.md §10).

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK |
| EtapaObraId | uuid | FK → EtapaObra, obrigatório |
| Data | date | obrigatório |
| StatusAprovacao | enum | `Aprovada`, `Pendente`, `Rejeitada` |
| RecursosLiberados | decimal(14,2) | nullable, preenchido quando `StatusAprovacao = Aprovada` |
| Observacao | text | nullable |
| RegistradoPorUsuarioId | uuid | FK → AspNetUsers |

**Regra de visibilidade pública**: apenas `Medicao.StatusAprovacao = Aprovada` é
exposta em `/api/v1/public/medicoes` (FR-003, spec Assumptions).

## Ocorrencia

Observação técnica vinculada a uma etapa (spec: Key Entities → Ocorrência).

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK |
| EtapaObraId | uuid | FK → EtapaObra, obrigatório |
| Descricao | text | obrigatório |
| Data | date | obrigatório |
| RegistradoPorUsuarioId | uuid | FK → AspNetUsers |

## MutiraoEscala

Convocação de trabalho comunitário (spec: Key Entities → Mutirão/Escala).

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK |
| Data | date | obrigatório |
| Turno | enum | `Manha`, `Tarde`, `Integral` |
| VagasTotais | int | obrigatório, > 0 — número de vagas (famílias) da escala |
| PontuacaoPorPresenca | int | obrigatório, > 0 — definido na criação, pode variar por escala (clarificação) |

**Relacionamentos**: 1—N `Presenca` (limitado a `VagasTotais` registros, FR-025).

## Presenca

Registro de participação de uma família em um mutirão específico (spec: Key Entities →
Presença).

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK |
| MutiraoEscalaId | uuid | FK → MutiraoEscala, obrigatório |
| FamiliaId | uuid | FK → Familia, obrigatório |
| DataRegistro | timestamptz | obrigatório |
| PontuacaoConcedida | int | copiado de `MutiraoEscala.PontuacaoPorPresenca` no momento do registro (histórico imutável mesmo se a escala for editada depois) |

**Regras**: par (`MutiraoEscalaId`, `FamiliaId`) único (uma família não registra presença
duplicada na mesma escala); contagem de `Presenca` por `MutiraoEscalaId` MUST NOT
exceder `MutiraoEscala.VagasTotais` (FR-025).

## ConfiguracaoSistema

Configurações globais ajustáveis pelo Administrador Geral (spec: Assumptions —
limite mínimo de pontuação configurável).

| Campo | Tipo | Regras |
|---|---|---|
| Id | uuid | PK (linha única / singleton) |
| LimiteMinimoPontuacaoMutirao | int | usado por FR-028 para sinalizar baixa participação |

## Usuario / Role / RefreshToken (ASP.NET Core Identity)

| Entidade | Campos principais | Notas |
|---|---|---|
| `AspNetUsers` (Usuario) | Id (uuid), NomeCompleto, Email, PasswordHash | gerenciado pelo Identity |
| `AspNetRoles` | `AdminGeral`, `AssistenteSocial`, `TecnicoObra` | seed via Liquibase changelog |
| `RefreshToken` | Id, UsuarioId, TokenHash, ExpiresAt (7 dias), RevokedAt, CreatedAt | rotação a cada uso (research.md §7) |

## Diagrama de relacionamento (visão lógica)

```text
Familia 1───N Membro
Familia 1───N Documento
Familia 1───N FamiliaStatusHistorico
Familia 0..1───1 UnidadeHabitacional
Familia 1───N Presenca N───1 MutiraoEscala

EtapaObra 1───N Medicao
EtapaObra 1───N Ocorrencia

AspNetUsers 1───N RefreshToken
AspNetUsers 1───N FamiliaStatusHistorico (UsuarioId)
AspNetUsers 1───N Medicao (RegistradoPorUsuarioId)
AspNetUsers 1───N Ocorrencia (RegistradoPorUsuarioId)
```
