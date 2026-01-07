# 📘 Documentação Técnica Completa - Sistema CRM

**Versão do Sistema:** 1.6.0  
**Data da Análise:** 2025-01-07  
**Framework Base:** Laravel 11.9 + Vue.js 3.4.3

---

## 📋 Índice

1. [Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
2. [Stack Tecnológico](#2-stack-tecnológico)
3. [Arquitetura Modular](#3-arquitetura-modular)
4. [Sistema de Autenticação e Autorização](#4-sistema-de-autenticação-e-autorização)
5. [Sistema de Broadcast e WebSockets](#5-sistema-de-broadcast-e-websockets)
6. [Sistema de Filas e Jobs](#6-sistema-de-filas-e-jobs)
7. [Estrutura de Banco de Dados](#7-estrutura-de-banco-de-dados)
8. [Frontend - Vue.js](#8-frontend---vuejs)
9. [Sistema de WhatsApp](#9-sistema-de-whatsapp)
10. [Sistema de Teams e Permissões](#10-sistema-de-teams-e-permissões)
11. [Integrações Externas](#11-integrações-externas)
12. [Sistema de Workflows](#12-sistema-de-workflows)
13. [Sistema de Mídia e Armazenamento](#13-sistema-de-mídia-e-armazenamento)
14. [Configurações e Ambiente](#14-configurações-e-ambiente)

---

## 1. Visão Geral da Arquitetura

### 1.1 Padrão Arquitetural

O sistema utiliza uma **arquitetura modular baseada em Laravel Modules**, seguindo o padrão **MVC (Model-View-Controller)** com separação clara de responsabilidades:

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (Vue.js 3)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Components  │  │  Composables  │  │    Store     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                        ↕ HTTP/REST API
┌─────────────────────────────────────────────────────────┐
│              Backend (Laravel 11.9)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Controllers  │  │   Services   │  │    Models    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │    Events    │  │     Jobs     │  │  Observers   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                        ↕
┌─────────────────────────────────────────────────────────┐
│              Infraestrutura                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Database   │  │     Redis     │  │   Storage    │  │
│  │   (MySQL)    │  │  (Queues/     │  │   (Laravel   │  │
│  │              │  │   Cache)      │  │   Storage)   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  ┌──────────────┐  ┌──────────────┐                    │
│  │   Reverb     │  │   External    │                    │
│  │  (WebSocket) │  │   Services    │                    │
│  └──────────────┘  └──────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Princípios de Design

- **Modularidade**: Sistema dividido em módulos independentes
- **Separação de Responsabilidades**: Cada camada tem função específica
- **Inversão de Dependência**: Uso de interfaces e injeção de dependência
- **Event-Driven**: Sistema de eventos para comunicação assíncrona
- **Queue-Based Processing**: Processamento assíncrono via filas Redis

---

## 2. Stack Tecnológico

### 2.1 Backend

| Tecnologia | Versão | Uso |
|------------|--------|-----|
| **PHP** | ^8.2 | Linguagem principal |
| **Laravel Framework** | ^11.9 | Framework base |
| **Laravel Reverb** | ^1.5 | WebSocket server |
| **Laravel Sanctum** | ^4.0 | Autenticação API |
| **Spatie Laravel Permission** | ^6.4 | Sistema de permissões |
| **Laravel Model Caching** | ^11.0 | Cache de modelos |
| **Predis** | ^2.0 | Cliente Redis |
| **Guzzle HTTP** | ^7.2 | Cliente HTTP |
| **AWS SDK PHP** | ^3.268 | Integração AWS (S3, SES, SQS) |
| **Twilio SDK** | ^8.2 | Integração VoIP/WhatsApp |
| **Microsoft Graph** | ^1.94 | Integração Office 365 |
| **Google API Client** | ^2.15.0 | Integração Google |

### 2.2 Frontend

| Tecnologia | Versão | Uso |
|------------|--------|-----|
| **Vue.js** | ^3.4.3 | Framework frontend |
| **Vue Router** | ^4.2.2 | Roteamento |
| **Pinia** | ^2.1.0 | Gerenciamento de estado |
| **Vuex** | ^4.0.2 | Store (legado) |
| **Axios** | ^1.7.8 | Cliente HTTP |
| **Laravel Echo** | ^2.2.0 | WebSocket client |
| **Pusher JS** | ^8.4.0 | WebSocket (fallback) |
| **Tailwind CSS** | ^3.2.7 | Framework CSS |
| **Headless UI** | ^1.7.14 | Componentes UI |
| **Vite** | ^5.3.3 | Build tool |
| **Luxon** | ^3.4.4 | Manipulação de datas |
| **Chartist** | ^1.3.0 | Gráficos |

### 2.3 Banco de Dados

- **MySQL/MariaDB**: Banco principal (utf8mb4_unicode_ci)
- **Redis**: Cache, sessões, filas e broadcast
- **PostgreSQL**: Suportado (configurável)

### 2.4 Infraestrutura

- **Laravel Reverb**: WebSocket server nativo
- **Redis**: Sistema de filas e cache
- **Laravel Storage**: Sistema de arquivos (local/S3)
- **Supervisor**: Gerenciamento de workers de fila

---

## 3. Arquitetura Modular

### 3.1 Estrutura de Módulos

O sistema utiliza **Laravel Modules (nwidart/laravel-modules ^11.0)** para organização modular:

```
modules/
├── Core/              # Módulo central - funcionalidades base
├── Users/             # Gerenciamento de usuários e teams
├── Whatsapp/          # Integração WhatsApp
├── Deals/             # Gestão de negócios
├── Contacts/          # Gestão de contatos
├── Activities/        # Atividades e calendário
├── Calls/             # Chamadas VoIP (Twilio/Sempre)
├── Documents/         # Gestão de documentos
├── MailClient/        # Cliente de e-mail
├── Workflows/         # Sistema de workflows (n8n-like)
├── Billable/          # Faturamento e produtos
├── Payment/           # Pagamentos
├── Notes/             # Notas
├── Comments/          # Comentários
├── Brands/            # Marcas
├── WebForms/          # Formulários web
├── Auth/              # Autenticação
├── Translator/        # Tradução
├── ThemeStyle/        # Temas e estilos
├── Updater/           # Sistema de atualização
└── Installer/         # Instalador
```

### 3.2 Estrutura Padrão de um Módulo

Cada módulo segue a estrutura:

```
ModuleName/
├── app/
│   ├── Controllers/       # Controladores
│   ├── Models/            # Modelos Eloquent
│   ├── Services/          # Lógica de negócio
│   ├── Events/            # Eventos
│   ├── Listeners/         # Listeners de eventos
│   ├── Jobs/              # Jobs de fila
│   ├── Observers/        # Observers de modelos
│   ├── Policies/          # Políticas de autorização
│   ├── Providers/         # Service Providers
│   └── Concerns/          # Traits
├── database/
│   ├── migrations/        # Migrações
│   └── seeders/          # Seeders
├── resources/
│   ├── js/                # JavaScript/Vue
│   │   ├── components/   # Componentes Vue
│   │   ├── composables/  # Composables
│   │   ├── views/         # Views
│   │   └── app.js         # Entry point
│   └── lang/              # Traduções
├── routes/
│   ├── api.php            # Rotas API
│   ├── web.php            # Rotas web
│   └── channels.php       # Canais broadcast
├── config/                # Configurações
├── tests/                 # Testes
└── module.json            # Metadados do módulo
```

### 3.3 Módulo Core

O módulo **Core** é o núcleo do sistema e fornece:

- **Resource System**: Sistema de recursos CRUD genérico
- **Field System**: Sistema de campos dinâmicos
- **Filter System**: Sistema de filtros
- **Table System**: Sistema de tabelas
- **Card System**: Sistema de cards para dashboard
- **Chart System**: Sistema de gráficos
- **Permission System**: Sistema base de permissões
- **Settings System**: Sistema de configurações
- **Broadcast System**: Sistema de broadcast
- **HTTP Service**: Cliente HTTP centralizado
- **Gate System**: Sistema de autorização frontend

---

## 4. Sistema de Autenticação e Autorização

### 4.1 Autenticação

**Laravel Sanctum** é usado para autenticação de API:

- **Token-based**: Tokens de API para requisições
- **Session-based**: Sessões para aplicação web
- **CSRF Protection**: Proteção CSRF ativa

**Configuração:**
- Driver: `sanctum`
- Guard: `web` (padrão)
- Token expiration: Configurável

### 4.2 Autorização - Sistema de Permissões

O sistema utiliza **Spatie Laravel Permission** com extensões customizadas:

#### 4.2.1 Hierarquia de Permissões

```
Super Admin
    ↓ (acesso total)
Roles (Funções)
    ↓ (permissões específicas)
Users (Usuários)
    ↓ (herda permissões da role)
Resources (Recursos)
    ↓ (aplicação de permissões)
Actions (Ações: view, edit, delete)
```

#### 4.2.2 Tipos de Permissões

1. **Super Admin**: Acesso total ao sistema
   - Verificado via `user->super_admin` ou `user->hasRole('super-admin')`
   - Bypass em todas as verificações de permissão

2. **Role-based Permissions**: Permissões por função
   - Cada role tem um conjunto de permissões
   - Usuários herdam permissões de suas roles

3. **Resource-based Permissions**: Permissões por recurso
   - `view own {resource}`: Ver apenas próprios registros
   - `view all {resource}`: Ver todos os registros
   - `view team {resource}`: Ver registros do team
   - `edit own {resource}`: Editar apenas próprios registros
   - `edit all {resource}`: Editar todos os registros
   - `edit team {resource}`: Editar registros do team
   - `delete own {resource}`: Deletar apenas próprios registros
   - `delete all {resource}`: Deletar todos os registros

4. **Policy-based Authorization**: Políticas customizadas
   - Cada modelo pode ter uma Policy
   - Métodos: `view`, `create`, `update`, `delete`, etc.
   - Acessado via `Gate::allows()` ou `$user->can()`

#### 4.2.3 Verificação de Permissões

**Backend (PHP):**
```php
// Verificar se usuário tem permissão
$user->can('view', $deal);

// Verificar role
$user->hasRole('super-admin');

// Verificar permissão específica
$user->hasPermissionTo('view all deals');
```

**Frontend (JavaScript):**
```javascript
// Verificar permissão
gate.userCan('view all deals');

// Verificar autorização em registro
gate.allows('edit', deal);

// Verificar se é super admin
gate.isSuperAdmin();
```

### 4.3 Sistema de Teams

O sistema implementa um modelo de **Teams (Equipes)** para controle de acesso:

#### 4.3.1 Estrutura de Teams

```
Team
├── user_id (gerente)
├── name
├── description
└── Relações:
    ├── users (membros via pivot team_user)
    └── whatsappDevices (dispositivos via pivot team_whatsapp_devices)
```

#### 4.3.2 Relações de Usuário com Teams

1. **Membro de Team**: `$user->teams()` - via tabela pivot `team_user`
2. **Gerente de Team**: `$user->managedTeams()` - via `team.user_id`
3. **Todos os Teams**: `$user->allTeams()` - combina membro + gerente

#### 4.3.3 Métodos Principais

```php
// Verificar se usuário está no mesmo team
$user->isInSameTeamWith($otherUser);

// Verificar se usuário pertence ao team
$user->belongsToTeam($team);

// Obter IDs de dispositivos acessíveis
$user->getAccessibleDeviceIds();
```

---

## 5. Sistema de Broadcast e WebSockets

### 5.1 Arquitetura de Broadcast

O sistema utiliza **Laravel Reverb** como servidor WebSocket nativo:

```
┌─────────────┐
│   Client    │
│  (Vue.js)   │
└──────┬──────┘
       │ WebSocket
       ↓
┌─────────────┐      ┌─────────────┐
│   Reverb    │ ←──→ │    Redis     │
│   Server    │      │  (Pub/Sub)  │
└──────┬──────┘      └─────────────┘
       │
       ↓
┌─────────────┐
│   Laravel   │
│  Backend    │
└─────────────┘
```

### 5.2 Configuração

**Backend (`config/broadcasting.php`):**
- Driver padrão: `reverb`
- Conexões suportadas: `reverb`, `pusher`, `redis`, `log`, `null`

**Frontend (`resources/js/app.js`):**
- Cliente: Laravel Echo + Pusher JS
- Configuração via variáveis de ambiente:
  - `VITE_REVERB_APP_KEY`
  - `VITE_REVERB_HOST`
  - `VITE_REVERB_PORT`
  - `VITE_REVERB_SCHEME`

### 5.3 Canais de Broadcast

#### 5.3.1 Canais Públicos

- `whatsapp.messages`: Mensagens gerais
- `whatsapp.queue`: Fila de conversas

#### 5.3.2 Canais Privados (Autorizados)

- `whatsapp.conversation.{conversationId}`: Canal específico de conversa
- `whatsapp.deal.{dealId}`: Canal específico de deal
- `whatsapp.device.{deviceId}`: Canal específico de dispositivo
- `whatsapp.user.{userId}`: Canal de notificações do usuário
- `App.Models.User.{id}`: Canal genérico de usuário

### 5.4 Eventos de Broadcast

#### 5.4.1 Eventos WhatsApp

**WhatsappMessageReceived:**
- Disparado quando nova mensagem é recebida
- Implementa `ShouldBroadcastNow` (broadcast imediato)
- Envia para todos os usuários com acesso à conversa
- Canais: `whatsapp.user.{userId}` para cada usuário autorizado

**WhatsappMessageUpdated:**
- Disparado quando mensagem é atualizada
- Mesmo comportamento do `WhatsappMessageReceived`

#### 5.4.2 Autorização de Canais

A autorização é feita em `routes/channels.php` e `modules/Whatsapp/routes/channels.php`:

```php
Broadcast::channel('whatsapp.conversation.{conversationId}', function ($user, $conversationId) {
    $deal = Deal::find($conversationId);
    
    // 1. Usuário atribuído
    if ($deal->user_id === $user->id) return true;
    
    // 2. Super admin
    if ($user->hasRole('super-admin')) return true;
    
    // 3. Mesmo team do usuário atribuído
    if ($user->isInSameTeamWith($assignedUser)) return true;
    
    // 4. Acesso via team ao dispositivo
    if (in_array($deal->device_id, $user->getAccessibleDeviceIds())) return true;
    
    // 5. Permissão específica
    if ($user->can('view', $deal)) return true;
    
    return false;
});
```

### 5.5 Frontend - Uso de Broadcast

```javascript
// Inicializar broadcast
import { useBroadcast } from '@/Core/composables/useBroadcast'

const { subscribe, unsubscribe } = useBroadcast()

// Inscrever em canal privado
subscribe('whatsapp.conversation.123', (event, data) => {
    console.log('Nova mensagem:', data)
})

// Inscrever em canal de usuário
subscribe('whatsapp.user.1', (event, data) => {
    console.log('Notificação:', data)
})
```

---

## 6. Sistema de Filas e Jobs

### 6.1 Configuração de Filas

O sistema utiliza **Redis** como driver de filas:

**Configuração (`config/queue.php`):**
- Connection: `redis`
- Queue: Múltiplas filas especializadas

### 6.2 Filas Especializadas

| Fila | Uso | Workers |
|------|-----|--------|
| `default` | Jobs gerais | 1 |
| `whatsapp-webhooks` | Webhooks do WhatsApp | 1+ |
| `whatsapp-media` | Processamento de mídia | 1+ |
| `whatsapp-notifications` | Notificações WhatsApp | 1+ |
| `workflow-execution` | Execução de workflows | 1+ |
| `webhook-processing` | Processamento de webhooks | 1+ |
| `workflow-schedule` | Agendamento de workflows | 1+ |
| `email-sending` | Envio de e-mails | 1+ |

### 6.3 Jobs Principais

#### 6.3.1 WhatsApp

**ProcessWhatsappWebhook:**
- Processa webhooks recebidos do WhatsApp
- Fila: `whatsapp-webhooks`
- Tries: 3
- Timeout: 300s

**ProcessMessageChunk:**
- Processa lotes de mensagens
- Fila: `whatsapp-webhooks`
- Tries: 3
- Timeout: 300s

**ProcessMessageMediaBatch:**
- Processa mídias em lote
- Fila: `whatsapp-media`
- Tries: 3

**ProcessWhatsappNotifications:**
- Processa notificações
- Fila: `whatsapp-notifications`
- Tries: 2
- Timeout: 30s

#### 6.3.2 Workflows

**ExecuteWorkflowJob:**
- Executa workflow
- Fila: `workflow-execution`
- Retry com backoff exponencial

**ProcessWebhookJob:**
- Processa webhook de workflow
- Fila: `webhook-processing`

**ScheduleWorkflowJob:**
- Agenda execução de workflow
- Fila: `workflow-schedule`

### 6.4 Processamento de Filas

**Comando Supervisor:**
```bash
php artisan queue:work --queue=whatsapp-webhooks,whatsapp-media,whatsapp-notifications --tries=3
```

**Script de Desenvolvimento (`composer.json`):**
```json
"dev": [
    "php artisan serve",
    "php artisan queue:work --verbose",
    "php artisan queue:work --queue=whatsapp-webhooks,whatsapp-media,whatsapp-notifications --name=whatsapp-worker-1 --tries=3",
    "php artisan reverb:start"
]
```

### 6.5 Monitoramento de Filas

O sistema implementa monitoramento de filas com thresholds:

```php
$queueThresholds = [
    'workflow-execution' => ['warning' => 100, 'critical' => 500],
    'webhook-processing' => ['warning' => 50, 'critical' => 200],
    'workflow-schedule' => ['warning' => 20, 'critical' => 100],
    'email-sending' => ['warning' => 50, 'critical' => 200],
];
```

---

## 7. Estrutura de Banco de Dados

### 7.1 Configuração

- **Driver**: MySQL/MariaDB (padrão)
- **Charset**: `utf8mb4`
- **Collation**: `utf8mb4_unicode_ci`
- **Prefix**: `tbl_` (configurável via `DB_PREFIX`)
- **Engine**: InnoDB

### 7.2 Tabelas Principais

#### 7.2.1 Core

| Tabela | Descrição |
|--------|-----------|
| `users` | Usuários do sistema |
| `roles` | Funções/roles |
| `permissions` | Permissões |
| `model_has_roles` | Relação usuário-role |
| `role_has_permissions` | Relação role-permissão |
| `settings` | Configurações do sistema |
| `media` | Arquivos de mídia |
| `mediables` | Relação polimórfica de mídia |
| `meta` | Metadados (sistema metable) |
| `activity_log` | Log de atividades |

#### 7.2.2 Teams

| Tabela | Descrição |
|--------|-----------|
| `teams` | Teams/equipes |
| `team_user` | Relação usuário-team (pivot) |
| `team_whatsapp_devices` | Relação team-dispositivo (pivot) |

#### 7.2.3 WhatsApp

| Tabela | Descrição |
|--------|-----------|
| `whatsapp_devices` | Dispositivos WhatsApp |
| `whatsapp_messages` | Mensagens WhatsApp |
| `whatsapp_media_files` | Arquivos de mídia do WhatsApp |
| `whatsapp_conversation_queue` | Fila de conversas |
| `user_whatsapp_devices` | Relação usuário-dispositivo (pivot) |

#### 7.2.4 CRM

| Tabela | Descrição |
|--------|-----------|
| `deals` | Negócios/Oportunidades |
| `contacts` | Contatos |
| `companies` | Empresas |
| `activities` | Atividades |
| `calls` | Chamadas |
| `documents` | Documentos |
| `notes` | Notas |
| `comments` | Comentários |

#### 7.2.5 Workflows

| Tabela | Descrição |
|--------|-----------|
| `workflow_flows` | Workflows |
| `workflow_flow_executions` | Execuções de workflows |
| `workflow_execution_logs` | Logs de execução |

### 7.3 Relacionamentos Principais

#### 7.3.1 Users → Teams

```sql
users (1) ──< team_user >── (N) teams
users (1) ──< teams.user_id >── (N) teams (gerente)
```

#### 7.3.2 Users → WhatsApp Devices

```sql
users (1) ──< user_whatsapp_devices >── (N) whatsapp_devices
users (1) ──< users.whatsapp_device_id >── (1) whatsapp_devices (direto)
teams (1) ──< team_whatsapp_devices >── (N) whatsapp_devices
```

#### 7.3.3 Deals → WhatsApp

```sql
deals (1) ──< deals.chat_id >── (1) whatsapp_messages (via chat_id)
deals (1) ──< deals.device_id >── (1) whatsapp_devices
deals (1) ──< deals.user_id >── (1) users (atribuído)
```

### 7.4 Soft Deletes

Muitas tabelas utilizam **soft deletes**:
- `deleted_at` timestamp nullable
- Recuperação de registros deletados
- Exclusão lógica ao invés de física

---

## 8. Frontend - Vue.js

### 8.1 Arquitetura Frontend

```
resources/js/
├── app.js                    # Entry point
├── components/               # Componentes globais
├── composables/              # Composables Vue 3
├── services/                 # Serviços (HTTP, Broadcast)
├── store/                    # Pinia/Vuex stores
├── router/                   # Vue Router
└── utils/                    # Utilitários
```

### 8.2 Sistema de Módulos Frontend

Cada módulo tem seu próprio entry point:

```javascript
// modules/Whatsapp/resources/js/app.js
import { createApp } from 'vue'
import WhatsappComponents from './components'

export default function (app, router, store) {
    // Registrar componentes
    // Registrar rotas
    // Registrar stores
}
```

### 8.3 Composables Principais

**useBroadcast:**
- Gerencia inscrições em canais WebSocket
- Auto-reconexão
- Cleanup automático

**useResource:**
- CRUD genérico de recursos
- Paginação
- Filtros e busca

**useGate:**
- Verificação de permissões frontend
- Sincronizado com backend

**useTable:**
- Gerenciamento de tabelas
- Ordenação
- Seleção múltipla

**useForm:**
- Gerenciamento de formulários
- Validação
- Submissão

### 8.4 Sistema de Componentes

**Componentes Base:**
- `BaseFormField.vue`: Campo de formulário base
- `BaseDetailField.vue`: Campo de detalhe base
- `BaseIndexField.vue`: Campo de listagem base
- `BaseSelectField.vue`: Campo de seleção base

**Componentes de UI:**
- Headless UI components
- Tailwind CSS styling
- Responsive design

### 8.5 Build e Desenvolvimento

**Vite Configuration:**
- Entry: `resources/js/app.js`
- Output: `public/build`
- HMR: Hot Module Replacement
- Aliases: `@/ModuleName` → `modules/ModuleName/resources/js`

**Comandos:**
```bash
npm run dev      # Desenvolvimento com HMR
npm run build    # Build de produção
npm run watch    # Watch mode
```

---

## 9. Sistema de WhatsApp

### 9.1 Arquitetura

```
WhatsApp API/Webhook
    ↓
WhatsappWebhookController
    ↓
ProcessWhatsappWebhook (Job)
    ↓
WhatsappMessageService
    ↓
WhatsappMessage Model
    ↓
WhatsappMessageReceived (Event)
    ↓
Broadcast → Frontend
```

### 9.2 Componentes Principais

#### 9.2.1 Models

**WhatsappDevice:**
- Representa um dispositivo WhatsApp conectado
- Campos: `name`, `phone_number`, `status`, `api_key`
- Relações: `users`, `teams`, `messages`

**WhatsappMessage:**
- Representa uma mensagem
- Campos: `message_id`, `chat_id`, `from`, `to`, `body`, `timestamp`, `type`
- Relações: `deal`, `contact`, `user`, `mediaFiles`

**WhatsappMediaFile:**
- Representa arquivo de mídia
- Campos: `message_id`, `media_id`, `mime_type`, `file_path`, `file_size`
- Relação: `message`

#### 9.2.2 Services

**WhatsappConversationQueueService:**
- Gerencia fila de conversas
- Atribuição automática
- Timeout de atendimento

**WhatsappBroadcastingService:**
- Publica eventos de broadcast
- Determina usuários com acesso
- Envia para canais apropriados

**WhatsappNotificationService:**
- Envia notificações
- Diferentes tipos de notificação
- Integração com sistema de notificações

#### 9.2.3 Controllers

**WhatsappWebhookController:**
- Recebe webhooks do WhatsApp
- Validação de assinatura
- Dispatch de jobs

**WhatsappAssignmentController:**
- Gerencia atribuição de conversas
- Lista conversas disponíveis
- Filtros de acesso baseados em teams

### 9.3 Fluxo de Mensagem Recebida

```
1. Webhook recebido
   ↓
2. WhatsappWebhookController::handle()
   ↓
3. Validação de assinatura
   ↓
4. ProcessWhatsappWebhook (Job) → Fila
   ↓
5. Processamento assíncrono:
   - Criar/atualizar mensagem
   - Processar mídia (se houver)
   - Vincular a deal/contact
   ↓
6. WhatsappMessageReceived (Event)
   ↓
7. Broadcast para usuários autorizados
   ↓
8. Frontend recebe via WebSocket
```

### 9.4 Regras de Acesso a Conversas

Ver seção [10. Sistema de Teams e Permissões](#10-sistema-de-teams-e-permissões)

---

## 10. Sistema de Teams e Permissões

### 10.1 Hierarquia de Acesso a Conversas

A verificação de acesso segue esta ordem:

1. **Usuário Atribuído**: `deal->user_id === user->id`
2. **Super Admin**: `user->hasRole('super-admin')`
3. **Mesmo Team**: `user->isInSameTeamWith($assignedUser)`
4. **Acesso via Team ao Dispositivo**: `device_id` em `user->getAccessibleDeviceIds()`
5. **Permissão Específica**: `user->can('view', $deal)`

### 10.2 Pontos de Verificação

#### 10.2.1 Backend - Listagem de Conversas

**Arquivo:** `WhatsappAssignmentController::getMyConversations()`

```php
if ($user->isSuperAdmin()) {
    // Sem filtro - todas as conversas
} else {
    // Buscar usuários do mesmo team
    $teamUserIds = User::whereHas('teams', function($q) use ($user) {
        $q->whereIn('teams.id', $user->allTeams()->pluck('id'));
    })->orWhereHas('managedTeams', function($q) use ($user) {
        $q->whereIn('teams.id', $user->allTeams()->pluck('id'));
    })->pluck('id');
    
    // Incluir próprio usuário
    $teamUserIds->push($user->id);
    
    // Filtrar deals
    $query->whereIn('user_id', $teamUserIds);
}
```

#### 10.2.2 Broadcast - Eventos de Mensagem

**Arquivo:** `WhatsappMessageReceived::getUsersWithAccessToConversation()`

```php
private function getUsersWithAccessToConversation(): array
{
    $userIds = [];
    
    // 1. Usuário atribuído + usuários do mesmo team
    if ($deal->user_id) {
        $userIds[] = $deal->user_id;
        $assignedUser = User::find($deal->user_id);
        $teamUserIds = User::whereHas('teams', ...)
            ->orWhereHas('managedTeams', ...)
            ->pluck('id');
        $userIds = array_merge($userIds, $teamUserIds);
    }
    
    // 2. Super admins
    $superAdmins = User::whereHas('roles', ...)->pluck('id');
    $userIds = array_merge($userIds, $superAdmins);
    
    // 3. Usuários com acesso via team ao dispositivo
    if ($deal->device_id) {
        $deviceTeamIds = Team::whereHas('whatsappDevices', ...)->pluck('id');
        $deviceUserIds = User::whereHas('teams', function($q) use ($deviceTeamIds) {
            $q->whereIn('teams.id', $deviceTeamIds);
        })->pluck('id');
        $userIds = array_merge($userIds, $deviceUserIds);
    }
    
    return array_unique($userIds);
}
```

#### 10.2.3 Channels - Autorização de Canais

**Arquivo:** `modules/Whatsapp/routes/channels.php`

```php
Broadcast::channel('whatsapp.conversation.{conversationId}', function ($user, $conversationId) {
    $deal = Deal::find($conversationId);
    if (!$deal) return false;
    
    // 1. Usuário atribuído
    if ($deal->user_id === $user->id) return true;
    
    // 2. Super admin
    if ($user->hasRole('super-admin')) return true;
    
    // 3. Mesmo team do usuário atribuído
    if ($deal->user_id) {
        $assignedUser = User::find($deal->user_id);
        if ($user->isInSameTeamWith($assignedUser)) return true;
    }
    
    // 4. Acesso via team ao dispositivo
    if (in_array($deal->device_id, $user->getAccessibleDeviceIds())) return true;
    
    // 5. Permissão específica
    if ($user->can('view', $deal)) return true;
    
    return false;
});
```

### 10.3 Métodos de Acesso a Dispositivos

```php
public function getAccessibleDeviceIds(): array
{
    $deviceIds = [];
    
    // 1. Dispositivo vinculado diretamente
    if ($this->whatsapp_device_id) {
        $deviceIds[] = $this->whatsapp_device_id;
    }
    
    // 2. Dispositivos via tabela pivot
    $deviceIds = array_merge($deviceIds, $this->whatsappDevices->pluck('id')->toArray());
    
    // 3. Dispositivos via teams
    $teamDeviceIds = $this->allTeams()
        ->flatMap(fn($team) => $team->whatsappDevices->pluck('id'))
        ->unique()
        ->toArray();
    
    $deviceIds = array_merge($deviceIds, $teamDeviceIds);
    
    return array_unique($deviceIds);
}
```

---

## 11. Integrações Externas

### 11.1 Integrações Disponíveis

| Integração | Uso | Status |
|------------|-----|--------|
| **Google** | Calendar, Gmail, OAuth | ✅ Ativo |
| **Microsoft** | Office 365, Outlook, Calendar | ✅ Ativo |
| **Twilio** | VoIP, WhatsApp Business API | ✅ Ativo |
| **Sempre** | Telefonia | ✅ Ativo |
| **AWS** | S3, SES, SQS, DynamoDB | ✅ Ativo |
| **Zapier** | Automações | ✅ Ativo |
| **OpenAI** | IA, Geração de conteúdo | ✅ Ativo |
| **Pusher** | WebSocket (fallback) | ⚠️ Deprecado (usar Reverb) |

### 11.2 Google Integration

**Serviços:**
- Google Calendar: Sincronização de eventos
- Gmail: Integração de e-mail
- OAuth 2.0: Autenticação

**Configuração:**
- Client ID e Secret
- Redirect URIs
- Scopes: `calendar`, `gmail.readonly`

### 11.3 Microsoft Integration

**Serviços:**
- Microsoft Graph API
- Outlook Calendar
- Office 365

**Configuração:**
- Azure App Registration
- Client ID e Secret
- Tenant ID

### 11.4 Twilio Integration

**Serviços:**
- VoIP: Chamadas de voz
- WhatsApp Business API
- SMS

**Configuração:**
- Account SID
- Auth Token
- Phone Number

### 11.5 AWS Integration

**Serviços:**
- **S3**: Armazenamento de arquivos
- **SES**: Envio de e-mails
- **SQS**: Filas de mensagens
- **DynamoDB**: Banco NoSQL

**Configuração:**
- Access Key ID
- Secret Access Key
- Region

### 11.6 OpenAI Integration

**Serviços:**
- GPT Models: Geração de texto
- DALL-E: Geração de imagens
- Whisper: Transcrição de áudio
- Embeddings: Busca vetorial

**Configuração:**
- API Key
- Organization (opcional)

---

## 12. Sistema de Workflows

### 12.1 Arquitetura

O sistema implementa um **workflow engine** inspirado no n8n:

```
Workflow Definition (JSON)
    ↓
WorkflowRunner
    ↓
WorkflowExecute (Core Engine)
    ↓
Nodes (Processamento)
    ↓
Connections (Fluxo)
    ↓
Execution Result
```

### 12.2 Componentes

**WorkflowFlow (Model):**
- Armazena definição do workflow
- Campos: `name`, `active`, `nodes` (JSON), `connections` (JSON), `settings` (JSON)

**WorkflowFlowExecution (Model):**
- Armazena execuções
- Campos: `workflow_flow_id`, `mode`, `status`, `input_data`, `output_data`, `error_data`

**WorkflowExecutionLog (Model):**
- Logs detalhados de execução
- Campos: `workflow_flow_execution_id`, `node_id`, `node_type`, `level`, `message`

### 12.3 Modos de Execução

1. **Manual**: Execução manual pelo usuário
2. **Trigger**: Execução por trigger/evento
3. **Webhook**: Execução via webhook HTTP
4. **Schedule**: Execução agendada (cron)

### 12.4 Status de Execução

- `new`: Nova execução
- `running`: Em execução
- `success`: Sucesso
- `error`: Erro
- `waiting`: Aguardando

### 12.5 Filas de Workflow

- `workflow-execution`: Execução de workflows
- `webhook-processing`: Processamento de webhooks
- `workflow-schedule`: Agendamento
- `workflow-execution-dlq`: Dead Letter Queue

---

## 13. Sistema de Mídia e Armazenamento

### 13.1 Laravel Storage

O sistema utiliza **Laravel Storage** nativo (não unified storage):

**Drivers Suportados:**
- `local`: Sistema de arquivos local
- `s3`: Amazon S3
- `public`: Arquivos públicos

### 13.2 Sistema de Mídia (Mediable)

Utiliza **plank/laravel-mediable ^6.1**:

**Tabelas:**
- `media`: Arquivos de mídia
- `mediables`: Relação polimórfica

**Uso:**
```php
$model->attachMedia($media, 'tag');
$model->getMedia('tag');
```

### 13.3 Armazenamento de Mídia WhatsApp

**WhatsappMediaFile:**
- Armazena metadados de mídia
- Caminho físico no storage
- Relação com mensagem

**Processamento:**
- Download assíncrono via job
- Armazenamento em `storage/app/whatsapp/media`
- Thumbnails para imagens

---

## 14. Configurações e Ambiente

### 14.1 Variáveis de Ambiente Principais

```env
# Aplicação
APP_NAME=Concord CRM
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com
APP_KEY=base64:...

# Banco de Dados
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=concordcrm
DB_USERNAME=root
DB_PASSWORD=
DB_PREFIX=tbl_

# Redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

# Broadcast
BROADCAST_CONNECTION=reverb
REVERB_APP_KEY=
REVERB_APP_SECRET=
REVERB_APP_ID=
REVERB_HOST=localhost
REVERB_PORT=8080
REVERB_SCHEME=http

# Queue
QUEUE_CONNECTION=redis

# Storage
FILESYSTEM_DISK=local
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=
AWS_BUCKET=

# Integrações
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
MICROSOFT_CLIENT_ID=
MICROSOFT_CLIENT_SECRET=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
OPENAI_API_KEY=
```

### 14.2 Configurações por Módulo

Cada módulo pode ter seu próprio arquivo de configuração em `modules/ModuleName/config/`.

### 14.3 Sistema de Settings

O sistema possui um **Settings Manager** centralizado:

- Armazenamento em `settings` table
- Cache de configurações
- Drivers: `database`, `cache`

**Uso:**
```php
settings('key', 'default');
settings(['key' => 'value']);
```

---

## 15. Considerações de Performance

### 15.1 Cache

- **Model Caching**: Cache de modelos Eloquent
- **Redis Cache**: Cache de dados
- **Config Cache**: Cache de configurações
- **Route Cache**: Cache de rotas

### 15.2 Otimizações de Query

- **Eager Loading**: `with()` para evitar N+1
- **Lazy Loading**: Carregamento sob demanda
- **Query Scopes**: Reutilização de queries
- **Indexes**: Índices em colunas frequentes

### 15.3 Processamento Assíncrono

- Jobs em filas para operações pesadas
- Processamento em lote (chunking)
- Timeouts apropriados
- Retry com backoff exponencial

---

## 16. Segurança

### 16.1 Autenticação

- Laravel Sanctum para API
- CSRF protection
- Password hashing (bcrypt)
- Token expiration

### 16.2 Autorização

- Policy-based authorization
- Role-based access control
- Resource-level permissions
- Team-based access control

### 16.3 Validação

- Form Request validation
- Input sanitization
- SQL injection protection (Eloquent)
- XSS protection

### 16.4 Criptografia

- AES-256-CBC encryption
- Encrypted attributes
- Secure storage de tokens

---

## 17. Logging e Monitoramento

### 17.1 Logging

- Laravel Log (Monolog)
- Channels: `single`, `daily`, `slack`
- Levels: `debug`, `info`, `warning`, `error`, `critical`

### 17.2 Activity Log

- **spatie/laravel-activitylog ^4.3**
- Log de ações de usuários
- Tabela: `activity_log`
- Campos: `subject_type`, `subject_id`, `causer_type`, `causer_id`, `description`, `properties`

### 17.3 Monitoramento de Filas

- Thresholds de warning/critical
- Dead Letter Queue
- Retry tracking

---

## 18. Internacionalização (i18n)

### 18.1 Sistema de Tradução

- Laravel Translation
- Módulo Translator customizado
- Arquivos em `modules/ModuleName/lang/{locale}/`

### 18.2 Locales Suportados

- `pt_BR`: Português (Brasil) - Padrão
- `en`: Inglês
- `es`: Espanhol
- `ru`: Russo

### 18.3 Frontend i18n

- **vue-i18n ^9.14.2**
- Traduções sincronizadas com backend
- Lazy loading de traduções

---

## 19. Testes

### 19.1 Estrutura

- **PHPUnit ^11.0.1** para testes backend
- Testes em `tests/` e `modules/ModuleName/tests/`
- Factories para modelos

### 19.2 Tipos de Testes

- **Unit Tests**: Testes unitários
- **Feature Tests**: Testes de funcionalidade
- **Integration Tests**: Testes de integração

---

## 20. Deploy e Manutenção

### 20.1 Comandos Úteis

```bash
# Cache
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Queue
php artisan queue:work
php artisan queue:restart

# Reverb
php artisan reverb:start

# Migrations
php artisan migrate
php artisan migrate:rollback

# Modules
php artisan module:list
php artisan module:enable ModuleName
php artisan module:disable ModuleName
```

### 20.2 Supervisor Configuration

```ini
[program:concordcrm-queue]
command=php /path/to/artisan queue:work --sleep=3 --tries=3
autostart=true
autorestart=true
user=www-data
```

---

## 21. Referências Técnicas

### 21.1 Arquivos de Configuração Principais

- `config/app.php`: Configuração da aplicação
- `config/database.php`: Configuração de banco
- `config/broadcasting.php`: Configuração de broadcast
- `config/queue.php`: Configuração de filas
- `config/permission.php`: Configuração de permissões
- `config/reverb.php`: Configuração do Reverb

### 21.2 Arquivos de Rotas

- `routes/web.php`: Rotas web
- `routes/api.php`: Rotas API
- `routes/channels.php`: Canais broadcast
- `modules/ModuleName/routes/api.php`: Rotas do módulo
- `modules/ModuleName/routes/channels.php`: Canais do módulo

### 21.3 Service Providers Principais

- `App\Providers\AppServiceProvider`
- `Modules\Core\Providers\BootstrapServiceProvider`
- `Modules\Core\Providers\CoreServiceProvider`
- `Modules\ModuleName\Providers\ModuleNameServiceProvider`

---

## 22. Conclusão

Este documento apresenta uma visão técnica completa do sistema CRM, cobrindo:

- ✅ Arquitetura modular baseada em Laravel
- ✅ Frontend Vue.js 3 com composables
- ✅ Sistema de broadcast com Laravel Reverb
- ✅ Sistema de filas Redis para processamento assíncrono
- ✅ Sistema de permissões e teams
- ✅ Integração WhatsApp completa
- ✅ Sistema de workflows
- ✅ Múltiplas integrações externas

O sistema é **altamente modular**, **escalável** e **extensível**, permitindo fácil adição de novos módulos e funcionalidades.

---

**Última Atualização:** 2025-01-07  
**Versão do Documento:** 1.0  
**Mantido por:** Equipe de Desenvolvimento
