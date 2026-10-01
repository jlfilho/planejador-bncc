# Contrato de API: Catálogo de Habilidades BNCC

**Prefixo Base**: `/api/v1/bncc`  
**Acesso**: Autenticado (Bearer JWT de qualquer conta com papel `PROFESSOR` ou `ADMIN`). Somente leitura.

---

## 1. `GET /api/v1/bncc/skills`

Consulta habilidades ativas da BNCC com suporte a filtros combinados e busca textual.

### Requisição
- **Headers**:
  - `Authorization: Bearer <accessToken>`
- **Query Parameters**:
  - `search` *(opcional, string)*: Termo para busca em código ou descrição curricular (ex.: `"EF02CO"` ou `"algoritmos"`).
  - `ano` *(opcional, integer)*: Ano escolar específico (ex.: `1` ou `2`).
  - `eixo` *(opcional, string)*: Filtrar por eixo temático (ex.: `"Pensamento Computacional (PC)"`, `"Mundo Digital (MD)"`, `"Cultura Digital (CD)"`).
  - `nivel` *(opcional, string)*: Nível educacional (ex.: `"Ensino Fundamental"`).

### Resposta de Sucesso (`200 OK`)
- **Body**:
  ```json
  [
    {
      "id": "e45f94d2-2821-49b0-9b43-7e4726b21aa1",
      "codigo": "EF02CO02",
      "nivel": "Ensino Fundamental",
      "ano": 2,
      "eixo": "Pensamento Computacional (PC)",
      "descricao": "Criar e simular algoritmos representados em linguagem oral, escrita ou pictográfica, construídos como sequências com repetições simples...",
      "explicacao": "Usar linguagem oral, textual ou pictográfica para descrever algoritmos...",
      "exemplos": "Os alunos podem construir algoritmos com conjuntos de instruções pré-definidas...",
      "ativa": true
    },
    {
      "id": "a91b2c3d-1122-3344-5566-778899aabbcc",
      "codigo": "EF02CO04",
      "nivel": "Ensino Fundamental",
      "ano": 2,
      "eixo": "Mundo Digital (MD)",
      "descricao": "Diferenciar componentes físicos (hardware) e programas que fornecem as instruções (software) para o hardware.",
      "explicacao": "O objetivo da habilidade é mostrar aos alunos que em seu cotidiano existem dispositivos físicos...",
      "exemplos": "Pode-se utilizar dispositivos do cotidiano do aluno para diferenciar o dispositivo físico daquilo que o controla.",
      "ativa": true
    }
  ]
  ```

### Respostas de Erro
- `401 Unauthorized`: Token ausente ou inválido.
- `400 Bad Request`: Parâmetros de consulta inválidos (ex.: `ano` não numérico).
