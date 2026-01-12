# Arquitetura Multitenancy - CRM & WhatsApp API

## 📋 Índice

1. [Visão Geral](#visão-geral)
2. [Arquitetura do Sistema](#arquitetura-do-sistema)
3. [Identificação de Tenants](#identificação-de-tenants)
4. [Isolamento de Dados](#isolamento-de-dados)
5. [API WhatsApp e Sessões](#api-whatsapp-e-sessões)
6. [Jobs e Filas](#jobs-e-filas)
7. [Reverb e Broadcasting](#reverb-e-broadcasting)
8. [PM2 e Processos](#pm2-e-processos)
9. [Fluxo de Dados](#fluxo-de-dados)
10. [Configuração e Deploy](#configuração-e-deploy)

---

## 1. Visão Geral

Este sistema implementa **multitenancy** (multi-inquilino) seguindo o padrão **Shared Database, Separate Schema**, onde:

- **Laravel CRM**: Todos os tenants compartilham o mesmo banco de dados, mas os dados são isolados através de uma coluna `tenant_id` em cada tabela
- **WhatsApp API (Node.js)**: As sessões são isoladas por tenant através de diretórios separados (`tenant_{tenantId}/session_{sessionId}`)
- **Webhooks**: Incluem `tenantId` no payload para garantir processamento no contexto correto

### Benefícios desta Arquitetura

✅ **Isolamento de Dados**: Cada tenant vê apenas seus próprios dados  
✅ **Escalabilidade**: Fácil adicionar novos tenants sem mudanças estruturais  
✅ **Manutenção Simplificada**: Um único banco de dados para gerenciar  
✅ **Performance**: Índices otimizados por `tenant_id`  
✅ **Segurança**: Isolamento automático através de Global Scopes  

---

## 2. Arquitetura do Sistema

### 2.1 Componentes Principais

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENTE (Browser)                         │
│  - Subdomínio: tenant1.crm.com                              │
│  - Header: X-Tenant-Slug: tenant1                          │
└──────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              Laravel CRM (PHP)                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Middleware: IdentifyTenant                           │  │
│  │  - Identifica tenant via subdomínio/header           │  │
│  │  - Define contexto no TenantService                  │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  TenantService                                        │  │
│  │  - Gerencia tenant atual na requisição               │  │
│  │  - Armazenado no container Laravel                   │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  TenantScope (Global Scope)                          │  │
│  │  - Filtra automaticamente queries por tenant_id       │  │
│  │  - Aplicado em todos os modelos com BelongsToTenant │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Database (MySQL)                                     │  │
│  │  - Tabela: tenants (id, slug, domain, settings)     │  │
│  │  - Todas as tabelas têm tenant_id                    │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────────┘
                      │
                      │ HTTP Request (tenantId no payload)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│         WhatsApp API (Node.js + Express)                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  SessionManager                                       │  │
│  │  - Cria diretórios: tenant_{tenantId}/session_{id}   │  │
│  │  - Armazena tenantId em sessionInfo                  │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  WhatsAppService                                      │  │
│  │  - Gerencia clientes WhatsApp Web.js                  │  │
│  │  - Processa mensagens e eventos                       │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  WebhookService                                       │  │
│  │  - Envia webhooks com tenantId no payload            │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  File System                                          │  │
│  │  - sessions/tenant_1/session_abc123/                 │  │
│  │  - sessions/tenant_2/session_def456/                 │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Fluxo de Requisição

```
1. Cliente faz requisição → tenant1.crm.com/api/deals
2. IdentifyTenant Middleware:
   - Extrai subdomínio: "tenant1"
   - Busca tenant no banco: Tenant::findBySlug('tenant1')
   - Define no TenantService: setCurrentTenant($tenant)
3. Controller processa requisição:
   - Deal::all() → Automaticamente filtrado por tenant_id
   - TenantScope aplica: WHERE tenant_id = 1
4. Resposta retornada apenas com dados do tenant1
```

---

## 3. Identificação de Tenants

### 3.1 Métodos de Identificação

O sistema identifica tenants através de **3 métodos**, em ordem de prioridade:

#### Prioridade 1: Headers HTTP
```http
X-Tenant-ID: 1
X-Tenant-Slug: tenant1
```

#### Prioridade 2: Subdomínio
```
tenant1.crm.com → Identifica tenant com slug "tenant1"
```

#### Prioridade 3: Domínio Completo
```
crm-tenant1.com → Identifica tenant com domain "crm-tenant1.com"
```

### 3.2 Implementação

**Arquivo**: `app/Http/Middleware/IdentifyTenant.php`

```php
public function handle(Request $request, Closure $next): Response
{
    // 1. Identificar tenant
    $tenant = $this->identifyTenant($request);
    
    // 2. Se não encontrou, usar default
    if (!$tenant) {
        $tenant = Tenant::getDefault();
    }
    
    // 3. Validar se está ativo
    if (!$tenant->is_active) {
        return response()->json(['error' => 'Tenant is not active'], 403);
    }
    
    // 4. Definir no contexto
    TenantService::setCurrentTenant($tenant);
    
    // 5. Adicionar ao request
    $request->merge(['tenant_id' => $tenant->id]);
    
    return $next($request);
}
```

### 3.3 Registro do Middleware

**Arquivo**: `bootstrap/app.php`

```php
$middleware->web(append: [
    \App\Http\Middleware\IdentifyTenant::class,
    // ... outros middlewares
]);

$middleware->api(append: [
    \App\Http\Middleware\IdentifyTenant::class,
    // ... outros middlewares
]);
```

---

## 4. Isolamento de Dados

### 4.1 Global Scope (TenantScope)

**Arquivo**: `modules/Core/app/Scopes/TenantScope.php`

```php
class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $tenant = TenantService::getCurrentTenant();
        
        if ($tenant) {
            $builder->where($model->getTable().'.tenant_id', $tenant->id);
        }
    }
}
```

**Como funciona:**
- Aplicado automaticamente em todas as queries
- Filtra por `tenant_id` antes de executar a query
- Transparente para o desenvolvedor

### 4.2 Trait BelongsToTenant

**Arquivo**: `modules/Core/app/Concerns/BelongsToTenant.php`

```php
trait BelongsToTenant
{
    public static function bootBelongsToTenant(): void
    {
        // 1. Aplicar Global Scope
        static::addGlobalScope(new TenantScope);
        
        // 2. Auto-preencher tenant_id ao criar
        static::creating(function ($model) {
            if (!$model->tenant_id) {
                $model->tenant_id = TenantService::getCurrentTenantId();
            }
        });
    }
    
    // 3. Relacionamento com Tenant
    public function tenant(): BelongsTo
    {
        return $this->belongsTo(Tenant::class);
    }
}
```

### 4.3 Uso em Modelos

**Exemplo**: `modules/Users/app/Models/User.php`

```php
class User extends Model
{
    use BelongsToTenant; // ← Adiciona isolamento automático
    
    protected $fillable = [
        'name', 'email', 'password',
        'tenant_id', // ← Campo obrigatório
    ];
}
```

**Resultado:**
```php
// Automaticamente filtrado por tenant_id
User::all(); // WHERE tenant_id = 1

// Bypass do scope (quando necessário)
User::withAllTenants()->get(); // Todos os tenants

// Query específica para um tenant
User::forTenant(2)->get(); // WHERE tenant_id = 2
```

### 4.4 Tabelas com tenant_id

As seguintes tabelas possuem `tenant_id`:

- ✅ `users`
- ✅ `deals`
- ✅ `contacts`
- ✅ `companies`
- ✅ `activities`
- ✅ `teams`
- ✅ `whatsapp_devices`
- ✅ `whatsapp_messages`
- ✅ `documents`
- ✅ `notes`
- ✅ `pipelines`
- ✅ `stages`
- ✅ `products`
- ✅ `workflow_flows`

---

## 5. API WhatsApp e Sessões

### 5.1 Estrutura de Diretórios

```
api_whatsapp/
└── sessions/
    ├── tenant_1/
    │   ├── session_abc123/
    │   │   ├── session.json
    │   │   └── ... (arquivos do WhatsApp Web.js)
    │   └── session_def456/
    │       └── ...
    └── tenant_2/
        └── session_xyz789/
            └── ...
```

### 5.2 SessionManager

**Arquivo**: `api_whatsapp/src/services/session-manager.service.js`

```javascript
class SessionManager {
    getTenantSessionPath(tenantId, sessionId) {
        if (tenantId) {
            return path.join(this.sessionsPath, `tenant_${tenantId}`, sessionId);
        }
        // Backward compatibility: default tenant
        return path.join(this.sessionsPath, sessionId);
    }
    
    async createSession(options = {}) {
        const sessionId = options.sessionId || uuidv4();
        const tenantId = options.tenantId || null;
        const sessionPath = this.getTenantSessionPath(tenantId, sessionId);
        
        const sessionInfo = {
            id: sessionId,
            tenantId: tenantId, // ← Armazenado na sessão
            path: sessionPath,
            createdAt: new Date().toISOString(),
            status: 'created',
        };
        
        await this.saveSessionInfo(sessionId, sessionInfo);
        return sessionInfo;
    }
}
```

### 5.3 Criação de Sessão (Laravel → Node.js)

**Arquivo**: `modules/Whatsapp/app/Services/WhatsappSessionManager.php`

```php
public function createSession(array $data): array
{
    // 1. Obter tenant_id atual
    $tenantId = TenantService::getCurrentTenantId();
    
    // 2. Chamar API Node.js com tenantId
    $result = $this->apiService->post('/api/whatsapp/sessions', [
        'tenantId' => $tenantId, // ← Enviado para API
        'sessionId' => $sessionId,
        'phoneNumber' => $phoneNumber,
        ...$options,
    ]);
    
    // 3. Salvar dispositivo no banco (já com tenant_id via BelongsToTenant)
    $device = WhatsappDevice::create([
        'session_id' => $sessionId,
        'device_name' => $deviceName,
        // tenant_id é preenchido automaticamente pelo trait
    ]);
}
```

### 5.4 Webhooks com tenantId

**Arquivo**: `api_whatsapp/src/services/webhook.service.js`

```javascript
async sendWebhook(sessionId, messageData, tenantId = null) {
    const payload = {
        ...messageData,
        tenantId: tenantId, // ← Incluído no payload
    };
    
    await this.makeRequest(webhookConfig, payload);
}
```

**Arquivo**: `api_whatsapp/src/services/handlers/webhook-message-handler.js`

```javascript
async sendWebhookIfConfigured(sessionId, messageData) {
    const sessionInfo = await this.whatsappService.sessionManager.getSession(sessionId);
    const tenantId = sessionInfo?.tenantId || null; // ← Obtido da sessão
    
    await this.whatsappService.webhookService.sendWebhook(
        sessionId,
        webhookPayload,
        tenantId // ← Passado para o serviço
    );
}
```

### 5.5 Processamento de Webhook (Laravel)

**Arquivo**: `modules/Whatsapp/app/Jobs/ProcessWhatsappWebhook.php`

```php
class ProcessWhatsappWebhook implements ShouldQueue
{
    public function handle()
    {
        $tenantId = $this->payload['tenantId'] ?? null;
        
        if ($tenantId) {
            // Definir tenant no contexto do job
            $tenant = Tenant::find($tenantId);
            TenantService::setCurrentTenant($tenant);
        }
        
        // Processar mensagem (já no contexto do tenant correto)
        // ...
    }
}
```

---

## 6. Jobs e Filas

### 6.1 Configuração de Filas

**Arquivo**: `config/queue.php`

```php
'default' => env('QUEUE_CONNECTION', 'sync'), // sync, database, redis

'connections' => [
    'database' => [
        'driver' => 'database',
        'table' => 'jobs',
        'queue' => 'default',
    ],
    
    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => 'default',
    ],
],
```

### 6.2 Jobs Relacionados ao WhatsApp

#### ProcessWhatsappWebhook
- **Fila**: `whatsapp-webhooks`
- **Função**: Processa webhooks recebidos da API Node.js
- **Tenant**: Definido via `tenantId` no payload

#### ProcessWhatsappSync
- **Fila**: `whatsapp-sync`
- **Função**: Sincroniza mensagens do WhatsApp
- **Tenant**: Contexto do tenant atual

#### SyncConversationMessagesInChunks
- **Fila**: `whatsapp-sync`
- **Função**: Sincroniza mensagens em chunks para performance
- **Tenant**: Contexto do tenant atual

### 6.3 Jobs de Workflow

#### ExecuteWorkflowJob
- **Fila**: `workflow-execution`
- **Função**: Executa workflows de automação
- **Tenant**: Contexto do tenant atual (via BelongsToTenant)

### 6.4 Processamento de Jobs

**Comando**: `php artisan queue:work`

```bash
# Processar fila específica
php artisan queue:work --queue=whatsapp-webhooks

# Processar múltiplas filas
php artisan queue:work --queue=whatsapp-webhooks,whatsapp-sync

# Com supervisor (produção)
php artisan queue:work --queue=whatsapp-webhooks --tries=3 --timeout=60
```

### 6.5 Isolamento de Jobs

Os jobs **herdam o contexto do tenant** da requisição que os despachou:

```php
// No controller
TenantService::setCurrentTenant($tenant); // tenant_id = 1

// Despachar job
ProcessWhatsappWebhook::dispatch($payload);
// Job será executado no contexto do tenant 1
```

---

## 7. Reverb e Broadcasting

### 7.1 O que é Reverb?

**Reverb** é o servidor WebSocket oficial do Laravel para broadcasting em tempo real.

### 7.2 Configuração

**Arquivo**: `config/reverb.php`

```php
'servers' => [
    'reverb' => [
        'host' => env('REVERB_SERVER_HOST', '0.0.0.0'),
        'port' => env('REVERB_SERVER_PORT', 8080),
        'hostname' => env('REVERB_HOSTNAME', 'localhost'),
    ],
],

'apps' => [
    'apps' => [
        [
            'key' => env('REVERB_APP_KEY'),
            'secret' => env('REVERB_APP_SECRET'),
            'app_id' => env('REVERB_APP_ID'),
        ],
    ],
],
```

**Arquivo**: `config/broadcasting.php`

```php
'default' => env('BROADCAST_CONNECTION', 'reverb'),

'connections' => [
    'reverb' => [
        'driver' => 'reverb',
        'key' => env('REVERB_APP_KEY'),
        'secret' => env('REVERB_APP_SECRET'),
        'app_id' => env('REVERB_APP_ID'),
        'options' => [
            'host' => env('REVERB_HOST'),
            'port' => env('REVERB_PORT', 8080),
            'scheme' => env('REVERB_SCHEME', 'http'),
        ],
    ],
],
```

### 7.3 Broadcasting com Tenant Context

**Exemplo**: Broadcast de nova mensagem WhatsApp

```php
// No controller ou job
$tenant = TenantService::getCurrentTenant();

Broadcast::channel("tenant.{$tenant->id}.whatsapp", function ($user) use ($tenant) {
    return $user->tenant_id === $tenant->id;
});

// Broadcasting
event(new WhatsappMessageReceived($message));
```

**Frontend**: `modules/Core/resources/js/services/Broadcast.js`

```javascript
// Conectar ao canal do tenant
const tenantId = window.currentTenant?.id;
Echo.private(`tenant.${tenantId}.whatsapp`)
    .listen('WhatsappMessageReceived', (e) => {
        console.log('Nova mensagem:', e.message);
    });
```

### 7.4 Iniciar Reverb

```bash
php artisan reverb:start
```

**Com PM2** (recomendado para produção):

```javascript
// ecosystem.config.js (Laravel)
{
    name: 'reverb',
    script: 'php',
    args: 'artisan reverb:start',
    instances: 1,
    autorestart: true,
}
```

---

## 8. PM2 e Processos

### 8.1 O que é PM2?

**PM2** é um gerenciador de processos para Node.js que mantém aplicações rodando em background.

### 8.2 Configuração (WhatsApp API)

**Arquivo**: `api_whatsapp/ecosystem.config.js`

```javascript
module.exports = {
    apps: [
        {
            name: 'whatsapp-api',
            script: 'start.js',
            cwd: '/var/www/crm/api_whatsapp',
            instances: 1,
            autorestart: true,
            watch: false,
            max_memory_restart: '1G',
            env: {
                NODE_ENV: 'production',
                PORT: 3000,
            },
        },
    ],
};
```

### 8.3 Comandos PM2

```bash
# Iniciar aplicação
pm2 start ecosystem.config.js

# Parar aplicação
pm2 stop whatsapp-api

# Reiniciar aplicação
pm2 restart whatsapp-api

# Ver logs
pm2 logs whatsapp-api

# Ver status
pm2 status

# Salvar configuração
pm2 save

# Configurar para iniciar no boot
pm2 startup
```

### 8.4 Estrutura de Processos

```
PM2
├── whatsapp-api (Node.js)
│   └── Porta: 3000
│   └── Gerencia sessões WhatsApp
│
Laravel Queue Workers
├── queue:work --queue=whatsapp-webhooks
├── queue:work --queue=whatsapp-sync
└── queue:work --queue=workflow-execution

Reverb Server
└── php artisan reverb:start
    └── Porta: 8080 (WebSocket)
```

### 8.5 Monitoramento

```bash
# Dashboard PM2
pm2 monit

# Métricas
pm2 describe whatsapp-api
```

---

## 9. Fluxo de Dados

### 9.1 Fluxo Completo: Envio de Mensagem

```
1. Usuário envia mensagem no CRM
   └─> Controller: WhatsappMessageController@send
       └─> TenantService::getCurrentTenantId() → tenant_id = 1

2. Laravel faz requisição para API Node.js
   POST /api/whatsapp/sessions/{sessionId}/send
   Headers: { "X-Tenant-Slug": "tenant1" }
   Body: { "message": "Olá!", "to": "5511999999999" }

3. API Node.js processa
   └─> WhatsAppService.sendMessage()
       └─> Cliente WhatsApp Web.js envia mensagem

4. WhatsApp Web.js recebe confirmação
   └─> EventHandler.onMessageAck()
       └─> WebhookService.sendWebhook()
           └─> Payload: { ..., "tenantId": 1 }

5. Laravel recebe webhook
   POST /api/whatsapp/webhook
   └─> ProcessWhatsappWebhook::dispatch($payload)
       └─> Job processa no contexto do tenant 1
           └─> Salva mensagem no banco (com tenant_id = 1)

6. Broadcasting via Reverb
   └─> event(new WhatsappMessageReceived($message))
       └─> Canal: tenant.1.whatsapp
           └─> Frontend recebe atualização em tempo real
```

### 9.2 Fluxo: Criação de Sessão

```
1. Usuário cria dispositivo WhatsApp no CRM
   └─> Controller: WhatsappDeviceController@store
       └─> TenantService::getCurrentTenantId() → tenant_id = 1

2. Laravel chama API Node.js
   POST /api/whatsapp/sessions
   Body: { "tenantId": 1, "sessionId": "device_abc123" }

3. API Node.js cria sessão
   └─> SessionManager.createSession({ tenantId: 1, ... })
       └─> Cria diretório: sessions/tenant_1/device_abc123/
       └─> Salva sessionInfo com tenantId

4. WhatsApp Web.js gera QR Code
   └─> Retorna QR Code para Laravel

5. Laravel salva dispositivo
   └─> WhatsappDevice::create([..., "tenant_id" => 1])
       └─> tenant_id preenchido automaticamente pelo trait
```

---

## 10. Configuração e Deploy

### 10.1 Variáveis de Ambiente

**.env (Laravel)**

```env
# Tenant
TENANT_DEFAULT_SLUG=default

# Database
DB_CONNECTION=mysql
DB_PREFIX=tbl_

# Queue
QUEUE_CONNECTION=redis

# Reverb
REVERB_APP_KEY=your-key
REVERB_APP_SECRET=your-secret
REVERB_APP_ID=your-app-id
REVERB_HOST=localhost
REVERB_PORT=8080
REVERB_SCHEME=http

# WhatsApp API
WHATSAPP_API_URL=http://localhost:3000
WHATSAPP_WEBHOOK_URL=http://localhost:8000/api/whatsapp/webhook
```

**.env (Node.js - api_whatsapp)**

```env
PORT=3000
NODE_ENV=production
DEFAULT_WEBHOOK_URL=http://localhost:8000/api/whatsapp/webhook
```

### 10.2 Migrations

```bash
# Executar migrations
php artisan migrate

# Migrar dados existentes para tenant default
php artisan tenant:migrate-existing-data --tenant=default
```

### 10.3 Seeders

```bash
# Criar tenant default
php artisan db:seed --class=TenantSeeder
```

### 10.4 Iniciar Serviços

```bash
# 1. Laravel Queue Workers
php artisan queue:work --queue=whatsapp-webhooks,whatsapp-sync

# 2. Reverb Server
php artisan reverb:start

# 3. WhatsApp API (Node.js)
cd api_whatsapp
pm2 start ecosystem.config.js
```

### 10.5 Supervisor (Produção)

**Arquivo**: `/etc/supervisor/conf.d/crm-queue.conf`

```ini
[program:crm-queue-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/crm/artisan queue:work --queue=whatsapp-webhooks,whatsapp-sync --tries=3 --timeout=60
autostart=true
autorestart=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/crm/storage/logs/queue-worker.log
```

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start crm-queue-worker:*
```

---

## 11. Boas Práticas

### 11.1 Sempre Use o Trait

```php
// ✅ Correto
class Deal extends Model
{
    use BelongsToTenant;
}

// ❌ Errado (sem isolamento)
class Deal extends Model
{
    // Sem BelongsToTenant
}
```

### 11.2 Bypass do Scope Apenas Quando Necessário

```php
// ✅ Correto: Bypass apenas para operações administrativas
User::withAllTenants()->get(); // Admin precisa ver todos

// ❌ Errado: Bypass em queries normais
User::withAllTenants()->where('email', 'user@example.com')->first();
```

### 11.3 Sempre Inclua tenantId em Webhooks

```javascript
// ✅ Correto
await webhookService.sendWebhook(sessionId, payload, tenantId);

// ❌ Errado
await webhookService.sendWebhook(sessionId, payload);
```

### 11.4 Valide Tenant em Jobs

```php
// ✅ Correto
public function handle()
{
    $tenantId = $this->payload['tenantId'] ?? null;
    if ($tenantId) {
        $tenant = Tenant::find($tenantId);
        TenantService::setCurrentTenant($tenant);
    }
    // ... processar
}
```

---

## 12. Troubleshooting

### 12.1 Dados de Outro Tenant Aparecendo

**Problema**: Query retorna dados de múltiplos tenants

**Solução**:
1. Verificar se o modelo usa `BelongsToTenant`
2. Verificar se `TenantService::getCurrentTenant()` retorna o tenant correto
3. Verificar se o middleware `IdentifyTenant` está registrado

### 12.2 Webhook Sem tenantId

**Problema**: Webhook não inclui `tenantId` no payload

**Solução**:
1. Verificar se `sessionInfo.tenantId` está sendo passado
2. Verificar se `webhookService.sendWebhook()` recebe `tenantId`

### 12.3 Sessões Misturadas Entre Tenants

**Problema**: Sessões de um tenant aparecem em outro

**Solução**:
1. Verificar estrutura de diretórios: `sessions/tenant_{id}/session_{id}`
2. Verificar se `getTenantSessionPath()` está sendo usado corretamente

---

## 13. Conclusão

Esta arquitetura multitenancy oferece:

✅ **Isolamento Completo**: Dados e sessões isolados por tenant  
✅ **Escalabilidade**: Fácil adicionar novos tenants  
✅ **Manutenibilidade**: Código limpo e organizado  
✅ **Performance**: Índices otimizados e queries eficientes  
✅ **Segurança**: Isolamento automático através de scopes  

O sistema está pronto para produção e pode suportar múltiplos tenants de forma segura e eficiente.

---

**Última atualização**: 2026-01-12  
**Versão**: 1.0.0
