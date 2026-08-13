# Feature Specification: MLBEntidades — Plataforma de Gestão MCMV Entidades

**Feature Branch**: `001-mlbentidades-gestao`

**Created**: 2026-08-13

**Status**: Draft

**Input**: User description: "Desenvolver o MLBEntidades, uma plataforma de gestão para um movimento social contemplado no programa Minha Casa Minha Vida Entidades (MCMV Entidades). O movimento é responsável pela autogestão completa do empreendimento habitacional — desde o cadastro das famílias beneficiárias até o planejamento e execução da auto-construção do bairro. O sistema possui uma área pública (landing page informativa) e uma área administrativa autenticada com três perfis (Administrador Geral, Assistente Social, Técnico de Obra) cobrindo gestão de famílias, unidades habitacionais, controle de obra, mutirões/presença e dashboard."

## Clarifications

### Session 2026-08-13

- Q: Quem deve poder visualizar dados sensíveis da família (renda, vulnerabilidade social, documentos como RG/CPF)? → A: Visível a todos os perfis autenticados; apenas a edição/cadastro desses dados é restrita por perfil.
- Q: O registro de uma medição do governo como "aprovada" vem de integração automática com sistema externo do governo, ou é registro manual da equipe? → A: Registro manual — Técnico de Obra ou Administrador Geral registra a medição e seu status de aprovação com base em comunicação oficial recebida fora do sistema; não há integração automática com sistemas do governo nesta versão.
- Q: O que dispara a transição do status da família de "Em Construção" para "Finalizada"? → A: Marcação manual — Técnico de Obra ou Administrador Geral marca a família como "Finalizada" quando considerar sua unidade pronta para entrega; não há regra automática vinculada a percentuais de etapa.
- Q: O sistema deve impedir o cadastro de uma família cujo responsável/membro já tenha CPF cadastrado em outra família? → A: Bloquear duplicidade — o sistema impede o cadastro/edição quando o CPF já estiver vinculado a outra família ativa.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Cadastro e Acompanhamento de Famílias Beneficiárias (Priority: P1)

Como Assistente Social, quero cadastrar as famílias beneficiárias com sua composição
familiar, renda e situação de vulnerabilidade, controlar a documentação exigida e
acompanhar cada família ao longo do workflow de status, para que o movimento saiba
exatamente em que estágio do processo cada família se encontra e possa agir sobre
pendências de documentação ou elegibilidade.

**Why this priority**: É a base de todo o sistema — nenhum outro módulo (atribuição de
unidade, mutirão, dashboard) tem sentido sem famílias cadastradas e classificadas.
Entrega valor imediato mesmo isoladamente: substitui controle manual/planilhas por um
cadastro único e rastreável.

**Independent Test**: Pode ser testado cadastrando uma família com composição familiar,
renda e documentos; avançando e revertendo seu status pelo workflow completo; e
buscando/filtrando a listagem por status, nome e número de membros — sem depender de
nenhum outro módulo.

**Acceptance Scenarios**:

1. **Given** que o Assistente Social está autenticado, **When** ele cadastra uma nova
   família informando composição familiar (membros, idades, vínculos), renda familiar e
   situação de vulnerabilidade social, **Then** a família é criada com status
   "Pré-cadastro".
2. **Given** uma família em "Pré-cadastro" com documentação obrigatória pendente,
   **When** o Assistente Social registra o recebimento de cada documento (RG, CPF,
   comprovante de renda, certidões), **Then** o sistema exibe o status de completude
   documental da família.
3. **Given** uma família com documentação completa, **When** o Assistente Social ou
   Administrador Geral avança seu status para "Em Análise" e depois "Aprovada",
   **Then** o sistema registra a transição e a família passa a poder receber unidade
   habitacional.
4. **Given** uma família em qualquer etapa do workflow, **When** o Assistente Social ou
   Administrador Geral reprova ou reverte o status da família registrando um motivo,
   **Then** o sistema aplica a reversão e mantém o motivo associado ao histórico da
   família.
5. **Given** uma lista de famílias cadastradas em diferentes status, **When** um usuário
   filtra por status, nome ou número de membros, **Then** apenas as famílias que
   atendem aos critérios são exibidas.

---

### User Story 2 - Controle da Obra por Etapas e Medições (Priority: P2)

Como Técnico de Obra, quero cadastrar as etapas da construção, registrar o percentual
de conclusão de cada uma, controlar as medições do governo e registrar ocorrências
técnicas, para que o avanço físico do empreendimento seja acompanhado de forma
confiável e vinculado à liberação de recursos.

**Why this priority**: É o segundo pilar do sistema e não depende do cadastro de
famílias para existir — pode ser implantado e validado de forma independente, e é
pré-requisito para a área pública (status da obra) e para o dashboard.

**Independent Test**: Pode ser testado cadastrando etapas da obra (ex.: fundação,
estrutura, alvenaria, cobertura, acabamento), atualizando o percentual de conclusão de
cada uma, registrando uma medição vinculada a uma etapa e adicionando uma ocorrência
técnica — de forma isolada dos demais módulos.

**Acceptance Scenarios**:

1. **Given** que o Técnico de Obra está autenticado, **When** ele cadastra uma nova
   etapa da obra com nome e ordem, **Then** a etapa passa a existir com percentual de
   conclusão inicial igual a zero.
2. **Given** uma etapa existente, **When** o Técnico de Obra atualiza seu percentual de
   conclusão, **Then** o novo percentual é refletido no status geral da obra.
3. **Given** uma etapa com percentual de conclusão registrado, **When** o Técnico de
   Obra ou Administrador Geral registra uma medição do governo vinculada a essa etapa,
   **Then** a medição é associada à etapa e sinalizada como aprovada, validando a
   conclusão informada.
4. **Given** uma etapa em andamento, **When** o Técnico de Obra registra uma ocorrência
   ou observação técnica, **Then** a ocorrência é associada à etapa e ao registro
   histórico da obra.

---

### User Story 3 - Página Pública de Acompanhamento do Empreendimento (Priority: P3)

Como membro da comunidade ou beneficiário, quero acessar uma página pública sem
necessidade de login para acompanhar informações gerais do empreendimento, o status
atual da obra por etapa, o histórico de medições aprovadas pelo governo e o calendário
de próximas atividades e mutirões, para me manter informado sobre o andamento do
projeto sem precisar de credenciais de acesso.

**Why this priority**: Depende de dados já existentes de obra (US2) e mutirões (US5)
para ter conteúdo relevante, mas é a interface de maior visibilidade externa e
transparência para a comunidade — entregue após os módulos administrativos que a
alimentam terem dados mínimos publicáveis.

**Independent Test**: Pode ser testado acessando a landing page sem autenticação e
verificando que as informações gerais do empreendimento, o progresso por etapa, o
histórico de medições aprovadas e o calendário de mutirões futuros são exibidos
corretamente com os dados publicados pela área administrativa.

**Acceptance Scenarios**:

1. **Given** um visitante sem autenticação, **When** ele acessa a landing page,
   **Then** o sistema exibe informações gerais sobre o empreendimento e o movimento
   social sem exigir login.
2. **Given** etapas de obra com percentuais de conclusão registrados, **When** o
   visitante acessa a seção de status da obra, **Then** o progresso por etapa é
   exibido de forma consolidada e atualizada.
3. **Given** medições aprovadas registradas na área administrativa, **When** o
   visitante acessa o histórico de medições, **Then** apenas as medições aprovadas
   pelo governo são listadas, em ordem cronológica.
4. **Given** mutirões futuros cadastrados, **When** o visitante acessa o calendário de
   atividades, **Then** as próximas datas, turnos e temas dos mutirões são exibidos.

---

### User Story 4 - Gestão de Unidades Habitacionais e Atribuição a Famílias (Priority: P4)

Como Administrador Geral ou Técnico de Obra, quero cadastrar as unidades/lotes do
empreendimento, atribuir unidades a famílias aprovadas e visualizar o mapa de ocupação
do terreno, para controlar com precisão quais famílias ocuparão qual unidade e evitar
conflitos de atribuição.

**Why this priority**: Depende de famílias já aprovadas (US1) para ter sentido de
negócio completo, por isso vem depois — mas o cadastro de unidades em si (sem
atribuição) pode ser validado de forma isolada.

**Independent Test**: Pode ser testado cadastrando unidades com identificador,
metragem e localização; atribuindo uma unidade a uma família com status "Aprovada"; e
visualizando o mapa de ocupação mostrando unidades livres, reservadas e ocupadas.

**Acceptance Scenarios**:

1. **Given** que o Administrador Geral ou Técnico de Obra está autenticado, **When**
   ele cadastra uma unidade habitacional com identificador, metragem e localização no
   terreno, **Then** a unidade é criada com status "Livre".
2. **Given** uma família com status "Aprovada" e uma unidade "Livre", **When** o
   usuário atribui a unidade à família, **Then** a unidade passa a status "Ocupada"
   (ou "Reservada", conforme o momento do processo) e a família avança para o status
   "Unidade Atribuída".
3. **Given** uma unidade já atribuída a uma família, **When** um usuário tenta atribuí-la
   a outra família, **Then** o sistema impede a operação e informa que a unidade não
   está disponível.
4. **Given** unidades em diferentes situações de ocupação, **When** um usuário acessa o
   mapa de ocupação, **Then** as unidades livres, reservadas e ocupadas são
   visualmente diferenciadas.

---

### User Story 5 - Mutirão e Sistema de Presença e Pontuação (Priority: P5)

Como Técnico de Obra ou Administrador Geral, quero criar escalas de mutirão com data,
turno e vagas por família, registrar a presença das famílias, conceder pontuação por
presença e identificar famílias com baixa participação, para incentivar o engajamento
da comunidade na auto-construção e permitir que o Assistente Social atue sobre famílias
pouco engajadas.

**Why this priority**: Depende de famílias cadastradas (US1) para registrar presença,
mas é um módulo funcionalmente independente dos demais (obra, unidades) e alimenta o
calendário público (US3) e o dashboard (US6).

**Independent Test**: Pode ser testado criando uma escala de mutirão com vagas
definidas, registrando presença de famílias participantes, verificando a pontuação
concedida conforme o valor configurado para aquele mutirão, e confirmando que uma
família cuja pontuação acumulada cai abaixo do limite mínimo é sinalizada para o
Assistente Social.

**Acceptance Scenarios**:

1. **Given** que o Técnico de Obra está autenticado, **When** ele cria uma escala de
   mutirão informando data, turno, número de vagas por família e a pontuação concedida
   por presença naquele mutirão, **Then** a escala é criada e disponibilizada para
   inscrição/registro de presença.
2. **Given** uma escala de mutirão criada, **When** o Técnico de Obra registra a
   presença de uma família participante, **Then** a família recebe a pontuação
   definida para aquele mutirão específico.
3. **Given** o histórico de pontuação de uma família, **When** a pontuação acumulada da
   família cai abaixo do limite mínimo configurado pelo Administrador Geral, **Then**
   o sistema sinaliza a família para acompanhamento do Assistente Social.
4. **Given** famílias com pontuação e participação registradas, **When** um usuário
   consulta o relatório de pontuação e participação, **Then** o relatório exibe o
   histórico de presenças e a pontuação acumulada por família.

---

### User Story 6 - Dashboard Administrativo por Perfil (Priority: P6)

Como usuário autenticado de qualquer perfil, quero visualizar um dashboard com a visão
geral do projeto — total de famílias por status, percentual de conclusão da obra e
próximos mutirões — filtrado de acordo com as responsabilidades do meu perfil, para
acompanhar rapidamente o andamento do projeto sem precisar navegar por cada módulo
individualmente.

**Why this priority**: É uma camada de agregação sobre os demais módulos; só entrega
valor completo quando os módulos que alimentam seus dados (famílias, obra, mutirões)
já existem, por isso é a última prioridade.

**Independent Test**: Pode ser testado autenticando com cada um dos três perfis e
verificando que o conteúdo do dashboard exibido corresponde às responsabilidades
daquele perfil (ex.: Assistente Social vê indicadores de famílias, Técnico de Obra vê
indicadores de obra e mutirão, Administrador Geral vê todos os indicadores).

**Acceptance Scenarios**:

1. **Given** um Administrador Geral autenticado, **When** ele acessa o dashboard,
   **Then** o sistema exibe total de famílias por status, percentual de conclusão da
   obra e próximos mutirões, sem restrições.
2. **Given** um Assistente Social autenticado, **When** ele acessa o dashboard,
   **Then** o sistema exibe indicadores relacionados a famílias (total por status,
   famílias sinalizadas por baixa participação) com visibilidade reduzida ou nula sobre
   indicadores puramente técnicos de obra.
3. **Given** um Técnico de Obra autenticado, **When** ele acessa o dashboard, **Then**
   o sistema exibe indicadores de obra (percentual de conclusão por etapa) e de
   mutirão (próximos mutirões, presença); o dashboard não agrega indicadores de
   renda/vulnerabilidade das famílias, embora o Técnico de Obra possa consultar esses
   dados diretamente no cadastro de uma família (FR-031).

---

### Edge Cases

- O que acontece quando uma família tenta avançar de status sem toda a documentação
  obrigatória registrada? O sistema deve impedir o avanço e indicar os documentos
  pendentes.
- Como o sistema trata a tentativa de atribuir uma unidade a uma família que não está
  com status "Aprovada" (ou posterior)? A operação deve ser bloqueada.
- O que acontece se o número de famílias que registram presença em um mutirão exceder o
  número de vagas definido na escala? O sistema deve impedir o registro de presença
  além das vagas disponíveis ou sinalizar a inconsistência ao Técnico de Obra.
- Como o sistema trata uma medição do governo registrada para uma etapa cujo percentual
  de conclusão ainda não foi atualizado? O sistema deve permitir o registro, mas
  destacar a divergência entre percentual informado e medição aprovada.
- O que acontece quando um usuário autenticado tenta acessar uma funcionalidade fora do
  escopo do seu perfil (ex.: Assistente Social tentando registrar percentual de obra)?
  O acesso deve ser negado com uma mensagem clara.
- Como a página pública trata a ausência de dados publicáveis (ex.: nenhuma medição
  aprovada ainda)? A seção correspondente deve exibir um estado vazio informativo, sem
  erro.
- O que acontece quando a reprovação/reversão de status de uma família ocorre após ela
  já ter uma unidade atribuída? O sistema deve liberar a unidade previamente atribuída
  de volta para o status "Livre" ao registrar a reversão.
- O que acontece quando o Assistente Social tenta cadastrar uma família com um CPF de
  membro que já pertence a outra família ativa? O sistema deve bloquear o cadastro e
  identificar a duplicidade.

## Requirements *(mandatory)*

### Functional Requirements

**Área Pública**

- **FR-001**: O sistema MUST permitir que visitantes não autenticados visualizem
  informações gerais sobre o empreendimento e o movimento social.
- **FR-002**: O sistema MUST exibir a visitantes não autenticados o status atual da
  obra com progresso consolidado por etapa.
- **FR-003**: O sistema MUST exibir a visitantes não autenticados o histórico de
  medições aprovadas pelo governo, em ordem cronológica.
- **FR-004**: O sistema MUST exibir a visitantes não autenticados um calendário das
  próximas atividades e mutirões planejados.

**Perfis e Autorização**

- **FR-005**: O sistema MUST suportar três perfis de acesso autenticado: Administrador
  Geral, Assistente Social e Técnico de Obra, cada um com permissões distintas.
- **FR-006**: O sistema MUST restringir o acesso a cada funcionalidade administrativa
  de acordo com o perfil do usuário autenticado, negando acesso e informando o motivo
  quando fora do escopo do perfil.
- **FR-007**: O Administrador Geral MUST ter acesso irrestrito a todas as
  funcionalidades administrativas do sistema.
- **FR-031**: Qualquer perfil autenticado (Administrador Geral, Assistente Social ou
  Técnico de Obra) MUST ser capaz de visualizar todos os dados de uma família,
  incluindo campos sensíveis (renda, situação de vulnerabilidade social, documentos);
  a restrição por perfil aplica-se apenas a operações de cadastro e edição desses
  dados, conforme definido em FR-008 a FR-014.

**Gestão de Famílias**

- **FR-008**: Assistente Social e Administrador Geral MUST ser capazes de cadastrar
  uma família com composição familiar (membros, idades, vínculos), renda familiar e
  situação de vulnerabilidade social.
- **FR-009**: O sistema MUST controlar a documentação obrigatória por família (RG,
  CPF, comprovante de renda, certidões), registrando o status de cada documento
  exigido.
- **FR-010**: O sistema MUST controlar o status da família segundo o workflow:
  Pré-cadastro → Em Análise → Aprovada → Unidade Atribuída → Em Construção →
  Finalizada.
- **FR-011**: O sistema MUST impedir o avanço de uma família para "Em Análise" ou
  etapa posterior enquanto houver documentação obrigatória pendente.
- **FR-012**: O sistema MUST permitir que Assistente Social ou Administrador Geral
  reprovem ou revertam o status de uma família em qualquer etapa do workflow,
  exigindo o registro de um motivo para a reversão.
- **FR-013**: Ao reverter o status de uma família que já possui unidade habitacional
  atribuída, o sistema MUST liberar essa unidade de volta para o status "Livre".
- **FR-032**: A transição de status de uma família de "Em Construção" para
  "Finalizada" MUST ser uma marcação manual feita por Técnico de Obra ou
  Administrador Geral, sem regra automática vinculada ao percentual de conclusão das
  etapas da obra.
- **FR-014**: O sistema MUST permitir a listagem e busca de famílias com filtros por
  status, nome e número de membros.
- **FR-033**: O sistema MUST impedir o cadastro ou edição de uma família cujo CPF de
  um membro já esteja vinculado a outra família ativa, retornando um erro que
  identifique a duplicidade.

**Gestão de Unidades Habitacionais**

- **FR-015**: Administrador Geral e Técnico de Obra MUST ser capazes de cadastrar
  unidades/lotes do empreendimento com identificador único, metragem e localização no
  terreno.
- **FR-016**: O sistema MUST permitir a atribuição de uma unidade a uma família apenas
  quando o status da família for "Aprovada" ou posterior no workflow.
- **FR-017**: O sistema MUST impedir a atribuição de uma unidade que já esteja
  reservada ou ocupada por outra família.
- **FR-018**: Ao atribuir uma unidade a uma família, o sistema MUST atualizar o status
  da família para "Unidade Atribuída".
- **FR-019**: O sistema MUST exibir um mapa de ocupação distinguindo unidades livres,
  reservadas e ocupadas.

**Controle da Obra**

- **FR-020**: Técnico de Obra e Administrador Geral MUST ser capazes de cadastrar as
  etapas da obra (ex.: fundação, estrutura, alvenaria, cobertura, acabamento) com nome
  e ordem de execução.
- **FR-021**: O sistema MUST permitir o registro e a atualização do percentual de
  conclusão de cada etapa da obra.
- **FR-022**: O sistema MUST permitir que Técnico de Obra ou Administrador Geral
  registrem manualmente uma medição do governo vinculada a uma etapa específica,
  incluindo seu status de aprovação, com base em comunicação oficial recebida fora do
  sistema; cada medição aprovada sinaliza a liberação de recursos e valida a
  conclusão informada para aquela etapa. Esta versão MUST NOT depender de integração
  automática com sistemas externos do governo.
- **FR-023**: O sistema MUST permitir o registro de ocorrências e observações técnicas
  associadas a uma etapa da obra.

**Mutirão e Presença**

- **FR-024**: Técnico de Obra e Administrador Geral MUST ser capazes de criar escalas
  de mutirão informando data, turno, número de vagas por família e a pontuação
  concedida por presença naquele mutirão específico.
- **FR-025**: O sistema MUST permitir o registro de presença de famílias em um
  mutirão, respeitando o limite de vagas definido na escala.
- **FR-026**: O sistema MUST conceder à família a pontuação definida para o mutirão em
  que sua presença foi registrada.
- **FR-027**: O sistema MUST gerar um relatório de pontuação e participação por
  família, exibindo histórico de presenças e pontuação acumulada.
- **FR-028**: O sistema MUST sinalizar para o Assistente Social as famílias cuja
  pontuação acumulada estiver abaixo de um limite mínimo configurável pelo
  Administrador Geral.

**Dashboard Administrativo**

- **FR-029**: O sistema MUST fornecer um dashboard administrativo exibindo total de
  famílias por status, percentual de conclusão da obra e próximos mutirões.
- **FR-030**: O conteúdo do dashboard MUST ser filtrado de acordo com o perfil do
  usuário autenticado, exibindo apenas os indicadores relevantes às suas
  responsabilidades.

### Key Entities

- **Família**: Representa um núcleo familiar beneficiário. Atributos principais:
  composição familiar (membros, idades, vínculos e CPF de cada membro, único entre
  famílias ativas), renda familiar, situação de vulnerabilidade social, status no
  workflow, histórico de transições de status (incluindo motivos de
  reprovação/reversão), pontuação acumulada em mutirões. Todos os campos, incluindo os
  sensíveis, são visíveis a qualquer perfil autenticado; cadastro e edição permanecem
  restritos por perfil (FR-031). Relaciona-se com Documentos, Unidade Habitacional e
  Presenças em Mutirão.
- **Documento**: Representa um item de documentação obrigatória vinculado a uma
  família (ex.: RG, CPF, comprovante de renda, certidão). Atributos: tipo, status
  (pendente/recebido/validado).
- **Unidade Habitacional**: Representa um lote/unidade do empreendimento. Atributos:
  identificador, metragem, localização no terreno, status de ocupação (livre,
  reservada, ocupada). Relaciona-se com a Família à qual foi atribuída.
- **Etapa da Obra**: Representa uma fase da construção (ex.: fundação, estrutura).
  Atributos: nome, ordem, percentual de conclusão. Relaciona-se com Medições e
  Ocorrências.
- **Medição**: Representa uma medição de progresso do governo, registrada
  manualmente por Técnico de Obra ou Administrador Geral com base em comunicação
  oficial externa, vinculada a uma Etapa da Obra. Atributos: data, etapa vinculada,
  status de aprovação, recursos liberados.
- **Ocorrência**: Representa um registro de observação técnica vinculado a uma Etapa
  da Obra. Atributos: descrição, data, etapa vinculada.
- **Mutirão (Escala)**: Representa uma convocação de trabalho comunitário. Atributos:
  data, turno, número de vagas por família, pontuação concedida por presença naquele
  mutirão.
- **Presença**: Representa o registro de participação de uma família em um Mutirão
  específico. Atributos: família, mutirão vinculado, data do registro, pontuação
  concedida.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: O Assistente Social consegue cadastrar uma nova família e registrar toda
  a documentação obrigatória em menos de 10 minutos.
- **SC-002**: 100% das famílias com documentação obrigatória incompleta são impedidas
  de avançar para a etapa "Em Análise" ou posterior.
- **SC-003**: Visitantes da página pública conseguem localizar o status atual da obra
  e o histórico de medições aprovadas sem necessidade de login, em até 2 cliques a
  partir da página inicial.
- **SC-004**: 100% das tentativas de atribuição de uma unidade já ocupada ou reservada
  são bloqueadas pelo sistema.
- **SC-005**: Famílias com pontuação acumulada abaixo do limite mínimo são sinalizadas
  ao Assistente Social em até 24 horas após a atualização da pontuação.
- **SC-006**: Cada perfil autenticado consegue localizar, no dashboard, os indicadores
  relevantes à sua função sem precisar navegar para outros módulos do sistema.
- **SC-007**: 100% das tentativas de acesso a funcionalidades fora do escopo do perfil
  autenticado são bloqueadas e comunicadas ao usuário com uma mensagem clara.

## Assumptions

- O login da área administrativa usa autenticação padrão (usuário/senha ou mecanismo
  equivalente); o método específico de autenticação não é tratado nesta especificação
  e será definido na fase de planejamento técnico.
- O limite mínimo de pontuação que dispara a sinalização de baixa participação é
  configurável pelo Administrador Geral, com um valor padrão inicial definido na
  implementação.
- A pontuação concedida por presença é definida individualmente no momento da criação
  de cada escala de mutirão, permitindo que mutirões diferentes tenham pesos
  diferentes.
- O status "Reservada" para unidades habitacionais é usado para o período entre a
  atribuição inicial e a confirmação final de ocupação; a transição exata entre
  "Reservada" e "Ocupada" será detalhada na fase de planejamento técnico.
- Os relatórios de medições exibidos publicamente incluem apenas medições já
  aprovadas pelo governo; medições pendentes ou rejeitadas não são expostas na área
  pública.
- O escopo desta especificação cobre apenas um único empreendimento habitacional; o
  gerenciamento de múltiplos empreendimentos simultâneos pelo mesmo movimento social
  está fora do escopo desta versão.
