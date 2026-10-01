# Implementation Plan: Planejador BNCC - Geração e Edição de Planos de Aula com IA

**Branch**: `main` (branch em uso no repositório) | **Date**: 2026-10-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification de `specs/001-gerador-planos-bncc/spec.md` alinhada ao contrato confirmado em [data/docs/contracts/n8n.md](../../../data/docs/contracts/n8n.md)

---

## Summary

O Planejador BNCC é uma aplicação web fullstack desenhada para apoiar professores da educação básica na elaboração assistida de planos de aula alinhados à Base Nacional Comum Curricular (BNCC). 

A arquitetura adota um **Monorepo pnpm** segregado em duas aplicações:
1. `apps/web`: Frontend em **Next.js 15 (App Router)** com TypeScript, interface responsiva baseada em **CSS Tokens e componentes nativos** (sem Tailwind) inspirada com fidelidade no protótipo do Figma (`Planejador-BNCC`), e renderização segura e sanitizada de Markdown.
2. `apps/api`: Backend em **NestJS 11** com TypeScript e **Prisma ORM 6**, provendo a fronteira de dados segura para o frontend, autenticação por credenciais com tokens JWT em memória e refresh token em cookie `HttpOnly`, catálogo da BNCC somente leitura populado via seed idempotente (`data/bncc-recorte.json`), e cliente de integração com o webhook do **n8n** autenticado via `x-api-key`.

O fluxo garante estrita conformidade com o contrato de integração com a IA ([data/docs/contracts/n8n.md](../../../data/docs/contracts/n8n.md)):
- A requisição transmite `sessao` contendo o e-mail do professor autenticado (`user.email`), extraído exclusivamente do JWT.
- A resposta do n8n exige HTTP 200 com validação estrita de schema (`success: true`, eco exato de `sessao == user.email`, eco de `habilidade`, `format: "markdown"` e `answer` não vazio).
- A transação é atômica: a solicitação inicia um registro `AiRun` em `PENDING`. Em caso de sucesso, o plano é gravado como `RASCUNHO` associado privadamente ao professor e o `AiRun` é marcado como `SUCCEEDED` na mesma transação. Em caso de falha ou timeout (60s), o `AiRun` é marcado como `FAILED`, **nenhum plano parcial é persistido**, e o formulário web permanece intacto para nova tentativa puramente manual.
- O modo mock local (`N8N_MOCK_ENABLED=true`) assegura execução e testes completos (`pnpm test:integration`) sem consumir o workflow compartilhado do instrutor. Tentativas de acesso a planos de terceiros respondem com **404 (Not Found)**.

---

## Technical Context

**Language/Version**: TypeScript 5.9 / Node.js LTS v22.x (runtime verificado: v22.20.0).

**Primary Dependencies**:
- Monorepo: `pnpm@10.x/12.x`, Turborepo `^2.5.6`.
- Backend (`apps/api`): NestJS `^11.1.6` (`@nestjs/core`, `@nestjs/common`, `@nestjs/jwt`, `@nestjs/config`), Prisma `^6.16.2`, Argon2 `^0.44.0`, class-validator `^0.14.2`, cookie-parser `^1.4.7`.
- Frontend (`apps/web`): Next.js `^15.5.2`, React `^19.1.1`, CSS Modules com Design Tokens do Figma (sem Tailwind), parser e sanitizador de Markdown seguro.

**Storage**: PostgreSQL 16 (via Docker Compose `postgres:16-alpine`), gerenciado pelo Prisma ORM com migrações determinísticas.

**Testing**: Vitest `^3.2.4`, Supertest `^7.1.4`, Playwright `^1.55.0` (E2E).

**Target Platform**: Servidor Node.js 22 (backend em `http://localhost:3001`), Navegadores Web modernos (desktop, tablet, celular em `http://localhost:3000`).

**Project Type**: Fullstack Monorepo Web Application (`apps/web` + `apps/api`).

**Performance Goals**: Transição para feedback de preparação < 500ms; alternância editor/preview de Markdown < 200ms; tempo de resposta de endpoints de catálogo e leitura da API < 250ms (p95).

**Constraints**:
- Segregação estrita: frontend nunca recebe chaves de IA nem segredos do n8n.
- Contrato n8n confirmado: `sessao` recebe `user.email`; resposta valida eco de sessão e Markdown.
- Timeout estrito de 60 segundos na comunicação externa com n8n.
- Sem retry automático de IA.
- 0% de persistência de planos parciais em falha.
- Isolamento multi-tenant: plano alheio devolve 404 Not Found.
- CSS nativo com tokens, sem acréscimo de Tailwind.

**Scale/Scope**: 2 contas de demonstração locais pré-cadastradas, catálogo com recorte representativo da BNCC (`data/bncc-recorte.json`), 4 telas/frames do protótipo Figma.

---

## Constitution Check

*GATE: Avaliação de conformidade com todos os 8 princípios da Constituição do Projeto:*

| Princípio Constitucional | Status | Evidência no Planejamento Técnico |
| :--- | :---: | :--- |
| **I. Especificação Prévia (Spec-First)** | **PASS** | `spec.md` gerada, refinada com 5 clarificações e validada com 16/16 itens antes da escrita de código. |
| **II. Segregação e Proteção de Segredos** | **PASS** | Frontend desacoplado em `apps/web`. Segredos do banco e `x-api-key` do n8n residem exclusivamente em `apps/api/.env`. |
| **III. Autenticação e Isolamento Docente** | **PASS** | JWT em memória + cookie HttpOnly. Queries de plano filtram obrigatoriamente por `ownerId`, devolvendo 404 em acessos não autorizados. |
| **IV. IA como Rascunho e Supervisão** | **PASS** | Planos gerados nascem com status fixo `RASCUNHO` e flag `aiAssisted=true`. Edição livre em Markdown e salvamento explícito. |
| **V. Validação e Atomicidade** | **PASS** | DTOs validados; contrato n8n com eco de e-mail e schema estrito; `prisma.$transaction` atômica em sucesso; `FAILED` sem plano parcial em falha. |
| **VI. Persistência Reprodutível** | **PASS** | PostgreSQL em Docker Compose; migrações versionadas determinísticas; seed idempotente com `data/bncc-recorte.json` e 2 contas locais. |
| **VII. Design System, Acessibilidade e Responsividade** | **PASS** | Tokens CSS extraídos do Figma (`--ink`, `--green-dark`, etc.), sem Tailwind. 4 frames mapeados para desktop, tablet e celular. |
| **VIII. Qualidade Verificável e Segurança** | **PASS** | Testes automatizados (unitários e integração), scripts raiz unificados (`dev`, `lint`, `typecheck`, `test`, `test:integration`, `build`) e `.gitignore` blindando credenciais. |

---

## Project Structure

### Documentation (this feature)

```text
specs/001-gerador-planos-bncc/
├── spec.md                     # Especificação funcional refinada
├── plan.md                     # Este plano de implementação
├── research.md                 # Fase 0: Decisões técnicas, compatibilidade e n8n contract
├── data-model.md               # Fase 1: Entidades, Prisma schema e regras transacionais
├── quickstart.md               # Fase 1: Guia com comandos reais de provisionamento e testes
├── checklists/
│   └── requirements.md         # Checklist de qualidade de requisitos (16/16)
└── contracts/
    ├── auth.md                 # Contrato da API de autenticação e sessão
    ├── bncc.md                 # Contrato da API de consulta ao catálogo BNCC
    ├── plans.md                # Contrato da API de planos de aula e geração
    └── n8n-webhook.md          # Contrato M2M backend NestJS ↔ n8n webhook (alinhado a n8n.md)
```

### Source Code (repository root)

```text
planejador-bncc/
├── .gitignore                  # Regras globais de exclusão do git
├── docker-compose.yml          # Container PostgreSQL 16
├── package.json                # Gerenciador monorepo pnpm e scripts raiz
├── pnpm-workspace.yaml         # Definição dos workspaces: apps/*
├── turbo.json                  # Pipelines de build, dev, lint, test
├── data/
│   ├── bncc-recorte.json       # Dataset oficial do catálogo BNCC para seed
│   └── docs/contracts/n8n.md   # Contrato oficial confirmado do webhook n8n
├── apps/
│   ├── api/                    # Backend NestJS
│   │   ├── .env.example        # Modelo de variáveis de ambiente sem segredos
│   │   ├── package.json        # Dependências NestJS, Prisma, Argon2
│   │   ├── tsconfig.json       # Configuração TypeScript estrita
│   │   ├── prisma/
│   │   │   ├── schema.prisma   # Modelos User, Session, BnccSkill, AiRun, Plan
│   │   │   ├── seed.ts         # Script idempotente de carga inicial
│   │   │   └── migrations/     # Migrações determinísticas versionadas
│   │   └── src/
│   │       ├── main.ts         # Bootstrap do NestJS na porta 3001 com CORS
│   │       ├── auth/           # Módulo de autenticação (JWT, guards, DTOs)
│   │       ├── bncc/           # Módulo de catálogo somente leitura da BNCC
│   │       ├── plans/          # Módulo de planos, n8n client e transação atômica
│   │       └── common/         # Filtros de exceção, interceptors e decorators
│   └── web/                    # Frontend Next.js 15
│       ├── .env.example        # Modelo com NEXT_PUBLIC_API_URL
│       ├── package.json        # Dependências Next.js, React 19
│       ├── tsconfig.json       # Configuração TypeScript do Next.js
│       ├── app/
│       │   ├── layout.tsx      # Layout base com fontes e tokens globais
│       │   ├── styles.css      # Design tokens do Figma (--ink, --green-dark, etc.)
│       │   ├── page.tsx        # Redirecionamento inicial
│       │   ├── login/          # Tela de autenticação com contas de demonstração
│       │   └── planos/
│       │       ├── page.tsx    # Listagem de rascunhos do professor logado
│       │       ├── gerar/      # Frame 30:23234 (Formulário) e 30:23541 (Preparando)
│       │       └── [id]/       # Frame 30:23678 (Revisão e Edição de rascunho)
│       ├── components/
│       │   ├── ui/             # Componentes base (Button, Panel, Notice, EmptyState)
│       │   └── plans/          # Form, Editor Markdown, Preview e Modais
│       └── lib/
│           ├── auth.ts         # Gestão de token em memória e interceptor HTTP
│           └── api.ts          # Chamadas tipadas à API NestJS
```

---

## Complexity Tracking

| Violação / Desvio | Por que é necessário | Alternativa mais simples rejeitada por que |
| :--- | :--- | :--- |
| *Nenhuma violação* | *Todos os gates e princípios constitucionais foram estritamente cumpridos* | *N/A* |
