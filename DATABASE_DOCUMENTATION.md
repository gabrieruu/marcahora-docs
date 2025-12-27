# Documentação Técnica do Banco de Dados - MarcaHora Digital

## Visão Geral

O sistema MarcaHora Digital utiliza uma arquitetura de **microserviços com banco de dados por serviço**. Cada microserviço possui seu próprio banco de dados SQLite (em desenvolvimento, migração para PostgreSQL planejada para produção) para garantir baixo acoplamento e alta autonomia.

### Princípios Arquiteturais

1. **Database per Service**: Cada microserviço possui seu próprio banco de dados isolado
2. **Logical Foreign Keys**: Relacionamentos entre serviços são mantidos via IDs lógicos (sem constraints de FK no banco)
3. **Service Communication**: Validação de integridade referencial feita via APIs REST
4. **Soft Deletes**: Uso de flag `is_active` ao invés de DELETE físico

---

## Diagrama de Relacionamento Geral

```
┌─────────────────────────────────────────────────────────────────┐
│                        AUTH SERVICE                              │
│  ┌──────────┐                           ┌──────────┐            │
│  │  Client  │                           │  Owner   │            │
│  └────┬─────┘                           └────┬─────┘            │
│       │                                      │                   │
└───────┼──────────────────────────────────────┼───────────────────┘
        │                                      │
        │                                      │
┌───────┼──────────────────────────────────────┼───────────────────┐
│       │            ESTABLISHMENTS SERVICE    │                   │
│       │                                      │                   │
│       │              ┌──────────────────┐    │(owner_id)         │
│       │              │ Establishment    │◀───┘                   │
│       │              └────┬────┬────────┘                        │
│       │                   │    │                                 │
│       │                   │    │   ┌─────────────────┐           │
│       │                   │    └───│  SpecialDay     │           │
│       │                   │        └─────────────────┘           │
└───────┼───────────────────┼──────────────────────────────────────┘
        │                   │
        │                   │(establishment_id)
┌───────┼───────────────────┼──────────────────────────────────────┐
│       │   SERVICES SERVICE│                                      │
│       │                   │                                      │
│       │   ┌───────────────▼──────────┐    ┌──────────────────┐  │
│       │   │     Service              │◀───│ ServiceTemplate  │  │
│       │   └───────────┬──────────────┘    └──────────────────┘  │
│       │               │                                          │
└───────┼───────────────┼──────────────────────────────────────────┘
        │               │
        │               │(service_id)
┌───────┼───────────────┼──────────────────────────────────────────┐
│       │  ATTENDANTS   │                                          │
│       │   SERVICE     │                                          │
│       │               │    ┌─────────────────────┐               │
│       │               └───▶│ AttendantService    │               │
│       │                    └──────────┬──────────┘               │
│       │                               │                          │
│       │                    ┌──────────▼──────────┐               │
│       │                    │    Attendant        │               │
│       │                    └─────────────────────┘               │
└───────┼──────────────────────────────────────────────────────────┘
        │
        │(client_id, establishment_id, service_id, attendant_id)
┌───────▼──────────────────────────────────────────────────────────┐
│              APPOINTMENTS SERVICE                                │
│                                                                  │
│              ┌─────────────────────┐                             │
│              │    Appointment      │                             │
│              └─────────────────────┘                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. Auth Service Database

**Banco de dados**: `auth.db`
**Localização**: `auth-service/instance/auth.db`

### 1.1. Tabela: clients

Armazena informações de clientes que utilizam o sistema para agendar serviços.

| Campo       | Tipo         | Constraints                      | Descrição                               |
|-------------|--------------|----------------------------------|-----------------------------------------|
| id          | INTEGER      | PRIMARY KEY, AUTO INCREMENT      | Identificador único do cliente          |
| phone       | VARCHAR(20)  | UNIQUE, NOT NULL, INDEX          | Número de telefone (identificação)      |
| name        | VARCHAR(100) | NOT NULL                         | Nome completo do cliente                |
| email       | VARCHAR(120) | UNIQUE, NULLABLE                 | Email do cliente (opcional)             |
| created_at  | DATETIME     | DEFAULT CURRENT_TIMESTAMP        | Data de criação do registro             |
| last_login  | DATETIME     | NULLABLE                         | Data do último login                    |
| is_active   | BOOLEAN      | DEFAULT TRUE                     | Status ativo/inativo                    |

**Índices**:
- `idx_clients_phone` em `phone`

**Relacionamentos**:
- Um cliente pode ter N agendamentos (appointments-service)

---

### 1.2. Tabela: owners

Armazena informações de proprietários/administradores de estabelecimentos.

| Campo         | Tipo         | Constraints                      | Descrição                               |
|---------------|--------------|----------------------------------|-----------------------------------------|
| id            | INTEGER      | PRIMARY KEY, AUTO INCREMENT      | Identificador único do proprietário     |
| email         | VARCHAR(120) | UNIQUE, NOT NULL, INDEX          | Email (usado para login)                |
| password_hash | VARCHAR(255) | NOT NULL                         | Hash da senha (bcrypt)                  |
| name          | VARCHAR(100) | NOT NULL                         | Nome completo do proprietário           |
| phone         | VARCHAR(20)  | NULLABLE                         | Telefone de contato                     |
| created_at    | DATETIME     | DEFAULT CURRENT_TIMESTAMP        | Data de criação do registro             |
| last_login    | DATETIME     | NULLABLE                         | Data do último login                    |
| is_active     | BOOLEAN      | DEFAULT TRUE                     | Status ativo/inativo                    |
| is_admin      | BOOLEAN      | DEFAULT FALSE                    | Flag de administrador do sistema        |

**Índices**:
- `idx_owners_email` em `email`

**Relacionamentos**:
- Um owner pode ter N estabelecimentos (establishments-service)

**Regras de Negócio**:
- Senha armazenada com hash usando werkzeug.security
- `is_admin=true` permite criar/editar templates de serviços globais

---

## 2. Establishments Service Database

**Banco de dados**: `establishments.db`
**Localização**: `establishments-service/instance/establishments.db`

### 2.1. Tabela: establishments

Armazena informações de estabelecimentos comerciais.

| Campo          | Tipo         | Constraints                      | Descrição                               |
|----------------|--------------|----------------------------------|-----------------------------------------|
| id             | INTEGER      | PRIMARY KEY, AUTO INCREMENT      | Identificador único do estabelecimento  |
| owner_id       | INTEGER      | NOT NULL, INDEX                  | FK lógica para owners (auth-service)    |
| name           | VARCHAR(200) | NOT NULL                         | Nome do estabelecimento                 |
| description    | TEXT         | NULLABLE                         | Descrição detalhada                     |
| business_type  | VARCHAR(50)  | NOT NULL                         | Tipo de negócio                         |
| address        | VARCHAR(300) | NULLABLE                         | Endereço completo                       |
| phone          | VARCHAR(20)  | NULLABLE                         | Telefone de contato                     |
| email          | VARCHAR(120) | NULLABLE                         | Email de contato                        |
| working_hours  | TEXT         | NULLABLE                         | Horários de funcionamento (JSON)        |
| is_active      | BOOLEAN      | DEFAULT TRUE                     | Status ativo/inativo                    |
| created_at     | DATETIME     | DEFAULT CURRENT_TIMESTAMP        | Data de criação do registro             |
| updated_at     | DATETIME     | ON UPDATE CURRENT_TIMESTAMP      | Data da última atualização              |

**Índices**:
- `idx_establishments_owner_id` em `owner_id`

**Valores possíveis para `business_type`**:
- `barbershop` - Barbearia
- `salon` - Salão de Beleza
- `clinic` - Clínica
- `petshop` - Pet Shop
- `gym` - Academia
- `spa` - Spa
- `other` - Outro

**Formato do campo `working_hours`** (JSON string):
```json
{
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
```

**Relacionamentos**:
- Pertence a um owner (auth-service) - validado via API
- Possui N serviços (services-service)
- Possui N atendentes (attendants-service)
- Possui N agendamentos (appointments-service)
- Possui N dias especiais (special_days)

---

### 2.2. Tabela: special_days

Armazena exceções de horário (feriados, manutenção, horários especiais).

| Campo           | Tipo         | Constraints                      | Descrição                               |
|-----------------|--------------|----------------------------------|-----------------------------------------|
| id              | INTEGER      | PRIMARY KEY, AUTO INCREMENT      | Identificador único                     |
| establishment_id| INTEGER      | NOT NULL, INDEX                  | FK para establishments                  |
| date            | DATE         | NOT NULL                         | Data do dia especial                    |
| description     | VARCHAR(200) | NULLABLE                         | Descrição (ex: "Natal", "Manutenção")   |
| is_open         | BOOLEAN      | DEFAULT FALSE                    | Se estará aberto neste dia              |
| opening_time    | VARCHAR(5)   | NULLABLE                         | Horário de abertura (se is_open=true)   |
| closing_time    | VARCHAR(5)   | NULLABLE                         | Horário de fechamento (se is_open=true) |
| created_at      | DATETIME     | DEFAULT CURRENT_TIMESTAMP        | Data de criação do registro             |
| updated_at      | DATETIME     | ON UPDATE CURRENT_TIMESTAMP      | Data da última atualização              |

**Constraints**:
- `UNIQUE(establishment_id, date)` - Um estabelecimento não pode ter duas exceções na mesma data

**Índices**:
- `idx_special_days_establishment_id` em `establishment_id`

**Regras de Negócio**:
- Se `is_open=false`, o estabelecimento está fechado neste dia
- Se `is_open=true`, deve-se informar `opening_time` e `closing_time`
- Sobrepõe os horários de `working_hours` da tabela establishments

---

## 3. Services Service Database

**Banco de dados**: `services.db`
**Localização**: `services-service/instance/services.db`

### 3.1. Tabela: service_templates

Catálogo global de templates de serviços que podem ser reutilizados.

| Campo            | Tipo         | Constraints                      | Descrição                               |
|------------------|--------------|----------------------------------|-----------------------------------------|
| id               | INTEGER      | PRIMARY KEY, AUTO INCREMENT      | Identificador único do template         |
| name             | VARCHAR(100) | NOT NULL                         | Nome do serviço                         |
| description      | TEXT         | NULLABLE                         | Descrição detalhada                     |
| category         | VARCHAR(50)  | NOT NULL                         | Categoria do serviço                    |
| default_duration | INTEGER      | NOT NULL                         | Duração padrão em minutos               |
| default_price    | FLOAT        | NULLABLE                         | Preço sugerido (opcional)               |
| is_active        | BOOLEAN      | DEFAULT TRUE                     | Status ativo/inativo                    |
| created_at       | DATETIME     | DEFAULT CURRENT_TIMESTAMP        | Data de criação                         |
| updated_at       | DATETIME     | ON UPDATE CURRENT_TIMESTAMP      | Data da última atualização              |

**Categorias possíveis**:
- `hair` - Cabelo
- `beard` - Barba
- `nails` - Unhas
- `massage` - Massagem
- `skincare` - Cuidados com a pele
- `other` - Outro

**Regras de Negócio**:
- Templates são compartilhados entre todos os estabelecimentos
- Apenas admins (is_admin=true) podem criar/editar templates
- Estabelecimentos podem criar serviços baseados em templates

---

### 3.2. Tabela: services

Serviços específicos oferecidos por estabelecimentos.

| Campo           | Tipo         | Constraints                      | Descrição                               |
|-----------------|--------------|----------------------------------|-----------------------------------------|
| id              | INTEGER      | PRIMARY KEY, AUTO INCREMENT      | Identificador único do serviço          |
| establishment_id| INTEGER      | NOT NULL, INDEX                  | FK lógica para establishments           |
| template_id     | INTEGER      | NULLABLE                         | FK para service_templates (se baseado)  |
| name            | VARCHAR(100) | NOT NULL                         | Nome do serviço                         |
| description     | TEXT         | NULLABLE                         | Descrição detalhada                     |
| duration        | INTEGER      | NOT NULL                         | Duração em minutos                      |
| price           | FLOAT        | NOT NULL                         | Preço do serviço                        |
| is_active       | BOOLEAN      | DEFAULT TRUE                     | Status ativo/inativo                    |
| created_at      | DATETIME     | DEFAULT CURRENT_TIMESTAMP        | Data de criação                         |
| updated_at      | DATETIME     | ON UPDATE CURRENT_TIMESTAMP      | Data da última atualização              |

**Índices**:
- `idx_services_establishment_id` em `establishment_id`

**Relacionamentos**:
- Pertence a um estabelecimento (establishments-service) - validado via API
- Pode ser baseado em um template (service_templates) - opcional
- Pode ter N atendentes associados (attendants-service)
- Pode ter N agendamentos (appointments-service)

**Regras de Negócio**:
- Se `template_id` for NULL, é um serviço customizado
- Se baseado em template, pode sobrescrever nome, duração e preço
- Apenas o owner do estabelecimento pode criar/editar serviços

---

## 4. Attendants Service Database

**Banco de dados**: `attendants.db`
**Localização**: `attendants-service/instance/attendants.db`

### 4.1. Tabela: attendants

Profissionais que prestam serviços nos estabelecimentos.

| Campo           | Tipo         | Constraints                      | Descrição                               |
|-----------------|--------------|----------------------------------|-----------------------------------------|
| id              | INTEGER      | PRIMARY KEY, AUTO INCREMENT      | Identificador único do atendente        |
| establishment_id| INTEGER      | NOT NULL, INDEX                  | FK lógica para establishments           |
| name            | VARCHAR(100) | NOT NULL                         | Nome completo do atendente              |
| email           | VARCHAR(120) | NULLABLE                         | Email de contato                        |
| phone           | VARCHAR(20)  | NULLABLE                         | Telefone de contato                     |
| specialties     | TEXT         | NULLABLE                         | Descrição das especialidades            |
| is_active       | BOOLEAN      | DEFAULT TRUE                     | Status ativo/inativo                    |
| created_at      | DATETIME     | DEFAULT CURRENT_TIMESTAMP        | Data de criação                         |
| updated_at      | DATETIME     | ON UPDATE CURRENT_TIMESTAMP      | Data da última atualização              |

**Índices**:
- `idx_attendants_establishment_id` em `establishment_id`

**Relacionamentos**:
- Pertence a UM estabelecimento (establishments-service) - validado via API
- Pode prestar N serviços via attendant_services (many-to-many)
- Pode ter N agendamentos (appointments-service)

**Regras de Negócio**:
- Cada atendente trabalha em APENAS UM estabelecimento
- Um atendente pode prestar múltiplos serviços
- Atendentes inativos não aparecem para agendamento

---

### 4.2. Tabela: attendant_services

Tabela de junção para relacionamento many-to-many entre atendentes e serviços.

| Campo        | Tipo     | Constraints                      | Descrição                               |
|--------------|----------|----------------------------------|-----------------------------------------|
| id           | INTEGER  | PRIMARY KEY, AUTO INCREMENT      | Identificador único da associação       |
| attendant_id | INTEGER  | NOT NULL, INDEX                  | FK para attendants                      |
| service_id   | INTEGER  | NOT NULL, INDEX                  | FK lógica para services (services-service)|
| created_at   | DATETIME | DEFAULT CURRENT_TIMESTAMP        | Data da associação                      |

**Constraints**:
- `UNIQUE(attendant_id, service_id)` - Evita duplicação de associação

**Índices**:
- `idx_attendant_services_attendant_id` em `attendant_id`
- `idx_attendant_services_service_id` em `service_id`

**Regras de Negócio**:
- Um atendente só pode ser associado a serviços do mesmo estabelecimento
- A validação é feita via API chamando services-service
- Ao deletar (soft delete) um atendente, todas as associações são removidas

---

## 5. Appointments Service Database

**Banco de dados**: `appointments.db`
**Localização**: `appointments-service/instance/appointments.db`

### 5.1. Tabela: appointments

Agendamentos de serviços feitos por clientes.

| Campo            | Tipo         | Constraints                      | Descrição                               |
|------------------|--------------|----------------------------------|-----------------------------------------|
| id               | INTEGER      | PRIMARY KEY, AUTO INCREMENT      | Identificador único do agendamento      |
| client_id        | INTEGER      | NOT NULL, INDEX                  | FK lógica para clients (auth-service)   |
| establishment_id | INTEGER      | NOT NULL, INDEX                  | FK lógica para establishments           |
| service_id       | INTEGER      | NOT NULL, INDEX                  | FK lógica para services                 |
| attendant_id     | INTEGER      | NULLABLE, INDEX                  | FK lógica para attendants (opcional)    |
| appointment_date | DATE         | NOT NULL, INDEX                  | Data do agendamento                     |
| appointment_time | VARCHAR(5)   | NOT NULL                         | Horário do agendamento (HH:MM)          |
| duration         | INTEGER      | NOT NULL                         | Duração em minutos                      |
| status           | VARCHAR(20)  | NOT NULL, DEFAULT 'pending', INDEX| Status do agendamento                   |
| notes            | TEXT         | NULLABLE                         | Observações adicionais                  |
| created_at       | DATETIME     | DEFAULT CURRENT_TIMESTAMP        | Data de criação do registro             |
| updated_at       | DATETIME     | ON UPDATE CURRENT_TIMESTAMP      | Data da última atualização              |

**Índices**:
- `idx_appointments_client_id` em `client_id`
- `idx_appointments_establishment_id` em `establishment_id`
- `idx_appointments_service_id` em `service_id`
- `idx_appointments_attendant_id` em `attendant_id`
- `idx_appointments_date` em `appointment_date`
- `idx_appointments_status` em `status`

**Valores possíveis para `status`**:
- `pending` - Aguardando confirmação
- `confirmed` - Confirmado
- `cancelled` - Cancelado
- `completed` - Concluído

**Relacionamentos (validados via API)**:
- Pertence a um cliente (auth-service)
- Pertence a um estabelecimento (establishments-service)
- Pertence a um serviço (services-service)
- Opcionalmente pertence a um atendente (attendants-service)

**Regras de Negócio**:
- O serviço deve pertencer ao estabelecimento
- O atendente (se informado) deve pertencer ao estabelecimento
- O atendente (se informado) deve prestar o serviço
- Não pode haver conflito de horário (mesmo atendente, horário sobreposto)
- `duration` é copiado do serviço no momento da criação
- Clientes podem ver/editar apenas seus próprios agendamentos
- Owners podem ver/editar agendamentos dos seus estabelecimentos
- Cancelamentos marcam `status='cancelled'` (não deletam o registro)

---

## Relacionamentos Entre Microserviços

### Relacionamento: Owner → Establishments (1:N)

**Chave**: `owner_id` em `establishments.establishments`

**Validação**:
```python
# establishments-service valida owner via auth-service
response = requests.get(f"{AUTH_SERVICE_URL}/auth/validate",
                       headers={'Authorization': token})
```

---

### Relacionamento: Establishment → Services (1:N)

**Chave**: `establishment_id` em `services.services`

**Validação**:
```python
# services-service valida establishment via establishments-service
response = requests.get(f"{ESTABLISHMENTS_SERVICE_URL}/establishments/{establishment_id}")
```

---

### Relacionamento: Establishment → Attendants (1:N)

**Chave**: `establishment_id` em `attendants.attendants`

**Validação**:
```python
# attendants-service valida establishment via establishments-service
response = requests.get(f"{ESTABLISHMENTS_SERVICE_URL}/establishments/{establishment_id}")
```

---

### Relacionamento: Attendant ↔ Service (N:M)

**Tabela de Junção**: `attendants.attendant_services`

**Validações**:
1. Atendente existe e está ativo
2. Serviço existe (via services-service)
3. Serviço pertence ao mesmo estabelecimento do atendente

```python
# Validar que serviço pertence ao mesmo estabelecimento
attendant = Attendant.query.get(attendant_id)
service_response = requests.get(f"{SERVICES_SERVICE_URL}/services/{service_id}")
service = service_response.json()

if service['establishment_id'] != attendant.establishment_id:
    raise ValidationError("Serviço não pertence ao estabelecimento do atendente")
```

---

### Relacionamento: Client → Appointments (1:N)

**Chave**: `client_id` em `appointments.appointments`

**Validação**:
```python
# appointments-service valida client via auth-service
user = validate_token_with_auth_service(token)
if user['user_type'] != 'client':
    raise AuthorizationError()
```

---

### Relacionamento: Appointment → (Establishment, Service, Attendant)

**Chaves**:
- `establishment_id` → establishments-service
- `service_id` → services-service
- `attendant_id` → attendants-service

**Validações em cascata**:
1. Estabelecimento existe e está ativo
2. Serviço existe e pertence ao estabelecimento
3. Atendente (se informado) existe, está ativo e pertence ao estabelecimento
4. Atendente (se informado) presta o serviço especificado
5. Não há conflito de horário (outro agendamento no mesmo horário para o mesmo atendente)

---

## Estratégias de Integridade Referencial

### 1. Validação Síncrona via API

Antes de criar/atualizar registros, os serviços validam FKs lógicas via chamadas HTTP:

```python
def validate_establishment_ownership(establishment_id, owner_id):
    response = requests.get(
        f"{ESTABLISHMENTS_SERVICE_URL}/establishments/{establishment_id}"
    )
    if response.status_code == 200:
        establishment = response.json()
        if establishment['owner_id'] == owner_id:
            return establishment
    raise ValidationError("Estabelecimento não encontrado ou sem permissão")
```

### 2. Soft Deletes

Uso de flag `is_active` para desativar registros ao invés de deletá-los:

```python
# Ao "deletar" um estabelecimento
establishment.is_active = False
db.session.commit()

# Queries filtram por is_active
active_establishments = Establishment.query.filter_by(is_active=True).all()
```

### 3. Detecção de Conflitos

No appointments-service, antes de criar/atualizar um agendamento:

```python
def check_time_slot_availability(establishment_id, attendant_id,
                                 appointment_date, appointment_time,
                                 duration, exclude_appointment_id=None):
    # Busca agendamentos existentes do atendente na mesma data
    existing = Appointment.query.filter(
        Appointment.establishment_id == establishment_id,
        Appointment.attendant_id == attendant_id,
        Appointment.appointment_date == appointment_date,
        Appointment.status.in_(['pending', 'confirmed'])
    ).all()

    # Verifica sobreposição de horários
    for apt in existing:
        if apt.id == exclude_appointment_id:
            continue
        if time_ranges_overlap(apt.appointment_time, apt.duration,
                              appointment_time, duration):
            return False, "Horário não disponível"

    return True, None
```

---

## Estratégias de Migração para Produção

### Migração SQLite → PostgreSQL

1. **Adaptar tipos de dados**:
   - `INTEGER` → `SERIAL` (para AUTO INCREMENT)
   - `TEXT` → `TEXT` ou `VARCHAR`
   - `DATETIME` → `TIMESTAMP`

2. **Adicionar constraints**:
   ```sql
   ALTER TABLE establishments
   ADD CONSTRAINT fk_owner
   FOREIGN KEY (owner_id) REFERENCES owners(id);
   ```

3. **Configurar conexão**:
   ```python
   # config.py
   SQLALCHEMY_DATABASE_URI = os.getenv(
       'DATABASE_URL',
       'postgresql://user:pass@localhost/marcarhora'
   )
   ```

### Escalabilidade

1. **Sharding por estabelecimento**: Particionar dados de agendamentos por `establishment_id`
2. **Read Replicas**: Para queries de busca e relatórios
3. **Caching**: Redis para dados frequentemente acessados (templates, estabelecimentos)
4. **Message Queue**: RabbitMQ/Kafka para comunicação assíncrona entre serviços

---

## Índices e Performance

### Índices Críticos por Tabela

**clients**:
- `phone` (UNIQUE) - Login de clientes

**owners**:
- `email` (UNIQUE) - Login de owners

**establishments**:
- `owner_id` - Listar estabelecimentos por owner

**services**:
- `establishment_id` - Listar serviços por estabelecimento

**attendants**:
- `establishment_id` - Listar atendentes por estabelecimento

**attendant_services**:
- `attendant_id` - Listar serviços de um atendente
- `service_id` - Listar atendentes que prestam um serviço
- `UNIQUE(attendant_id, service_id)` - Evitar duplicatas

**appointments**:
- `client_id` - Listar agendamentos de um cliente
- `establishment_id` - Listar agendamentos de um estabelecimento
- `appointment_date` - Filtrar por data
- `status` - Filtrar por status
- `attendant_id` - Verificar disponibilidade de atendente

### Queries Otimizadas

```sql
-- Buscar disponibilidade de atendente em uma data
CREATE INDEX idx_appointments_availability
ON appointments(attendant_id, appointment_date, status);

-- Buscar agendamentos de cliente por data
CREATE INDEX idx_appointments_client_date
ON appointments(client_id, appointment_date);

-- Buscar estabelecimentos por tipo e status
CREATE INDEX idx_establishments_type_active
ON establishments(business_type, is_active);
```

---

## Backup e Recuperação

### Estratégia de Backup (Produção)

1. **Backup Full Diário**: Dump completo de cada banco às 02:00 AM
2. **Backup Incremental**: WAL (Write-Ahead Logging) contínuo
3. **Retenção**: 30 dias de backups diários, 12 meses de backups mensais
4. **Testes de Restore**: Mensais em ambiente de staging

### Comandos de Backup (PostgreSQL)

```bash
# Backup de um serviço
pg_dump -h localhost -U user -d marcarhora_auth > auth_backup.sql

# Restore
psql -h localhost -U user -d marcarhora_auth < auth_backup.sql
```

---

## Auditoria e Logs

### Campos de Auditoria

Todas as tabelas principais possuem:
- `created_at` - Timestamp de criação
- `updated_at` - Timestamp da última atualização

### Logs de Alteração (Futuro)

Implementar tabelas de auditoria:

```sql
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(50) NOT NULL,
    record_id INTEGER NOT NULL,
    action VARCHAR(20) NOT NULL, -- INSERT, UPDATE, DELETE
    user_id INTEGER NOT NULL,
    user_type VARCHAR(20) NOT NULL,
    changes JSONB,
    timestamp TIMESTAMP DEFAULT NOW()
);
```

---

## Considerações de Segurança

### 1. Senhas

- Armazenadas com hash bcrypt (werkzeug.security)
- Nunca retornadas em APIs
- Validação de força de senha (futuro)

### 2. Dados Sensíveis

- Telefones e emails não devem ser expostos publicamente
- Apenas owners podem ver dados completos dos seus estabelecimentos
- Clientes veem apenas seus próprios dados

### 3. Soft Deletes

- Registros marcados como inativos ao invés de deletados
- Permite auditoria e recuperação de dados
- Filtros automáticos em queries para excluir inativos

---

## Documentação Gerada

**Data**: 2025-12-27
**Versão do Schema**: 1.0.0
**Status**: Schemas de todos os 5 microserviços documentados
**Ambiente**: Desenvolvimento (SQLite) → Migração planejada para PostgreSQL em produção
