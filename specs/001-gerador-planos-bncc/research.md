# Pesquisa Técnica e Decisões Arquiteturais: Planejador BNCC

**Feature**: `001-gerador-planos-bncc`  
**Data**: 2026-10-01  
**Status**: Concluído (Fase 0 - Revisado com docs/contracts/n8n.md)

Este documento consolida as decisões técnicas, critérios de compatibilidade de versões, padrões de segurança e justificativas arquiteturais para a implementação do Planejador BNCC.

---

## 1. Arquitetura do Repositório e Monorepo

- **Decisão**: Adotar Monorepo gerenciado por **pnpm workspaces** com orquestrador **Turbo** (`turbo ^2.5.6`), contendo `apps/web` (Next.js 15) e `apps/api` (NestJS 11).
- **Racional**:
  - Garante isolamento estrito entre frontend e backend conforme o Princípio II da Constituição.
  - Otimiza caching de compilação, linting e execução de testes através de scripts raiz unificados (`dev`, `lint`, `typecheck`, `test`, `test:integration`, `build`).
  - Lockfile determinístico único (`pnpm-lock.yaml`) para todo o repositório, garantindo reprodutibilidade de dependências.
- **Alternativas Consideradas**:
  - *Repositórios separados (Polyrepo)*: Rejeitado por aumentar atrito no versionamento sincronizado de contratos de API e compartilhamento de scripts de qualidade na disciplina.
  - *Monorepo npm/yarn*: Rejeitado devido ao desempenho superior, gestão eficiente de symlinks e menor ocupação de disco do pnpm (`pnpm@10.x / 12.x`).

---

## 2. Matriz de Versões Compatíveis

Todas as ferramentas foram verificadas contra o ambiente de execução local (Node.js LTS `v22.20.0`, Docker `28.3.3`, Docker Compose `v2.39.2`):

| Tecnologia / Pacote | Versão Compatível | Finalidade |
| :--- | :--- | :--- |
| **Node.js** | `>= 22.0.0` (LTS) | Runtime JavaScript/TypeScript |
| **pnpm** | `^10.17.0` / `12.x` | Gerenciador de pacotes do monorepo |
| **TypeScript** | `^5.9.2` | Tipagem estática estrita em ambos os apps |
| **Turborepo** | `^2.5.6` | Orquestração de pipelines e build cache |
| **Next.js** | `^15.5.2` (React 19) | Frontend SPA / App Router em `apps/web` |
| **NestJS** | `^11.1.6` | API REST modular em `apps/api` |
| **Prisma ORM** | `^6.16.2` | Modelagem, migrações determinísticas e client |
| **PostgreSQL** | `16-alpine` | Banco relacional executado via Docker Compose |
| **Argon2** | `^0.44.0` | Hashing de senhas seguro e resistente a GPU |
| **Vitest** | `^3.2.4` | Framework de testes unitários e de integração |

---

## 3. Estratégia de Autenticação, Sessão e Segurança

- **Decisão**: Autenticação stateless de dois fatores lógicos:
  - **Access Token (JWT)**: Vida curta (15 minutos), retornado no payload de login/refresh e armazenado **estritamente em memória** no cliente web (nunca em `localStorage` ou `sessionStorage`).
  - **Refresh Token**: Vida longa (7 dias), transportado exclusivamente em cookie HTTP seguro (`HttpOnly`, `SameSite=Lax`, `Path=/api/v1/auth/refresh`).
  - **Persistência de Segredos**: O banco armazena exclusivamente o **hash da senha** (via Argon2) e o **hash do refresh token** (SHA-256). O refresh token bruto nunca é gravado.
  - **Ambiente Local vs. Produção**: Em produção, o cookie exige a flag `Secure` (HTTPS obrigatório). Em localhost de desenvolvimento (`http://localhost:3000`), a flag `Secure` é parametrizada via variável `COOKIE_SECURE=false` para viabilizar testes sem certificado SSL local.
  - **Proteção CSRF & CORS**: CORS estritamente restrito à origem `http://localhost:3000` (`WEB_ORIGIN`). Requisições com cookie para o endpoint `/refresh` são protegidas por política de `SameSite=Lax` e cabeçalho customizado.
- **Racional**:
  - Elimina a superfície de ataque para roubo de tokens persistentes via XSS.
  - Cumpre o Princípio III da Constituição (isolamento do professor e proteção de credenciais).
- **Alternativas Consideradas**:
  - *JWT longo em localStorage*: Rejeitado por vulnerabilidade severa a ataques XSS.
  - *Sessão stateful em Redis*: Rejeitado por complexidade desnecessária para o escopo da prática.

---

## 4. Integração com o Agente de IA (n8n Webhook)

- **Decisão**: Cliente HTTP no NestJS (`N8nClient`) dedicado a integrar com o webhook externo do n8n, estritamente alinhado com o contrato confirmado em `data/docs/contracts/n8n.md`:
  - **Regra de `sessao`**: O campo `sessao` transporta o **e-mail do usuário autenticado** (`user.email`), extraído exclusivamente do token Bearer JWT no backend. O cliente frontend nunca envia o identificador de sessão manualmente.
  - **Autenticação**: Cabeçalho `x-api-key` contendo o segredo `N8N_INTEGRATION_SECRET`. O frontend jamais recebe essa chave nem chama o n8n diretamente (Princípio II).
  - **Rastreabilidade**: Envio de cabeçalho `X-Request-Id` gerado via UUID v4 para correlacionar logs da API ao `AiRun`.
  - **Timeout Controlado**: Timeout configurável via `N8N_TIMEOUT_MS` com padrão de 60.000 ms (60 segundos). Ultrapassado o limite, a requisição é abortada via `AbortController`.
  - **Sem Retry Automático**: Nenhuma retentativa em loop é executada pelo backend; novas tentativas decorrem exclusivamente de reenvio manual do professor.
  - **Validação de Contrato (Schema Guard)**: Resposta validada estritamente contra schema: exige `success === true`, `format === "markdown"`, eco de `sessao === user.email`, eco de `habilidade`, e `answer` (Markdown não vazio).
  - **Modo Mock Local (`N8N_MOCK_ENABLED=true`)**: Permite executar todo o ciclo de desenvolvimento e testes de integração localmente sem consumir o workflow remoto compartilhado do instrutor, retornando uma resposta sintética válida pré-definida ecoando o e-mail do professor autenticado ou simulando falhas sob demanda (`[SIMULAR_TIMEOUT]`, `[SIMULAR_ERRO]`).
- **Racional**:
  - Garante total conformidade com a especificação oficial do instrutor e independência dos testes locais da turma sem sobrecarregar a infraestrutura externa.
- **Alternativas Consideradas**:
  - *Chamada direta da IA do Next.js*: Rejeitado categoricamente (Princípio II da Constituição proíbe expor chaves no cliente).
  - *Usar UUID na sessão do n8n*: Rejeitado após comparação com `docs/contracts/n8n.md`, que estabeleceu `sessao: "email-usuario"`.

---

## 5. Atomicidade da Transação de Geração de Planos

- **Decisão**: Orquestração transacional no NestJS com Prisma:
  1. Cria registro `AiRun` com status `PENDING` e `requestId` único.
  2. Executa a requisição externa ao n8n (fora da transação de banco para não prender conexões durante os 60s de espera).
  3. **Cenário de Sucesso**: Executa `prisma.$transaction` criando o `Plan` com status `RASCUNHO`, `aiAssisted = true`, vinculado ao `ownerId` do professor e marcando o `AiRun` correspondente como `SUCCEEDED`.
  4. **Cenário de Falha / Timeout**: Atualiza `AiRun` como `FAILED` com código de erro e **NENHUM registro de `Plan` é criado**. O backend devolve HTTP 502/504 com mensagem descritiva sem vazar detalhes internos.
- **Racional**:
  - Atende plenamente ao Princípio V da Constituição (tolerância a falhas sem persistência parcial).

---

## 6. Design System, Tokens do Figma e Responsividade (Sem Tailwind)

- **Decisão**: Utilização de **CSS Puro com Custom Properties (Tokens)** e **CSS Modules**, sem inclusão de Tailwind CSS.
- **Tokens do Design System (extraídos do Figma `Planejador-BNCC`)**:
  ```css
  :root {
    --ink: #17251e;
    --muted: #5d6c64;
    --line: #c8e8dc;
    --green: #30b77e;
    --green-dark: #0a3d30;
    --canvas: #f0f9f6;
    --white: #ffffff;
    --danger: #d9383a;
    --danger-bg: #fde8e8;
    --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    --radius-sm: 6px;
    --radius-md: 10px;
    --radius-lg: 16px;
  }
  ```
- **Mapeamento dos 4 Frames do Figma**:
  1. `30:23234` (*Gerar plano de aula*): Formulário de seleção de habilidades, campos pedagógicos, resumo lateral e botão "Gerar rascunho com IA".
  2. `30:23541` (*Preparando rascunho*): Estado de carregamento com feedback visual animado, bloqueio de submissão dupla e dica de espera.
  3. `30:23678` (*Revisar rascunho*): Tela de visualização e edição do rascunho em Markdown, preview renderizado, botão salvar explícito e indicador de auxílio por IA.
  4. `30:24109` (*Falha ao gerar rascunho*): Banner de alerta de erro, retenção integral dos dados preenchidos no formulário e botão de nova tentativa manual.
- **Adaptação Responsiva**:
  - **Desktop (>= 1024px)**: Layout em duas colunas (coluna principal de formulário + card lateral fixo de resumo e disparo).
  - **Tablet (768px - 1023px)**: Coluna lateral posicionada abaixo do formulário principal, mantendo botões em destaque e largura adaptada.
  - **Celular (< 768px)**: Layout vertical fluido com barra lateral recolhível (menu gaveta ou bottom bar), campos compactos empilhados e botões com área de toque mínima de 44px (WCAG).

---

## 7. Sanitização e Renderização Segura do Markdown

- **Decisão**: No frontend (`apps/web`), renderizar Markdown via componente tipado seguro (usando parser seguro como `react-markdown` configurado com sanitização estrita de nós HTML ou stripping de tags arbitrárias).
- **Racional**:
  - Impede vulnerabilidades de Stored XSS caso a resposta da IA ou a edição do docente contenha tags perigosas (`<script>`, `<iframe onload="...">`).
  - Garante fidelidade visual para títulos (`#`, `##`), listas (`-`, `1.`), blocos de citação e tabelas sem executar scripts.

---

## 8. Persistência e Seed Idempotente

- **Decisão**: 
  - Banco de dados PostgreSQL configurado em `docker-compose.yml`.
  - Migrações determinísticas em `apps/api/prisma/migrations`.
  - Script de seed idempotente (`apps/api/prisma/seed.ts` via `pnpm --filter @planejador/api prisma:seed`) que lê `data/bncc-recorte.json` e cria as duas contas de demonstração:
    - **Professor 1**: `ana.souza@escola.gov.br` (conforme exibido no Figma: "Ana Souza - PROFESSOR")
    - **Professor 2**: `carlos.melo@escola.gov.br` (para teste de isolamento 404 entre contas)
  - Senhas de demonstração geradas via hash Argon2; upsert baseado em código único da habilidade e e-mail único do usuário.
