# Guia Rápido de Execução e Validação: Planejador BNCC

**Feature**: `001-gerador-planos-bncc`  
**Data**: 2026-10-01  
**Status**: Concluído (Fase 1 - Revisado com docs/contracts/n8n.md)

Este guia apresenta o passo a passo com comandos reais para provisionar o ambiente local, executar migrações e sementes, iniciar os serviços de desenvolvimento e validar as jornadas de usuário ponta a ponta.

---

## 1. Pré-Requisitos do Ambiente

- **Node.js**: `v22.x` (LTS verificado: `v22.20.0`)
- **pnpm**: `^10.x` / `12.x` (instalado globalmente)
- **Docker Desktop**: Ativo e com suporte a Compose v2

---

## 2. Configuração das Variáveis de Ambiente

Crie os arquivos `.env` locais para a API e Web a partir dos modelos documentados:

```bash
# Na raiz do monorepo
cp apps/api/.env.example apps/api/.env
cp apps/web/.env.example apps/web/.env
```

### Variáveis essenciais em `apps/api/.env`:
```dotenv
PORT=3001
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/planejador_bncc?schema=public"
JWT_SECRET="chave-secreta-de-desenvolvimento-local-min-32-chars"
COOKIE_SECURE="false"
WEB_ORIGIN="http://localhost:3000"
N8N_WEBHOOK_URL="https://n8n-mestrado.tecnocomp.cloud/webhook/agente-planejador-bncc"
N8N_AUTH_HEADER_NAME="x-api-key"
N8N_INTEGRATION_SECRET="chave-temporaria-do-instrutor"
N8N_TIMEOUT_MS=60000
N8N_MOCK_ENABLED=true
```

### Variáveis essenciais em `apps/web/.env`:
```dotenv
PORT=3000
NEXT_PUBLIC_API_URL="http://localhost:3001/api/v1"
```

---

## 3. Inicialização dos Serviços e Banco de Dados

```bash
# 1. Subir container PostgreSQL em background
docker compose up -d

# 2. Instalar dependências do monorepo (lockfile determinístico)
pnpm install

# 3. Gerar Prisma Client e aplicar migrações determinísticas
pnpm --filter @planejador/api prisma:generate
pnpm --filter @planejador/api prisma:migrate

# 4. Executar seed idempotente (contas demo + catálogo BNCC data/bncc-recorte.json)
pnpm --filter @planejador/api prisma:seed
```

---

## 4. Execução em Desenvolvimento

Inicie as aplicações simultaneamente através do orquestrador Turbo:

```bash
pnpm dev
```

- **Frontend Web**: [http://localhost:3000](http://localhost:3000)
- **API NestJS**: [http://localhost:3001/api/v1](http://localhost:3001/api/v1)

---

## 5. Roteiro de Validação das Histórias de Usuário

### Cenário 1: Autenticação de Demonstração e Sessão (US-1 e US-5)
1. Acesse `http://localhost:3000/login`.
2. Efetue login com o **Professor 1**:
   - E-mail: `ana.souza@escola.gov.br`
   - Senha: `SenhaSegura123!`
3. Confirme o redirecionamento para o dashboard com o nome "Ana Souza - PROFESSOR" na barra lateral.

### Cenário 2: Consulta e Seleção de Habilidades BNCC (US-2)
1. Navegue para `http://localhost:3000/planos/gerar`.
2. No campo de busca do catálogo, digite `"EF02CO"`.
3. Selecione as habilidades `EF02CO02` e `EF02CO04`. Confirme a adição dos chips correspondentes no topo do formulário.

### Cenário 3: Geração Atômica com IA e Validação de Contrato n8n (US-3)
1. Preencha os campos pedagógicos:
   - Instrução: *"Criar uma atividade introdutória em dupla com foco em exemplos do cotidiano."*
   - Duração: `50`
   - Recursos Digitais: `Sim`
2. Clique em **"Gerar rascunho com IA"**.
3. Observe a transição visual para o estado de preparação (frame `30:23541`).
4. Ao concluir (usando `N8N_MOCK_ENABLED=true` com eco do e-mail `ana.souza@escola.gov.br` no campo `sessao`), o sistema persiste atomicamente o plano em `RASCUNHO` e redireciona para a visualização do rascunho gerado (frame `30:23678`).

### Cenário 4: Edição, Preview e Salvamento Explícito (US-4)
1. Modifique o texto do Markdown no editor.
2. Alterne para a aba **Visualizar** para verificar a apresentação renderizada e sanitizada.
3. Clique em **"Salvar rascunho"** e confirme a mensagem de sucesso.
4. Tente clicar em um link da barra lateral com modificações pendentes não salvas para verificar o **modal de bloqueio contra descarte acidental**.

### Cenário 5: Verificação de Isolamento e Bloqueio 404 entre Docentes (US-5)
1. Copie o ID da URL do plano gerado pela Professora Ana Souza.
2. Efetue logout.
3. Faça login com o **Professor 2**:
   - E-mail: `carlos.melo@escola.gov.br`
   - Senha: `SenhaSegura123!`
4. Acesse a lista de planos de Carlos Melo: confirme que o plano de Ana Souza **não aparece**.
5. Tente acessar diretamente a rota `http://localhost:3000/planos/<ID_DO_PLANO_DE_ANA>`:
   - **Resultado Esperado**: Erro **404 (Não Encontrado)** é apresentado; nenhum dado ou metadado do plano é revelado.

---

## 6. Comandos de Qualidade e Testes Automatizados

Execute os scripts de garantia de qualidade na raiz do monorepo:

```bash
# Validação de regras de linting
pnpm lint

# Checagem estrita de tipos TypeScript
pnpm typecheck

# Testes unitários de serviços e componentes
pnpm test

# Testes de integração (API + Prisma + n8n Client Mock com validação de contrato)
pnpm test:integration

# Build de produção do monorepo completo
pnpm build
```
