# Contrato de API: Planos de Aula e Geração com IA

**Prefixo Base**: `/api/v1/plans`  
**Acesso**: Autenticado via Bearer JWT. Isolamento estrito por `ownerId`. Tentativas de acesso a planos de outro professor respondem com HTTP 404.

---

## 1. `POST /api/v1/plans/generations`

Dispara o fluxo de geração assistida por IA via backend (integrado ao n8n) e persiste o plano como `RASCUNHO` sob transação atômica. O backend extrai o e-mail do professor autenticado a partir do token Bearer JWT e o repassa como `sessao` ao n8n (conforme [docs/contracts/n8n.md](../../../data/docs/contracts/n8n.md)).

### Requisição
- **Headers**:
  - `Authorization: Bearer <accessToken>`
  - `Content-Type: application/json`
- **Body**:
  ```json
  {
    "skillCodes": ["EF02CO02", "EF02CO04"],
    "instruction": "Criar uma atividade introdutória em dupla com foco em exemplos do cotidiano.",
    "durationMinutes": 50,
    "provisionalTitle": "Sequências que organizam o cotidiano",
    "usesDigitalResources": true
  }
  ```

### Resposta de Sucesso (`201 Created`)
- **Body**:
  ```json
  {
    "id": "7b8e3a21-99c4-42b1-8422-95dbf413a201",
    "title": "Sequências que organizam o cotidiano",
    "markdown": "# Plano de aula\n\n## Objetivos\n- Compreender sequências de passos lógicos...\n\n## Desenvolvimento\n1. Aquecimento (10 min)...\n2. Atividade em duplas (30 min)...\n3. Socialização e fechamento (10 min)...",
    "status": "RASCUNHO",
    "aiAssisted": true,
    "durationMinutes": 50,
    "usesDigitalResources": true,
    "aiRunId": "b182f719-74d1-419b-a641-fcbb3298c910",
    "createdAt": "2026-10-01T21:40:00.000Z",
    "updatedAt": "2026-10-01T21:40:00.000Z"
  }
  ```

### Respostas de Falha
- `400 Bad Request`: Campos obrigatórios inválidos (ex.: duração negativa, lista de códigos vazia).
- `502 Bad Gateway`: Falha no serviço de IA (n8n indisponível, resposta fora do schema ou e-mail de sessão divergente). `AiRun` é marcado como `FAILED` e nenhum plano é persistido.
- `504 Gateway Timeout`: O tempo limite de 60 segundos foi excedido. `AiRun` é marcado como `FAILED` e nenhum plano parcial é persistido.

---

## 2. `GET /api/v1/plans`

Retorna a lista contendo exclusivamente os planos de aula pertencentes ao professor autenticado.

### Requisição
- **Headers**:
  - `Authorization: Bearer <accessToken>`

### Resposta de Sucesso (`200 OK`)
- **Body**:
  ```json
  [
    {
      "id": "7b8e3a21-99c4-42b1-8422-95dbf413a201",
      "title": "Sequências que organizam o cotidiano",
      "status": "RASCUNHO",
      "aiAssisted": true,
      "durationMinutes": 50,
      "usesDigitalResources": true,
      "updatedAt": "2026-10-01T21:40:00.000Z"
    }
  ]
  ```

---

## 3. `GET /api/v1/plans/:id`

Recupera os detalhes completos de um rascunho de plano de aula específico.

### Requisição
- **Headers**:
  - `Authorization: Bearer <accessToken>`

### Resposta de Sucesso (`200 OK`)
- **Body**:
  ```json
  {
    "id": "7b8e3a21-99c4-42b1-8422-95dbf413a201",
    "title": "Sequências que organizam o cotidiano",
    "markdown": "# Plano de aula\n\n## Objetivos\n...",
    "status": "RASCUNHO",
    "aiAssisted": true,
    "durationMinutes": 50,
    "usesDigitalResources": true,
    "createdAt": "2026-10-01T21:40:00.000Z",
    "updatedAt": "2026-10-01T21:40:00.000Z"
  }
  ```

### Respostas de Erro
- `404 Not Found`: Plano inexistente **OU** pertencente a outro professor (isola dados e previne enumeração de identificadores).

---

## 4. `PATCH /api/v1/plans/:id`

Salva explicitamente modificações realizadas pelo professor no rascunho (edição do título ou do conteúdo Markdown).

### Requisição
- **Headers**:
  - `Authorization: Bearer <accessToken>`
  - `Content-Type: application/json`
- **Body**:
  ```json
  {
    "title": "Sequências que organizam o cotidiano (Revisado)",
    "markdown": "# Plano de aula: Sequências que organizam o cotidiano\n\n## Objetivos Revisados\n- Ajustado para a turma do 2º B..."
  }
  ```

### Resposta de Sucesso (`200 OK`)
- **Body**:
  ```json
  {
    "id": "7b8e3a21-99c4-42b1-8422-95dbf413a201",
    "title": "Sequências que organizam o cotidiano (Revisado)",
    "markdown": "# Plano de aula: Sequências que organizam o cotidiano\n\n## Objetivos Revisados\n...",
    "status": "RASCUNHO",
    "aiAssisted": true,
    "durationMinutes": 50,
    "usesDigitalResources": true,
    "updatedAt": "2026-10-01T21:45:00.000Z"
  }
  ```

### Respostas de Erro
- `400 Bad Request`: Payload inválido.
- `404 Not Found`: Plano inexistente ou pertencente a outro professor.
