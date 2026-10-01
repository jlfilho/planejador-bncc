# Feature Specification: Planejador BNCC - Geração de Planos de Aula Assistida por IA

**Feature Branch**: `main` (branch em uso no repositório)

**Created**: 2026-10-01

**Status**: Draft

**Input**: User description: "Desenvolver o Planejador BNCC para uso de professores. O professor entra com uma conta de demonstração previamente cadastrada. Após o login, consulta habilidades BNCC por nível, ano quando aplicável, eixo, código ou texto e seleciona uma ou mais habilidades. Informa uma instrução pedagógica, duração em minutos e se utilizará recursos digitais. Ao confirmar, visualiza um estado de preparação. O serviço de IA recebe a solicitação. Se a resposta for válida, a aplicação salva um plano privado em estado RASCUNHO, com indicação de auxílio por IA. O professor vê o Markdown, pode editá-lo, visualizar sua apresentação, salvar explicitamente e consultar a lista de seus rascunhos. Se a geração falhar, os campos são preservados e nenhum plano parcial é salvo. Uma nova tentativa é iniciada somente por ação do professor. Outro professor não pode ler nem editar o rascunho. Incluir login, logout, sessão, catálogo mínimo e duas contas de demonstração para testar o acesso privado. Sem cadastro público nem administração. Sem PDF, finalização, versionamento de planos ou publicação pública deles. Defina histórias priorizadas e critérios de aceitação verificáveis. Ainda não implemente código."

## Clarifications

### Session 2026-10-01
- Q: Como a aplicação deve responder quando um professor tentar acessar diretamente a rota ou identificador de um rascunho de outro professor? → A: Retornar status 404 (Não Encontrado), tratando o rascunho como inexistente para mitigar riscos de enumeração de dados e vazamento de metadados.
- Q: Como o estado de autenticação e sessão do professor deve ser mantido entre requisições no navegador? → A: Token Bearer (JWT) enviado via cabeçalho Authorization nas requisições à API, gerenciado pelo cliente web durante a sessão ativa.
- Q: Qual deve ser o tempo limite (timeout) para a requisição de geração com IA antes de abortar como falha atômica? → A: 60 segundos de tempo limite máximo para a chamada ao provedor externo de IA; ultrapassado esse período, a requisição é cancelada imediatamente, a transação abortada com 0 planos persistidos e os dados do formulário preservados na tela.
- Q: Como o editor deve agir se o professor tentar navegar para outra tela com alterações não salvas no rascunho? → A: Exibir modal de confirmação alertando sobre alterações não salvas, exigindo confirmação explícita do professor antes de descartar os dados editados.
- Q: Qual deve ser o formato do contrato de resposta esperado do serviço de IA para validação estrita no backend? → A: Objeto JSON estruturado contendo seções pedagógicas obrigatórias (título sugerido, objetivos específicos, introdução/aquecimento, desenvolvimento da aula, avaliação/fechamento) e campo com o Markdown consolidado, validado contra schema rígido antes de qualquer persistência.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Autenticação com Conta de Demonstração e Gestão de Sessão (Priority: P1)

Como professor da educação básica,
quero autenticar-me na aplicação com uma conta de demonstração pré-cadastrada e poder encerrar minha sessão,
para que eu tenha um ambiente de trabalho pedagógico privado e seguro para planejar minhas aulas.

**Why this priority**: A autenticação e a gestão de sessão são a base essencial para qualquer personalização, isolamento de dados e conformidade com o princípio de privacidade dos planos de cada docente.

**Independent Test**: Pode ser testado de forma totalmente autônoma realizando login com as credenciais de demonstração fornecidas, verificando a criação de sessão ativa com token Bearer e executando o logout para confirmar o descarte do token e bloqueio de acessos subsequentes.

**Acceptance Scenarios**:

1. **Given** um professor na página de acesso com credenciais de demonstração válidas, **When** submete o formulário de login, **Then** o sistema autentica o professor, emite um token Bearer (JWT), inicia uma sessão ativa no cliente e redireciona para a tela principal de planos de aula.
2. **Given** um professor na página de acesso, **When** insere credenciais não cadastradas ou inválidas, **Then** o sistema rejeita o acesso com mensagem de erro descritiva e não emite token de autenticação.
3. **Given** um professor autenticado com sessão ativa, **When** aciona a opção de logout, **Then** o sistema descarta imediatamente o token no cliente, encerra a sessão e redireciona para a tela de autenticação, impedindo acessos a rotas privadas.

---

### User Story 2 - Consulta e Seleção de Habilidades no Catálogo BNCC (Priority: P1)

Como professor autenticado,
quero pesquisar e filtrar habilidades da BNCC por nível de ensino, ano escolar, eixo temático, código oficial ou texto descritivo e selecionar uma ou mais habilidades ativas,
para que eu fundamente minha aula nas diretrizes curriculares nacionais oficiais de forma rápida e contextualizada.

**Why this priority**: A seleção fundamentada de habilidades ativas da BNCC é o requisito estrutural que alimenta tanto a elaboração pedagógica quanto o contexto enviado para geração assistida por IA.

**Independent Test**: Pode ser testado independentemente utilizando o catálogo de habilidades em modo somente leitura, aplicando filtros combinados por código/eixo/ano e selecionando/desmarcando habilidades ativas na tabela.

**Acceptance Scenarios**:

1. **Given** um professor autenticado na tela de criação de plano, **When** visualiza o catálogo da BNCC, **Then** o sistema exibe apenas habilidades catalogadas como ativas com código, ano, eixo e descrição em modo somente leitura.
2. **Given** um professor na tela de criação, **When** digita um termo de busca (código ou palavras-chave da descrição) ou filtra por ano e eixo, **Then** a listagem do catálogo exibe instantaneamente as habilidades correspondentes aos critérios informados.
3. **Given** habilidades exibidas no catálogo, **When** o professor marca uma ou mais habilidades válidas, **Then** as habilidades são adicionadas à lista de selecionadas como chips identificadores contendo código e síntese, com opção de remoção individual.

---

### User Story 3 - Configuração Pedagógica e Geração Atômica de Rascunho com IA (Priority: P1)

Como professor com habilidades selecionadas,
quero informar a instrução pedagógica, duração da aula em minutos, título provisório e a intenção de usar recursos digitais, acionando a preparação por IA,
para que eu obtenha um rascunho de plano de aula personalizado, estruturado e atômico, sem risco de corrupção ou perda dos meus dados em caso de falha.

**Why this priority**: É a proposta de valor central do produto: apoiar a elaboração de planos de aula por meio de IA, garantindo que o plano nasça estritamente como rascunho privado sob controle docente e com tolerância atômica a falhas.

**Independent Test**: Pode ser testado preenchendo todos os dados pedagógicos e submetendo a solicitação: validando que o plano criado tem status `RASCUNHO` e indicação de assistência por IA; e simulando uma indisponibilidade da IA (ou timeout de 60 segundos) para comprovar que nenhum plano parcial é gravado e que todos os dados do formulário permanecem intactos.

**Acceptance Scenarios**:

1. **Given** o formulário com habilidades selecionadas, instrução pedagógica válida, duração positiva em minutos e escolha sobre recursos digitais, **When** o professor confirma o envio em "Gerar rascunho com IA", **Then** a interface apresenta um estado visual de preparação (indicação de processamento) e bloqueia submissões duplicadas.
2. **Given** o estado de preparação em andamento, **When** o serviço de IA responde dentro de até 60 segundos com JSON estruturado válido e seções requeridas, **Then** o sistema persiste o plano com status `RASCUNHO`, associado ao professor autenticado, contendo a marcação de plano gerado com auxílio de IA, e direciona o professor para visualização do rascunho gerado.
3. **Given** o estado de preparação em andamento, **When** o serviço de IA falha (por timeout de 60 segundos, indisponibilidade ou resposta corrompida fora do schema JSON), **Then** o sistema encerra o estado de preparação, exibe mensagem clara de erro, NÃO salva nenhum registro de plano no sistema e preserva rigorosamente todos os campos previamente preenchidos no formulário.
4. **Given** uma falha ocorrida na geração e o formulário preservado, **When** nenhuma ação é tomada pelo professor, **Then** o sistema não executa novas tentativas automáticas, aguardando exclusivamente uma ação manual explícita de reenvio por parte do docente.

---

### User Story 4 - Visualização, Edição e Salvamento Explícito do Rascunho (Priority: P2)

Como professor proprietário de um rascunho,
quero visualizar o texto em formato Markdown, alternar para a pré-visualização formatada, editar o conteúdo pedagógico e salvar as alterações explicitamente,
para que eu refine, adeque e complemente o plano de aula conforme a realidade pedagógica da minha turma antes de qualquer uso.

**Why this priority**: Cumpre o princípio constitucional da soberania docente, garantindo que o resultado da IA seja um ponto de partida editável e que toda alteração seja salva por decisão consciente do professor.

**Independent Test**: Pode ser testado abrindo um rascunho existente, modificando o conteúdo Markdown, alternando entre as abas/painéis de edição e preview formatado, e acionando o botão de salvar, verificando a persistência das alterações e a interceptação de saídas acidentais com modal de confirmação.

**Acceptance Scenarios**:

1. **Given** um rascunho recém-gerado ou carregado, **When** o professor acessa a página do rascunho, **Then** o sistema exibe o conteúdo em Markdown, o status de `RASCUNHO`, a identificação de assistência por IA e o botão de salvar alterações.
2. **Given** o professor no rascunho, **When** altera entre o editor de Markdown e a visualização de apresentação (preview), **Then** o sistema renderiza a formatação visual (títulos, listas, destaques) de forma fidedigna e sem atraso perceptível.
3. **Given** alterações realizadas no editor Markdown, **When** o professor clica na ação de salvar explicitamente, **Then** o sistema atualiza o conteúdo do rascunho e emite confirmação de salvamento bem-sucedido.
4. **Given** um professor com alterações não salvas no editor Markdown, **When** tenta navegar para outra página ou fechar o rascunho, **Then** o sistema intercepta a navegação e exibe modal de confirmação alertando sobre alterações não salvas, prosseguindo apenas com consentimento explícito do docente.

---

### User Story 5 - Listagem de Rascunhos e Isolamento Privado entre Professores (Priority: P2)

Como professor autenticado,
quero consultar a lista contendo exclusivamente meus próprios rascunhos de planos de aula e ter garantia de que nenhum outro professor visualize ou edite meu trabalho,
para que eu gerencie meu histórico de planejamentos com total confidencialidade e segurança.

**Why this priority**: Garante o controle contínuo dos planos criados pelo docente e valida o princípio inegociável de isolamento estrito de dados entre diferentes usuários.

**Independent Test**: Pode ser testado autenticando como Professor 1, gerando um rascunho e verificando sua presença na listagem; em seguida autenticando como Professor 2 para confirmar que o rascunho de Professor 1 não aparece na listagem e que uma tentativa de acesso direto àquele rascunho é categoricamente negada com status 404.

**Acceptance Scenarios**:

1. **Given** o Professor 1 autenticado na tela de listagem de planos, **When** visualiza seus rascunhos, **Then** o sistema lista apenas os planos criados pelo Professor 1, apresentando título provisório, data de atualização, status `RASCUNHO` e indicador de auxílio por IA.
2. **Given** o Professor 2 autenticado no sistema, **When** acessa sua própria listagem de planos, **Then** os planos pertencentes ao Professor 1 não são exibidos em nenhuma hipótese.
3. **Given** o Professor 2 autenticado, **When** tenta acessar diretamente por identificador a rota de leitura ou modificação de um rascunho de propriedade do Professor 1, **Then** o sistema retorna status de não encontrado (404), ocultando a existência do plano e bloqueando qualquer vazamento de dados.

---

### Edge Cases

- **Ausência de Habilidades Selecionadas**: O professor tenta submeter a geração sem selecionar ao menos uma habilidade ativa; o sistema impede a ação e sinaliza visualmente a exigência mínima.
- **Valores Inválidos no Formulário**: O professor informa duração negativa, zero ou não numérica, ou omite a instrução pedagógica; o sistema valida os campos antes do envio, bloqueando a requisição com mensagens amigáveis de orientação.
- **Busca Sem Resultados no Catálogo**: O termo digitado no filtro não corresponde a nenhuma habilidade; o sistema exibe estado vazio indicando ausência de registros sem quebrar o layout.
- **Instabilidade ou Timeout da IA**: A chamada externa ao serviço de inteligência artificial excede o limite estrito de 60 segundos; o sistema encerra a requisição, não cria rascunho e avisa o professor mantendo os dados digitados intactos.
- **Resposta da IA Fora do Schema**: O provedor de IA retorna JSON inválido, campos obrigatórios ausentes ou resposta truncada; o validador de schema rejeita a resposta sumariamente, a operação é abortada sem registro na base e os dados do formulário permanecem preservados.
- **Sessão Expirada durante Edição**: O professor tenta salvar alterações em um rascunho após a expiração de sua sessão ou token; o sistema bloqueia a escrita, orienta o reingresso e assegura que os dados editados não sejam sobrescritos inadvertidamente.
- **Tentativa de Saída com Edições Pendentes**: O professor clica em links de navegação ou botão voltar enquanto edita o Markdown; o sistema bloqueia a transição e apresenta diálogo de confirmação antes de descartar qualquer conteúdo modificado.
- **Acesso Concorrente de Contas de Demonstração**: Professor 1 e Professor 2 operam em navegadores distintos; cada um mantém contexto estritamente isolado sem interferência em tokens de sessão cruzados.
- **Acesso a Rascunho de Terceiros**: Tentativa de acesso direto via ID a plano alheio resulta em 404 imediato sem exibir conteúdo ou confirmar existência do recurso.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema DEVE disponibilizar autenticação por credenciais com emissão de token Bearer (JWT) para duas contas de demonstração pré-configuradas (Professor 1 e Professor 2).
- **FR-002**: O sistema DEVE exigir o envio do token Bearer no cabeçalho `Authorization` para todas as requisições autenticadas da API e permitir ao professor encerrar sua sessão a qualquer momento (logout), invalidando/descartando o token no cliente.
- **FR-003**: O sistema DEVE manter um catálogo mínimo oficial de habilidades da BNCC em modo estritamente somente leitura.
- **FR-004**: O sistema DEVE permitir a pesquisa e filtragem de habilidades da BNCC por nível de ensino, ano escolar, eixo temático, código identificador ou termos contidos na descrição.
- **FR-005**: O sistema DEVE permitir a seleção e desseleção de uma ou mais habilidades ativas para inclusão na solicitação do plano.
- **FR-006**: O sistema DEVE disponibilizar formulário pedagógico exigindo: seleção de habilidades ativas, instrução pedagógica, duração em minutos (número positivo), título provisório e indicação de uso de recursos digitais (Sim/Não).
- **FR-007**: O sistema DEVE apresentar estado visual de preparação imediatamente após a confirmação da geração pelo professor, desabilitando novas submissões simultâneas.
- **FR-008**: O sistema DEVE encaminhar a solicitação estruturada ao serviço de IA exigindo resposta em JSON estruturado com seções pedagógicas obrigatórias e Markdown consolidado, aplicando timeout estrito de 60 segundos e validação rigorosa contra schema antes do aceite.
- **FR-009**: Em caso de sucesso na geração com IA, o sistema DEVE persistir o plano de aula em estado `RASCUNHO`, com indicação permanente de assistência por IA e associado privadamente ao professor autenticado.
- **FR-010**: Em caso de falha de conexão, timeout (60 segundos) ou retorno fora do schema JSON obrigatório, o sistema DEVE abortar a transação, assegurando que NENHUM registro parcial ou corrompido de plano seja salvo na base de dados.
- **FR-011**: Em caso de falha na geração, o sistema DEVE reter na interface todos os valores informados pelo professor no formulário, permitindo que uma nova tentativa ocorra unicamente por ação manual deliberada do usuário.
- **FR-012**: O sistema DEVE disponibilizar visualização do plano gerado em formato Markdown com suporte a edição textual direta pelo professor.
- **FR-013**: O sistema DEVE disponibilizar pré-visualização (preview) renderizada da apresentação visual do Markdown do plano de aula.
- **FR-014**: O sistema DEVE permitir que o professor salve explicitamente as modificações realizadas no rascunho do plano de aula e DEVE interceptar tentativas de navegação com alterações não salvas, exibindo modal de confirmação para evitar descarte involuntário.
- **FR-015**: O sistema DEVE listar todos os rascunhos pertencentes exclusivamente ao professor autenticado, exibindo título provisório, data de modificação, estado de rascunho e indicador de auxílio por IA.
- **FR-016**: O sistema DEVE aplicar controle de autorização estrito em todas as operações de plano de aula; tentativas de acesso direto a rascunhos de outro professor DEVEM retornar status 404 (Não Encontrado), impedindo enumeração e vazamento de metadados.
- **FR-017**: O sistema NÃO DEVE fornecer funcionalidades de cadastro público de novos usuários nem painéis de administração.
- **FR-018**: O sistema NÃO DEVE incluir geração/exportação de arquivos PDF, finalização de plano além do status de rascunho, versionamento histórico de alterações ou publicação pública de planos.

### Key Entities *(include if feature involves data)*

- **Professor (Conta de Demonstração)**: Representa o docente autenticado. Atributos conceituais: identificador único, nome de exibição, credenciais de demonstração, token de sessão Bearer e papel (`PROFESSOR`).
- **Habilidade BNCC**: Unidade de competência curricular oficial do catálogo nacional. Atributos conceituais: código oficial (ex.: `EF02CO02`), nível educacional, ano escolar aplicável, eixo temático (ex.: `Mundo Digital`, `Cultura Digital`), descrição curricular e status de ativação (ativa/inativa).
- **Solicitação de Preparação**: Parâmetros pedagógicos informados para compor o plano. Atributos conceituais: conjunto de referências às habilidades selecionadas, instrução pedagógica docente, duração em minutos, título provisório e flag de uso de recursos digitais.
- **Rascunho de Plano de Aula**: Artefato pedagógico gerado e mantido privadamente pelo professor. Atributos conceituais: identificador único, vínculo ao professor proprietário, título, seções estruturadas (título, objetivos, introdução, desenvolvimento, avaliação), conteúdo textual em formato Markdown consolidado, status obrigatório (`RASCUNHO`), flag de assistência por IA (`true`), data de criação e data da última modificação salva.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% dos planos de aula criados via geração assistida nascem gravados com status `RASCUNHO` e com a marcação visível de auxílio por IA.
- **SC-002**: 0% de planos parciais ou inconsistentes persistidos no banco de dados quando houver qualquer interrupção, timeout (após 60 segundos) ou retorno inválido/fora de schema na integração com IA.
- **SC-003**: 100% dos dados informados no formulário de preparação permanecem preservados na tela após uma falha de geração, permitindo reenvio sem redigitação.
- **SC-004**: 100% de eficácia no isolamento entre contas: nenhuma conta de demonstração consegue visualizar ou manipular rascunhos pertencentes a outro professor (taxa de vazamento 0%), com requisições não autorizadas respondidas com 404.
- **SC-005**: O tempo de transição visual para o estado de preparação após o acionamento do botão é inferior a 500 milissegundos.
- **SC-006**: A alternância entre o editor Markdown e o modo de visualização formatada ocorre de maneira imediata (tempo perceptível inferior a 200 milissegundos).

## Assumptions

- O ambiente dispõe de duas contas de demonstração pré-configuradas para validação de testes e avaliação pedagógica (sem necessidade de fluxo de auto-cadastro).
- O catálogo mínimo da BNCC é pré-carregado no banco com habilidades ativas estruturadas e permanece imutável pelos professores (somente leitura).
- O serviço de IA é integrado exclusivamente na camada de backend, com credenciais e chaves protegidas fora do alcance do cliente frontend.
- O formato de entrega do plano ao professor é Markdown puro estruturado a partir de JSON validado, compatível com renderização de títulos, listas e ênfases pedagógicas.
- Recursos de exportação em PDF, aprovação/finalização administrativa, publicação pública e versionamento histórico foram expressamente declarados fora de escopo para esta versão.
- A interface de usuário suporta uso responsivo e acessível em dispositivos desktop, tablet e celular, respeitando o Design System aprovado.
