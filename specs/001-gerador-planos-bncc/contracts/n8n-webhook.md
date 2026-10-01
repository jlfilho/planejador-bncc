# Contrato Máquina-a-Máquina: Backend API ↔ n8n Webhook

**Direção**: `NestJS API (apps/api)` → `n8n Webhook`  
**Endpoint Padrão**: `POST https://n8n-mestrado.tecnocomp.cloud/webhook/agente-planejador-bncc` (parametrizável via `N8N_WEBHOOK_URL`)  
**Fonte da Verdade**: Alinhado e confirmado com [data/docs/contracts/n8n.md](../../../data/docs/contracts/n8n.md)  
**Segurança**: O frontend NUNCA acessa ou conhece este endpoint nem o segredo de integração.

---

## 1. Transporte e Cabeçalhos Obrigatórios

- **Método**: `POST`
- **Protocolo**: HTTPS obrigatório
- **Cabeçalhos HTTP**:
  - `Content-Type: application/json; charset=utf-8`
  - `x-api-key: <N8N_INTEGRATION_SECRET>` (Credencial HTTP Header Auth configurada no workflow do n8n)
  - `X-Request-Id: <UUIDv4>` (Identificador único para correlação e rastreabilidade da execução no `AiRun`)

---

## 2. Payload Enviado pelo Backend

```json
{
  "sessao": "ana.souza@escola.gov.br",
  "habilidade": "EF02CO02 — Criar e simular algoritmos representados em linguagem oral, escrita ou pictográfica...\n\nEF02CO04 — Diferenciar componentes físicos (hardware) e programas...",
  "instrucao": "Criar uma atividade introdutória em dupla com foco em exemplos do cotidiano.",
  "duracao": 50,
  "recursos_digitais": true
}
```

### Regras Estritas de Composição dos Campos
1. **`sessao`**: **E-mail do professor autenticado** (`user.email`), extraído exclusivamente do token Bearer JWT no backend. Nunca aceita identificador vindo do formulário web.
2. **`habilidade`**: String contendo a habilidade oficial no formato `CÓDIGO — descrição oficial fornecida pelo instrutor`. Quando houver mais de uma habilidade selecionada, cada item é concatenado e separado por linha em branco (`\n\n`).
3. **`instrucao`**: Contexto pedagógico não vazio preenchido pelo docente.
4. **`duracao`**: Número inteiro positivo representando os minutos de aula (ex.: `50`).
5. **`recursos_digitais`**: Booleano obrigatório (`true` ou `false`).

---

## 3. Contrato de Resposta Aceito pela API

O backend considera a requisição um sucesso **estritamente se** o status HTTP for `200 OK` e o JSON retornado satisfizer o seguinte formato confirmado:

```json
{
  "success": true,
  "sessao": "ana.souza@escola.gov.br",
  "habilidade": "EF02CO02 — Criar e simular algoritmos...",
  "answer": "# Plano de aula\nConteúdo do rascunho estruturado...",
  "format": "markdown"
}
```

### Critérios de Validação Estrita (Schema Guard)
- `success`: DEVE ser exatamente o booleano `true`.
- `sessao`: DEVE ecoar exatamente o mesmo e-mail do professor enviado na requisição (`res.sessao === req.sessao`).
- `habilidade`: DEVE ecoar o texto das habilidades enviadas.
- `answer`: DEVE ser uma string não vazia contendo o conteúdo didático em Markdown.
- `format`: DEVE ser exatamente a string `"markdown"`.

---

## 4. Tratamento de Falhas, Timeout e Atomicidade

| Cenário de Erro | Status HTTP Retornado ao Frontend | Ação no Banco de Dados | Comportamento na Interface Web |
| :--- | :---: | :--- | :--- |
| **Timeout (> 60 segundos)** | `504 Gateway Timeout` | `AiRun` atualizado para `FAILED` com `failureCode: TIMEOUT`; **0 planos criados** | Alerta de timeout; campos preservados; botão liberado para reenvio manual |
| **Resposta Inválida / Fora de Schema** | `502 Bad Gateway` | `AiRun` atualizado para `FAILED` com `failureCode: SCHEMA_MISMATCH`; **0 planos criados** | Alerta de formato inválido; formulário preservado |
| **Divergência de Sessão (`sessao != user.email`)** | `502 Bad Gateway` | `AiRun` atualizado para `FAILED` com `failureCode: SESSION_MISMATCH`; **0 planos criados** | Alerta de erro de sessão da IA; formulário preservado |
| **Erro HTTP 4xx/5xx do n8n** | `502 Bad Gateway` | `AiRun` atualizado para `FAILED` com `failureCode: N8N_HTTP_ERROR`; **0 planos criados** | Alerta de indisponibilidade da IA; formulário preservado |
| **Sem Retry Automático** | — | — | Não dispara retry em background; aguarda exclusivamente ação manual do professor |

---

## 5. Modo Mock Local (`N8N_MOCK_ENABLED=true`) e Testes

Para garantir desenvolvimento autônomo e execução de testes automatizados sem consumir o workflow compartilhado:
- **Ativação**: Variável `N8N_MOCK_ENABLED="true"` em `apps/api/.env`.
- **Comportamento em Sucesso**:
  - Intercepta a requisição e valida os campos de entrada.
  - Aguarda delay realista de ~800ms (simulação assíncrona).
  - Retorna HTTP 200 com:
    - `success: true`
    - `sessao`: e-mail informado no payload (`req.sessao`)
    - `habilidade`: texto informado no payload (`req.habilidade`)
    - `answer`: rascunho pedagógico pré-definido válido em Markdown com objetivos, introdução, desenvolvimento e avaliação
    - `format`: `"markdown"`
- **Simulação de Erros**:
  - Se `instrucao` contiver `"[SIMULAR_TIMEOUT]"`, o mock lança timeout abortado para validar o retorno 504 e o status `FAILED` no `AiRun`.
  - Se `instrucao` contiver `"[SIMULAR_ERRO]"`, o mock retorna payload incompatível (ex.: `success: false` ou `format: "text"`) para validar o retorno 502 e o descarte atômico.
