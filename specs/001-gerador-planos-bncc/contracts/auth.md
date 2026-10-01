# Contrato de API: Autenticação e Gestão de Sessão

**Prefixo Base**: `/api/v1/auth`  
**Segurança**: Senhas em hash Argon2, Access Token em memória (Bearer JWT), Refresh Token em cookie HttpOnly com hash SHA-256 no banco.

---

## 1. `POST /api/v1/auth/login`

Autentica o professor utilizando credenciais de demonstração.

### Requisição
- **Headers**: `Content-Type: application/json`
- **Body**:
  ```json
  {
    "email": "ana.souza@escola.gov.br",
    "password": "SenhaSegura123!"
  }
  ```

### Resposta de Sucesso (`200 OK`)
- **Headers**:
  - `Set-Cookie: refreshToken=<token_aleatorio>; HttpOnly; SameSite=Lax; Path=/api/v1/auth/refresh; Max-Age=604800` (e `Secure` em produção)
- **Body**:
  ```json
  {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 900,
    "user": {
      "id": "c1f8a840-7e18-47bc-bbbb-f213df65d101",
      "name": "Ana Souza",
      "email": "ana.souza@escola.gov.br",
      "role": "PROFESSOR"
    }
  }
  ```

### Respostas de Erro
- `400 Bad Request`: Payload fora do schema de validação (ex.: e-mail malformado ou senha ausente).
- `401 Unauthorized`: Mensagem genérica `"Credenciais inválidas"` (para evitar enumeração de contas existentes).

---

## 2. `POST /api/v1/auth/refresh`

Rotaciona o token de atualização e emite um novo Access Token de curta duração.

### Requisição
- **Headers**:
  - `Cookie: refreshToken=<token_aleatorio>`
  - `Content-Type: application/json`

### Resposta de Sucesso (`200 OK`)
- **Headers**:
  - `Set-Cookie: refreshToken=<novo_token_aleatorio>; HttpOnly; SameSite=Lax; Path=/api/v1/auth/refresh; Max-Age=604800`
- **Body**:
  ```json
  {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 900
  }
  ```

### Respostas de Erro
- `401 Unauthorized`: Cookie ausente, token expirado, inválido ou já revogado.

---

## 3. `POST /api/v1/auth/logout`

Encerra a sessão ativa do professor, marcando o refresh token como revogado no banco e limpando o cookie no cliente.

### Requisição
- **Headers**:
  - `Authorization: Bearer <accessToken>` (opcional se token expirado)
  - `Cookie: refreshToken=<token_aleatorio>`

### Resposta de Sucesso (`200 OK`)
- **Headers**:
  - `Set-Cookie: refreshToken=; HttpOnly; SameSite=Lax; Path=/api/v1/auth/refresh; Max-Age=0`
- **Body**:
  ```json
  {
    "success": true,
    "message": "Sessão encerrada com sucesso."
  }
  ```

---

## 4. `GET /api/v1/auth/me`

Retorna os dados do professor autenticado para sincronização do estado da aplicação web.

### Requisição
- **Headers**:
  - `Authorization: Bearer <accessToken>`

### Resposta de Sucesso (`200 OK`)
- **Body**:
  ```json
  {
    "id": "c1f8a840-7e18-47bc-bbbb-f213df65d101",
    "name": "Ana Souza",
    "email": "ana.souza@escola.gov.br",
    "role": "PROFESSOR"
  }
  ```

### Respostas de Erro
- `401 Unauthorized`: Token Bearer ausente, expirado ou com assinatura inválida.
