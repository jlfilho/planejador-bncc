<!--
SYNC IMPACT REPORT
==================
Version Change: [CONSTITUTION_VERSION] (unratified template) -> 1.0.0
Ratification Date: 2026-10-01
Last Amended Date: 2026-10-01

Modified Principles:
- Replaced template placeholders with 8 definitive project principles:
  1. I. Especificação Prévia (Spec-First)
  2. II. Segregação Arquitetural e Proteção de Segredos
  3. III. Autenticação e Isolamento por Professor
  4. IV. IA como Rascunho Assistivo e Supervisão Docente
  5. V. Validação Rigorosa e Atomicidade Transacional
  6. VI. Persistência Reprodutível (Migrations e Seeds)
  7. VII. Fidelidade ao Design System, Acessibilidade e Responsividade
  8. VIII. Qualidade Verificável e Segurança de Artefatos

Added Sections:
- Diretrizes Técnicas e Segurança
- Fluxo de Desenvolvimento e Garantia da Qualidade

Removed Sections:
- Nenhuma (placeholders estruturados substituídos)

Deferred Items / Follow-up TODOs:
- Nenhum. Todos os placeholders foram completamente resolvidos.
-->

# Planejador BNCC Constitution

## Core Principles

### I. Especificação Prévia (Spec-First)
O comportamento funcional, contratos de dados e critérios de aceitação DEVEM ser formalmente definidos e validados antes de qualquer escrita de código de produção. Nenhuma funcionalidade pode ser implementada sem especificação aprovada prévia.
- Racional: Elimina ambiguidades pedagógicas e de regras de negócio, reduz retrabalho e direciona a criação de testes precisos.

### II. Segregação Arquitetural e Proteção de Segredos
Frontend, API e serviços de integração externa (como provedores de IA) DEVEM ser estritamente desacoplados. Todas as chaves de API, credenciais e variáveis de ambiente confidenciais DEVEM residir e ser acessadas exclusivamente na camada de backend.
- Racional: Impede o vazamento de credenciais no cliente web e garante limites de responsabilidade limpos e seguros.

### III. Autenticação e Isolamento por Professor
Todo acesso à plataforma DEVE ser devidamente autenticado. Cada operação de leitura, criação, edição ou remoção DEVE validar autorização estrita garantindo que o professor acesse unicamente seus próprios planos e dados.
- Racional: Assegura a privacidade, conformidade regulatória e isolamento de informações pedagógicas entre docentes.

### IV. IA como Rascunho Assistivo e Supervisão Docente
Todo conteúdo gerado por modelos de inteligência artificial DEVE ser tratado como rascunho (`DRAFT`) editável e preliminar. O sistema NUNCA DEVE finalizar ou publicar planos automaticamente sem a intervenção e validação explícita do docente.
- Racional: Preserva a soberania e autoridade pedagógica do professor, mitigando riscos de alucinações ou imprecisões conceituais.

### V. Validação Rigorosa e Atomicidade Transacional
Todas as entradas do usuário e respostas recebidas de serviços externos DEVEM ser validadas contra schemas estritos. Em caso de falha de validação, erro de conexão ou retorno inválido da IA, a operação DEVE abortar de forma segura, sendo PROIBIDO persistir estados parciais ou corrompidos.
- Racional: Garante consistência referencial, estabilidade operacional e integridade de dados no banco.

### VI. Persistência Reprodutível (Migrations e Seeds)
A evolução do esquema do banco de dados DEVE ser gerenciada exclusivamente por migrações versionadas determinísticas. Dados fundamentais e fixos (como o catálogo oficial da BNCC) DEVEM ser carregados por scripts de seed idempotentes e reprodutíveis.
- Racional: Assegura paridade entre ambientes de desenvolvimento, teste e produção, tornando a infraestrutura confiável.

### VII. Fidelidade ao Design System, Acessibilidade e Responsividade
As interfaces DEVEM implementar fielmente os componentes e design tokens estabelecidos no Figma (cores, tipografia, espaçamentos e estados visuais). O layout DEVE ser acessível (diretrizes WCAG/WAI-ARIA) e responsivo para desktop, tablet e celular.
- Racional: Garante consistência visual profissional, excelente usabilidade e inclusão para todos os dispositivos e perfis de docentes.

### VIII. Qualidade Verificável e Segurança de Artefatos
Comportamentos e fluxos críticos DEVEM ser cobertos por testes automatizados com execução documentada. Os artefatos do projeto DEVEM ser versionados no Git, sendo TERMINANTEMENTE PROIBIDO comitar arquivos `.env`, segredos, chaves privadas ou certificados.
- Racional: Garante a manutenibilidade, previsibilidade de entregas e previne incidentes de segurança cibernética.

## Diretrizes Técnicas e Segurança

- **Camada de Apresentação (Frontend)**: SPA/Interface moderna consumindo endpoints da API, aderindo estritamente ao Design System do Figma e garantindo navegação acessível e responsiva.
- **Camada de Serviços e Dados (Backend)**: API estruturada com validação de payload na entrada, autorização em todas as rotas de domínio pedagógico e orquestração controlada de chamadas a provedores de LLM.
- **Proteção de Segredos**: Arquivos de ambiente locais protegidos por `.gitignore`, com disponibilização de `.env.example` documentando os parâmetros sem valores confidenciais.

## Fluxo de Desenvolvimento e Garantia da Qualidade

- **Processo Orientado por Especificação**: Cada funcionalidade segue o ciclo de especificação, planejamento, tarefas e implementação antes de merges.
- **Gestão de Branches e Código**: Branches são gerenciadas de forma disciplinada; nenhum merge não autorizado é permitido.
- **Portões de Qualidade**: Código novo deve passar por validação de tipagem, linting e testes automatizados das rotas e regras de negócio críticas.

## Governance

- Esta Constituição tem prevalência sobre quaisquer outras práticas e convenções informais de desenvolvimento da equipe.
- Qualquer alteração nos princípios ou diretrizes exige atualização formal deste documento, justificativa explícita e aprovação.
- Políticas de Versionamento Semântico da Constituição:
  - **MAJOR**: Alteração, remoção ou redefinição incompatível de princípios de governança.
  - **MINOR**: Inclusão de novos princípios, seções ou expansão significativa de diretrizes.
  - **PATCH**: Correções ortográficas, esclarecimentos de redação ou ajustes não conceituais.
- Revisões de código e PRs DEVEM validar ativamente a conformidade com todos os princípios aqui ratificados.

**Version**: 1.0.0 | **Ratified**: 2026-10-01 | **Last Amended**: 2026-10-01
