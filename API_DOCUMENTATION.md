# Documentação Técnica das APIs - MarcaHora Digital

## Visão Geral

O sistema MarcaHora Digital é composto por 5 microserviços independentes que se comunicam via REST API. Cada serviço possui seu próprio banco de dados SQLite (em desenvolvimento) e expõe endpoints HTTP para realizar operações CRUD e funcionalidades específicas.

### Arquitetura de Microserviços

```
┌─────────────────┐    ┌──────────────────────┐    ┌──────────────────┐
│  auth-service   │───▶│ establishments-service│───▶│ services-service │
│   (Port 5001)   │    │     (Port 5002)       │    │   (Port 5003)    │
└─────────────────┘    └──────────────────────┘    └──────────────────┘
         │                      │                           │
         │                      │                           │
         ▼                      ▼                           ▼
┌─────────────────┐    ┌──────────────────────┐    ┌──────────────────┐
│ appointments-   │◀───│  attendants-service   │◀───│                  │
│    service      │    │     (Port 5004)       │    │                  │
│  (Port 5005)    │    └──────────────────────┘    └──────────────────┘
└─────────────────┘
```

### Convenções Gerais

- **Formato de dados**: JSON (request e response)
- **Autenticação**: JWT Bearer Token (via header `Authorization: Bearer <token>`)
- **Códigos HTTP**:
  - `200 OK` - Sucesso na operação GET/PUT
  - `201 Created` - Recurso criado com sucesso
  - `400 Bad Request` - Parâmetros inválidos ou faltando
  - `401 Unauthorized` - Token ausente ou inválido
  - `403 Forbidden` - Sem permissão para acessar o recurso
  - `404 Not Found` - Recurso não encontrado
  - `409 Conflict` - Conflito de dados (ex: email duplicado, horário indisponível)
  - `500 Internal Server Error` - Erro interno do servidor

---

## 1. Auth Service (Porta 5001)

Serviço de autenticação e gerenciamento de usuários (clientes e proprietários).

### Base URL
```
http://localhost:5001
```

### 1.1. Registro de Cliente

**Endpoint**: `POST /auth/client/register`

**Descrição**: Registra um novo cliente usando número de telefone como identificação única.

**Headers**: Nenhum (endpoint público)

**Request Body**:
```json
{
  "phone": "11988888888",
  "name": "João Silva",
  "email": "joao@example.com"  // opcional
}
```

**Response Success (201)**:
```json
{
  "message": "Cliente registrado com sucesso",
  "client": {
    "id": 1,
    "phone": "11988888888",
    "name": "João Silva",
    "email": "joao@example.com",
    "created_at": "2025-12-27T10:00:00",
    "is_active": true,
    "user_type": "client"
  }
}
```

**Erros**:
- `400`: Telefone ou nome não fornecidos
- `409`: Telefone já cadastrado

---

### 1.2. Login de Cliente

**Endpoint**: `POST /auth/client/login`

**Descrição**: Autenticação de cliente via número de telefone.

**Request Body**:
```json
{
  "phone": "11988888888"
}
```

**Response Success (200)**:
```json
{
  "message": "Login realizado com sucesso",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "phone": "11988888888",
    "name": "João Silva",
    "user_type": "client"
  }
}
```

**Erros**:
- `400`: Telefone não fornecido
- `404`: Cliente não encontrado ou inativo

---

### 1.3. Registro de Proprietário

**Endpoint**: `POST /auth/owner/register`

**Descrição**: Registra um novo proprietário/administrador.

**Request Body**:
```json
{
  "email": "dono@estabelecimento.com",
  "password": "senha123",
  "name": "Maria Santos",
  "phone": "11999999999"  // opcional
}
```

**Response Success (201)**:
```json
{
  "message": "Proprietário registrado com sucesso",
  "owner": {
    "id": 1,
    "email": "dono@estabelecimento.com",
    "name": "Maria Santos",
    "phone": "11999999999",
    "is_admin": false,
    "user_type": "owner",
    "created_at": "2025-12-27T10:00:00"
  }
}
```

**Erros**:
- `400`: Campos obrigatórios faltando
- `409`: Email já cadastrado

---

### 1.4. Login de Proprietário

**Endpoint**: `POST /auth/owner/login`

**Descrição**: Autenticação de proprietário via email e senha.

**Request Body**:
```json
{
  "email": "dono@estabelecimento.com",
  "password": "senha123"
}
```

**Response Success (200)**:
```json
{
  "message": "Login realizado com sucesso",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "email": "dono@estabelecimento.com",
    "name": "Maria Santos",
    "is_admin": false,
    "user_type": "owner"
  }
}
```

**Erros**:
- `400`: Email ou senha não fornecidos
- `401`: Credenciais inválidas

---

### 1.5. Validar Token

**Endpoint**: `GET /auth/validate`

**Descrição**: Valida um token JWT e retorna informações do usuário.

**Headers**:
```
Authorization: Bearer <token>
```

**Response Success (200)**:
```json
{
  "valid": true,
  "user": {
    "id": 1,
    "name": "João Silva",
    "user_type": "client"
  }
}
```

**Erros**:
- `401`: Token inválido ou expirado

---

### 1.6. Obter Usuário Atual

**Endpoint**: `GET /auth/me`

**Descrição**: Retorna informações do usuário autenticado.

**Headers**:
```
Authorization: Bearer <token>
```

**Response Success (200)**:
```json
{
  "user": {
    "id": 1,
    "phone": "11988888888",
    "name": "João Silva",
    "email": "joao@example.com",
    "user_type": "client",
    "created_at": "2025-12-27T10:00:00"
  }
}
```

---

### 1.7. Health Check

**Endpoint**: `GET /health`

**Response Success (200)**:
```json
{
  "status": "healthy",
  "service": "auth-service"
}
```

---

## 2. Establishments Service (Porta 5002)

Gerenciamento de estabelecimentos e seus horários de funcionamento.

### Base URL
```
http://localhost:5002
```

### 2.1. Criar Estabelecimento

**Endpoint**: `POST /establishments`

**Autenticação**: Requer token de owner

**Request Body**:
```json
{
  "name": "Barbearia Exemplo",
  "description": "A melhor barbearia da cidade",
  "business_type": "barbershop",
  "address": "Rua Teste, 123 - Centro",
  "phone": "11999999999",
  "email": "contato@barbearia.com",
  "working_hours": "{\"monday\": {\"open\": \"09:00\", \"close\": \"18:00\", \"is_open\": true}}"
}
```

**Response Success (201)**:
```json
{
  "message": "Estabelecimento criado com sucesso",
  "establishment": {
    "id": 1,
    "owner_id": 1,
    "name": "Barbearia Exemplo",
    "business_type": "barbershop",
    "address": "Rua Teste, 123 - Centro",
    "is_active": true,
    "created_at": "2025-12-27T10:00:00"
  }
}
```

**Tipos de negócio válidos**: `barbershop`, `salon`, `clinic`, `petshop`, `gym`, `spa`, `other`

---

### 2.2. Listar Estabelecimentos

**Endpoint**: `GET /establishments`

**Query Parameters**:
- `owner_id` (int): Filtrar por proprietário
- `business_type` (string): Filtrar por tipo de negócio
- `is_active` (boolean): Filtrar por status ativo/inativo

**Exemplo**: `GET /establishments?business_type=barbershop&is_active=true`

**Response Success (200)**:
```json
{
  "establishments": [
    {
      "id": 1,
      "owner_id": 1,
      "name": "Barbearia Exemplo",
      "business_type": "barbershop",
      "is_active": true
    }
  ],
  "count": 1
}
```

---

### 2.3. Obter Estabelecimento por ID

**Endpoint**: `GET /establishments/<establishment_id>`

**Response Success (200)**:
```json
{
  "id": 1,
  "owner_id": 1,
  "name": "Barbearia Exemplo",
  "description": "A melhor barbearia da cidade",
  "business_type": "barbershop",
  "address": "Rua Teste, 123 - Centro",
  "phone": "11999999999",
  "email": "contato@barbearia.com",
  "working_hours": "{...}",
  "is_active": true,
  "created_at": "2025-12-27T10:00:00"
}
```

---

### 2.4. Atualizar Estabelecimento

**Endpoint**: `PUT /establishments/<establishment_id>`

**Autenticação**: Requer token de owner (apenas dono do estabelecimento)

**Request Body**: Campos a serem atualizados (todos opcionais)
```json
{
  "name": "Nova Barbearia",
  "description": "Descrição atualizada",
  "address": "Novo endereço"
}
```

**Response Success (200)**:
```json
{
  "message": "Estabelecimento atualizado com sucesso",
  "establishment": { /* dados atualizados */ }
}
```

---

### 2.5. Deletar Estabelecimento

**Endpoint**: `DELETE /establishments/<establishment_id>`

**Autenticação**: Requer token de owner (apenas dono do estabelecimento)

**Descrição**: Realiza soft delete (marca como inativo)

**Response Success (200)**:
```json
{
  "message": "Estabelecimento deletado com sucesso"
}
```

---

### 2.6. Listar Estabelecimentos por Owner

**Endpoint**: `GET /establishments/owner/<owner_id>`

**Response Success (200)**:
```json
{
  "establishments": [ /* lista de estabelecimentos */ ],
  "count": 2
}
```

---

### 2.7. Atualizar Horário de Funcionamento

**Endpoint**: `PUT /establishments/<establishment_id>/working-hours`

**Autenticação**: Requer token de owner

**Request Body**:
```json
{
  "working_hours": {
    "monday": {
      "open": "09:00",
      "close": "18:00",
      "is_open": true
    },
    "tuesday": {
      "open": "09:00",
      "close": "18:00",
      "is_open": true
    },
    "sunday": {
      "is_open": false
    }
  }
}
```

**Response Success (200)**:
```json
{
  "message": "Horários de funcionamento atualizados com sucesso",
  "working_hours": { /* horários atualizados */ }
}
```

---

### 2.8. Obter Horário de Funcionamento

**Endpoint**: `GET /establishments/<establishment_id>/working-hours`

**Response Success (200)**:
```json
{
  "establishment_id": 1,
  "working_hours": {
    "monday": { "open": "09:00", "close": "18:00", "is_open": true }
  }
}
```

---

### 2.9. Listar Dias Especiais

**Endpoint**: `GET /establishments/<establishment_id>/special-days`

**Query Parameters**:
- `start_date` (YYYY-MM-DD): Data inicial para filtrar
- `end_date` (YYYY-MM-DD): Data final para filtrar

**Response Success (200)**:
```json
{
  "special_days": [
    {
      "id": 1,
      "establishment_id": 1,
      "date": "2025-12-25",
      "description": "Natal",
      "is_open": false
    }
  ]
}
```

---

### 2.10. Criar Dia Especial

**Endpoint**: `POST /establishments/<establishment_id>/special-days`

**Autenticação**: Requer token de owner

**Request Body**:
```json
{
  "date": "2025-12-25",
  "description": "Natal",
  "is_open": false
}
```

ou com horário especial:

```json
{
  "date": "2025-12-24",
  "description": "Véspera de Natal",
  "is_open": true,
  "opening_time": "09:00",
  "closing_time": "12:00"
}
```

**Response Success (201)**:
```json
{
  "message": "Dia especial criado com sucesso",
  "special_day": { /* dados do dia especial */ }
}
```

---

### 2.11. Atualizar Dia Especial

**Endpoint**: `PUT /establishments/<establishment_id>/special-days/<special_day_id>`

**Autenticação**: Requer token de owner

---

### 2.12. Deletar Dia Especial

**Endpoint**: `DELETE /establishments/<establishment_id>/special-days/<special_day_id>`

**Autenticação**: Requer token de owner

---

## 3. Services Service (Porta 5003)

Gerenciamento de templates de serviços e serviços específicos de estabelecimentos.

### Base URL
```
http://localhost:5003
```

### 3.1. Listar Templates de Serviços

**Endpoint**: `GET /services/templates`

**Query Parameters**:
- `category` (string): Filtrar por categoria (hair, beard, nails, massage, etc.)
- `is_active` (boolean): Filtrar por status ativo

**Response Success (200)**:
```json
{
  "templates": [
    {
      "id": 1,
      "name": "Corte Masculino",
      "description": "Corte de cabelo masculino tradicional",
      "category": "hair",
      "default_duration": 30,
      "default_price": 50.00,
      "is_active": true
    }
  ],
  "count": 1
}
```

---

### 3.2. Obter Template por ID

**Endpoint**: `GET /services/templates/<template_id>`

**Response Success (200)**:
```json
{
  "id": 1,
  "name": "Corte Masculino",
  "category": "hair",
  "default_duration": 30,
  "default_price": 50.00
}
```

---

### 3.3. Criar Template

**Endpoint**: `POST /services/templates`

**Autenticação**: Requer token de owner

**Request Body**:
```json
{
  "name": "Corte Feminino",
  "description": "Corte de cabelo feminino",
  "category": "hair",
  "default_duration": 45,
  "default_price": 70.00
}
```

**Response Success (201)**:
```json
{
  "message": "Template criado com sucesso",
  "template": { /* dados do template */ }
}
```

---

### 3.4. Atualizar Template

**Endpoint**: `PUT /services/templates/<template_id>`

**Autenticação**: Requer token de owner (apenas admins)

---

### 3.5. Deletar Template

**Endpoint**: `DELETE /services/templates/<template_id>`

**Autenticação**: Requer token de owner (apenas admins)

---

### 3.6. Criar Serviço para Estabelecimento

**Endpoint**: `POST /services`

**Autenticação**: Requer token de owner

**Request Body (baseado em template)**:
```json
{
  "establishment_id": 1,
  "template_id": 1,
  "name": "Corte Masculino Premium",
  "duration": 45,
  "price": 80.00,
  "description": "Corte com acabamento especial"
}
```

**Request Body (serviço customizado)**:
```json
{
  "establishment_id": 1,
  "name": "Serviço Especial",
  "duration": 60,
  "price": 100.00,
  "description": "Serviço totalmente customizado"
}
```

**Response Success (201)**:
```json
{
  "message": "Serviço criado com sucesso",
  "service": {
    "id": 1,
    "establishment_id": 1,
    "template_id": 1,
    "name": "Corte Masculino Premium",
    "duration": 45,
    "price": 80.00
  }
}
```

---

### 3.7. Listar Serviços de um Estabelecimento

**Endpoint**: `GET /services/establishment/<establishment_id>`

**Query Parameters**:
- `is_active` (boolean): Filtrar por status ativo

**Response Success (200)**:
```json
{
  "services": [
    {
      "id": 1,
      "establishment_id": 1,
      "name": "Corte Masculino Premium",
      "duration": 45,
      "price": 80.00,
      "is_active": true
    }
  ],
  "count": 1
}
```

---

### 3.8. Obter Serviço por ID

**Endpoint**: `GET /services/<service_id>`

---

### 3.9. Atualizar Serviço

**Endpoint**: `PUT /services/<service_id>`

**Autenticação**: Requer token de owner (apenas dono do estabelecimento)

---

### 3.10. Deletar Serviço

**Endpoint**: `DELETE /services/<service_id>`

**Autenticação**: Requer token de owner (apenas dono do estabelecimento)

---

## 4. Attendants Service (Porta 5004)

Gerenciamento de atendentes e suas associações com serviços.

### Base URL
```
http://localhost:5004
```

### 4.1. Criar Atendente

**Endpoint**: `POST /attendants`

**Autenticação**: Requer token de owner

**Request Body**:
```json
{
  "establishment_id": 1,
  "name": "João Silva",
  "email": "joao@barbearia.com",
  "phone": "11987654321",
  "specialties": "Especialista em cortes masculinos e barba"
}
```

**Response Success (201)**:
```json
{
  "message": "Atendente criado com sucesso",
  "attendant": {
    "id": 1,
    "establishment_id": 1,
    "name": "João Silva",
    "email": "joao@barbearia.com",
    "is_active": true
  }
}
```

---

### 4.2. Listar Atendentes

**Endpoint**: `GET /attendants`

**Query Parameters**:
- `establishment_id` (int): Filtrar por estabelecimento
- `is_active` (boolean): Filtrar por status ativo

**Response Success (200)**:
```json
{
  "attendants": [
    {
      "id": 1,
      "establishment_id": 1,
      "name": "João Silva",
      "is_active": true
    }
  ],
  "count": 1
}
```

---

### 4.3. Obter Atendente por ID

**Endpoint**: `GET /attendants/<attendant_id>`

**Descrição**: Retorna detalhes do atendente com seus serviços associados.

**Response Success (200)**:
```json
{
  "id": 1,
  "establishment_id": 1,
  "name": "João Silva",
  "email": "joao@barbearia.com",
  "specialties": "Especialista em cortes masculinos",
  "services": [
    {
      "id": 1,
      "name": "Corte Masculino Premium",
      "price": 80.00
    }
  ]
}
```

---

### 4.4. Atualizar Atendente

**Endpoint**: `PUT /attendants/<attendant_id>`

**Autenticação**: Requer token de owner

---

### 4.5. Deletar Atendente

**Endpoint**: `DELETE /attendants/<attendant_id>`

**Autenticação**: Requer token de owner

**Descrição**: Soft delete + remove todas as associações com serviços

---

### 4.6. Listar Atendentes por Estabelecimento

**Endpoint**: `GET /attendants/establishment/<establishment_id>`

---

### 4.7. Associar Serviço ao Atendente

**Endpoint**: `POST /attendants/<attendant_id>/services`

**Autenticação**: Requer token de owner

**Request Body**:
```json
{
  "service_id": 1
}
```

**Response Success (201)**:
```json
{
  "message": "Serviço associado ao atendente com sucesso",
  "association": {
    "attendant_id": 1,
    "service_id": 1
  }
}
```

**Erros**:
- `409`: Serviço já associado ao atendente
- `400`: Serviço não pertence ao mesmo estabelecimento

---

### 4.8. Remover Associação de Serviço

**Endpoint**: `DELETE /attendants/<attendant_id>/services/<service_id>`

**Autenticação**: Requer token de owner

---

### 4.9. Listar Atendentes que Prestam um Serviço

**Endpoint**: `GET /attendants/service/<service_id>`

**Response Success (200)**:
```json
{
  "attendants": [
    {
      "id": 1,
      "name": "João Silva",
      "establishment_id": 1
    }
  ]
}
```

---

## 5. Appointments Service (Porta 5005)

Gerenciamento de agendamentos e busca de estabelecimentos.

### Base URL
```
http://localhost:5005
```

### 5.1. Buscar Estabelecimentos

**Endpoint**: `GET /search/establishments`

**Query Parameters**:
- `business_type` (string): Filtrar por tipo de negócio
- `service_name` (string): Filtrar estabelecimentos que oferecem serviço específico

**Exemplo**: `GET /search/establishments?business_type=barbershop&service_name=corte`

**Response Success (200)**:
```json
{
  "establishments": [
    {
      "id": 1,
      "name": "Barbearia Exemplo",
      "business_type": "barbershop",
      "address": "Rua Teste, 123",
      "phone": "11999999999"
    }
  ]
}
```

---

### 5.2. Obter Serviços de um Estabelecimento

**Endpoint**: `GET /search/establishments/<establishment_id>/services`

**Response Success (200)**:
```json
{
  "services": [
    {
      "id": 1,
      "name": "Corte Masculino Premium",
      "price": 80.00,
      "duration": 45
    }
  ]
}
```

---

### 5.3. Obter Atendentes de um Estabelecimento

**Endpoint**: `GET /search/establishments/<establishment_id>/attendants`

---

### 5.4. Obter Atendentes que Prestam um Serviço

**Endpoint**: `GET /search/service/<service_id>/attendants`

---

### 5.5. Criar Agendamento

**Endpoint**: `POST /appointments`

**Autenticação**: Requer token (client ou owner)

**Request Body**:
```json
{
  "establishment_id": 1,
  "service_id": 1,
  "attendant_id": 1,
  "appointment_date": "2025-12-28",
  "appointment_time": "10:00",
  "notes": "Cliente prefere corte com máquina 2"
}
```

**Response Success (201)**:
```json
{
  "message": "Agendamento criado com sucesso",
  "appointment": {
    "id": 1,
    "client_id": 2,
    "establishment_id": 1,
    "service_id": 1,
    "attendant_id": 1,
    "appointment_date": "2025-12-28",
    "appointment_time": "10:00",
    "duration": 45,
    "status": "pending"
  }
}
```

**Validações**:
- Verifica se serviço pertence ao estabelecimento
- Verifica se atendente pertence ao estabelecimento
- Verifica se atendente presta o serviço
- Verifica conflito de horário

**Erros**:
- `409`: Horário não disponível (conflito)
- `400`: Dados inválidos ou incompatíveis

---

### 5.6. Listar Agendamentos

**Endpoint**: `GET /appointments`

**Autenticação**: Requer token (client ou owner)

**Query Parameters**:
- `establishment_id` (int): Filtrar por estabelecimento
- `client_id` (int): Filtrar por cliente
- `status` (string): Filtrar por status (pending, confirmed, cancelled, completed)
- `start_date` (YYYY-MM-DD): Data inicial
- `end_date` (YYYY-MM-DD): Data final

**Comportamento**:
- Clientes veem apenas seus próprios agendamentos
- Owners veem agendamentos dos seus estabelecimentos

**Response Success (200)**:
```json
{
  "appointments": [
    {
      "id": 1,
      "client_id": 2,
      "establishment_id": 1,
      "service_id": 1,
      "appointment_date": "2025-12-28",
      "appointment_time": "10:00",
      "status": "confirmed"
    }
  ],
  "count": 1
}
```

---

### 5.7. Obter Agendamento por ID

**Endpoint**: `GET /appointments/<appointment_id>`

**Autenticação**: Requer token

**Permissões**:
- Cliente pode ver apenas seus próprios agendamentos
- Owner pode ver agendamentos dos seus estabelecimentos

---

### 5.8. Atualizar Agendamento

**Endpoint**: `PUT /appointments/<appointment_id>`

**Autenticação**: Requer token

**Request Body** (campos opcionais):
```json
{
  "appointment_date": "2025-12-29",
  "appointment_time": "14:00",
  "attendant_id": 2,
  "status": "confirmed",
  "notes": "Nova observação"
}
```

**Permissões**:
- Cliente pode reagendar ou cancelar seus próprios agendamentos
- Owner pode modificar qualquer campo dos agendamentos dos seus estabelecimentos

**Validações**:
- Verifica conflito de horário se data/hora for alterada
- Verifica se novo atendente pertence ao estabelecimento e presta o serviço

---

### 5.9. Cancelar Agendamento

**Endpoint**: `DELETE /appointments/<appointment_id>`

**Autenticação**: Requer token

**Descrição**: Marca o agendamento como cancelado (não deleta do banco)

**Response Success (200)**:
```json
{
  "message": "Agendamento cancelado com sucesso"
}
```

---

### 5.10. Listar Agendamentos de um Cliente

**Endpoint**: `GET /appointments/client/<client_id>`

**Autenticação**: Requer token

**Permissões**: Apenas o próprio cliente ou admin

---

### 5.11. Obter Agendamentos para Calendário

**Endpoint**: `GET /appointments/calendar`

**Autenticação**: Requer token

**Query Parameters**:
- `year` (int): Ano (obrigatório)
- `month` (int): Mês (obrigatório)
- `establishment_id` (int): Filtrar por estabelecimento (opcional para owners)

**Response Success (200)**:
```json
{
  "year": 2025,
  "month": 12,
  "appointments_by_day": {
    "1": [
      {
        "id": 1,
        "appointment_time": "10:00",
        "status": "confirmed",
        "service_name": "Corte Masculino Premium"
      }
    ],
    "15": [
      {
        "id": 2,
        "appointment_time": "14:00",
        "status": "pending"
      }
    ]
  }
}
```

---

## Fluxo de Autenticação

### Para Clientes

1. Cliente registra via `POST /auth/client/register` com telefone
2. Cliente faz login via `POST /auth/client/login` recebendo JWT token
3. Cliente usa o token em todas as requisições protegidas

### Para Proprietários

1. Owner registra via `POST /auth/owner/register` com email/senha
2. Owner faz login via `POST /auth/owner/login` recebendo JWT token
3. Owner usa o token em todas as requisições protegidas

### Validação de Token

Todos os endpoints protegidos validam o token chamando internamente `GET /auth/validate` do auth-service.

---

## Comunicação Entre Microserviços

Os microserviços se comunicam via HTTP REST:

1. **appointments-service** valida tokens com **auth-service**
2. **appointments-service** busca dados de estabelecimentos no **establishments-service**
3. **appointments-service** busca dados de serviços no **services-service**
4. **appointments-service** busca dados de atendentes no **attendants-service**
5. **services-service** valida ownership de estabelecimento com **establishments-service**
6. **attendants-service** valida serviços com **services-service** e estabelecimentos com **establishments-service**

Todas as validações são feitas via chamadas HTTP síncronas.

---

## Tratamento de Erros Padronizado

Todos os serviços retornam erros no formato:

```json
{
  "error": "Mensagem descritiva do erro"
}
```

### Códigos HTTP de Erro Comuns

- **400 Bad Request**: Parâmetros faltando ou formato inválido
- **401 Unauthorized**: Token ausente, inválido ou expirado
- **403 Forbidden**: Usuário não tem permissão para acessar o recurso
- **404 Not Found**: Recurso não encontrado
- **409 Conflict**: Conflito de dados (ex: email duplicado, horário ocupado)
- **500 Internal Server Error**: Erro inesperado no servidor

---

## Exemplos de Uso Completo

### Exemplo 1: Cliente Agendando um Serviço

```bash
# 1. Registrar cliente
curl -X POST http://localhost:5001/auth/client/register \
  -H "Content-Type: application/json" \
  -d '{"phone":"11988888888","name":"João Silva"}'

# 2. Fazer login
curl -X POST http://localhost:5001/auth/client/login \
  -H "Content-Type: application/json" \
  -d '{"phone":"11988888888"}'
# Resposta: { "token": "..." }

# 3. Buscar estabelecimentos
curl -X GET "http://localhost:5005/search/establishments?business_type=barbershop"

# 4. Ver serviços do estabelecimento
curl -X GET http://localhost:5005/search/establishments/1/services

# 5. Criar agendamento
curl -X POST http://localhost:5005/appointments \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "establishment_id": 1,
    "service_id": 1,
    "attendant_id": 1,
    "appointment_date": "2025-12-28",
    "appointment_time": "10:00"
  }'
```

### Exemplo 2: Owner Criando um Estabelecimento

```bash
# 1. Registrar owner
curl -X POST http://localhost:5001/auth/owner/register \
  -H "Content-Type: application/json" \
  -d '{"email":"dono@barbearia.com","password":"senha123","name":"Maria"}'

# 2. Fazer login
curl -X POST http://localhost:5001/auth/owner/login \
  -H "Content-Type: application/json" \
  -d '{"email":"dono@barbearia.com","password":"senha123"}'

# 3. Criar estabelecimento
curl -X POST http://localhost:5002/establishments \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Minha Barbearia",
    "business_type": "barbershop",
    "address": "Rua ABC, 123"
  }'

# 4. Criar serviço
curl -X POST http://localhost:5003/services \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "establishment_id": 1,
    "template_id": 1,
    "name": "Corte Masculino",
    "duration": 30,
    "price": 50.00
  }'
```

---

## Variáveis de Ambiente

Cada serviço pode ser configurado via variáveis de ambiente:

### auth-service
```
SECRET_KEY=<chave-secreta-jwt>
AUTH_SERVICE_PORT=5001
```

### establishments-service
```
AUTH_SERVICE_URL=http://localhost:5001
ESTABLISHMENTS_SERVICE_PORT=5002
```

### services-service
```
AUTH_SERVICE_URL=http://localhost:5001
ESTABLISHMENTS_SERVICE_URL=http://localhost:5002
SERVICES_SERVICE_PORT=5003
```

### attendants-service
```
AUTH_SERVICE_URL=http://localhost:5001
ESTABLISHMENTS_SERVICE_URL=http://localhost:5002
SERVICES_SERVICE_URL=http://localhost:5003
ATTENDANTS_SERVICE_PORT=5004
```

### appointments-service
```
AUTH_SERVICE_URL=http://localhost:5001
ESTABLISHMENTS_SERVICE_URL=http://localhost:5002
SERVICES_SERVICE_URL=http://localhost:5003
ATTENDANTS_SERVICE_URL=http://localhost:5004
APPOINTMENTS_SERVICE_PORT=5005
```

---

## Versionamento da API

**Versão Atual**: v1.0.0 (implícita, sem prefixo nos endpoints)

Futuras versões poderão usar prefixo `/v2/` nos endpoints para manter compatibilidade.

---

## Documentação Gerada

Data: 2025-12-27
Status: Atualizada com todos os microserviços implementados
