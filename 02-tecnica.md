# Ticketing — Documentazione Tecnica

## Guida per Sviluppatori

**Versione:** 1.0
**Stack:** Laravel 12 · PHP 8.2+ · Go · MySQL · Redis
**Ultimo aggiornamento:** 2025

---

## 1. Prerequisiti

| Strumento | Versione minima            |
| --------- | -------------------------- |
| PHP       | 8.2+                       |
| Composer  | 2.x                        |
| MySQL     | 8.0+                       |
| Redis     | 7.x                        |
| Go        | 1.21+                      |
| Node.js   | 18+ (per frontend Lovable) |

---

## 2. Setup del Progetto

### 2.1 Clonare le repository

```bash
# Backend Laravel
git clone https://github.com/gambadaniele1-hue/ticketing-api.git
cd ticketing-api

# Microservizio mail Go
git clone https://github.com/gambadaniele1-hue/ticketing-mail.git
```

### 2.2 Configurare il backend Laravel

```bash
# Installa dipendenze
composer install

# Copia il file di configurazione
cp .env.example .env

# Genera la chiave applicativa
php artisan key:generate
```

### 2.3 Configurare il file .env

Le variabili di ambiente principali da configurare:

```env
# Applicazione
APP_NAME=Ticketing
APP_ENV=local
APP_URL=http://localhost

# Database globale
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=ticketing_global
DB_USERNAME=root
DB_PASSWORD=

# Redis (coda messaggi per microservizio mail)
REDIS_HOST=127.0.0.1
REDIS_PORT=6379

# JWT
JWT_SECRET=
JWT_IDENTITY_TTL=900        # 15 minuti in secondi
JWT_ACCESS_TTL=3600         # 1 ora in secondi
JWT_REFRESH_TTL=604800      # 7 giorni in secondi

# Tenancy
TENANCY_DATABASE_AUTO_DELETE=false
CENTRAL_DOMAINS=localhost
```

### 2.4 Eseguire le migration

```bash
# Migration del database globale + seeder (piani, ruoli ecc.)
php artisan migrate --seed

# Migration del database tenant condiviso (shared)
php artisan migrate --database="shared" --path="database/migrations/tenant"

# Avvia il server
php artisan serve
```

---

## 3. Architettura Multi-Tenant

Il progetto usa il package **stancl/tenancy ^3.10** in modalità ibrida. Ogni richiesta HTTP viene instradata al database corretto in base al sottodominio tramite il middleware `InitializeTenancyByDomain`.

### Flusso di una richiesta tenant

```
Richiesta HTTP
alpha.piattaforma.com/api/v1/auth/me
        │
        ▼
InitializeTenancyByDomain
(legge il sottodominio "alpha",
 trova il tenant nel DB globale,
 switcha la connessione al DB tenant)
        │
        ▼
PreventAccessFromCentralDomains
(blocca se la richiesta arriva
 dal dominio centrale)
        │
        ▼
JwtMiddleware
(valida il JWT, verifica tenant_id)
        │
        ▼
Controller
```

### Modelli globali vs tenant

I modelli sono separati in due namespace distinti:

```
Models/
├── Global/          ← operano sul DB globale
│   ├── GlobalIdentity.php
│   ├── Plan.php
│   ├── Tenant.php
│   ├── TenantMembership.php
│   └── RefreshToken.php
└── Tenant/          ← operano sul DB del tenant attivo
    ├── User.php
    ├── Role.php
    ├── Permission.php
    ├── Team.php
    ├── Category.php
    ├── Ticket.php
    ├── Message.php
    └── SlaPolicy.php
```

I modelli tenant usano il trait `BelongsToTenantHybrid` per collegarsi automaticamente al database del tenant corrente.

---

## 4. Autenticazione JWT

### 4.1 I tre token

L'autenticazione è gestita da `JwtService` tramite `firebase/php-jwt`. I token vengono trasportati via **cookie HttpOnly + Secure + SameSite=Strict**.

| Token    | Durata   | Payload                               | Dove                    |
| -------- | -------- | ------------------------------------- | ----------------------- |
| Identity | 15 min   | `type`, `sub`, `email`                | Cookie `identity_token` |
| Access   | 1 ora    | `type`, `sub`, `tenant_id`, `role_id` | Cookie `access_token`   |
| Refresh  | 7 giorni | stringa 64 char (hash SHA-256 nel DB) | Cookie `refresh_token`  |

### 4.2 JwtService

Il service espone i metodi principali:

```php
// Genera un Identity Token
JwtService::generateIdentityToken(GlobalIdentity $identity): string

// Genera un Access Token per un tenant specifico
JwtService::generateAccessToken(User $user, string $tenantId): string

// Genera e salva un Refresh Token
JwtService::generateRefreshToken(GlobalIdentity $identity): string

// Valida un token e restituisce il payload
JwtService::validate(string $token): object

// Revoca un refresh token (ban / logout)
JwtService::revokeRefreshToken(string $token): void
```

### 4.3 JwtMiddleware

Applicato a tutte le route protette. Verifica:

1. Presenza e validità del cookie `access_token`.
2. Firma JWT corretta.
3. Token non scaduto.
4. `tenant_id` nel payload corrisponde al tenant della richiesta corrente — **prevenzione attacchi cross-tenant**.

### 4.4 RBAC — Controllo permessi

Ogni utente ha un ruolo, ogni ruolo ha permessi identificati da slug. Il controllo avviene tramite:

```php
// Nel controller o nel middleware
if (!$user->hasPermission('tickets.create')) {
    abort(403);
}
```

**Slug dei permessi (esempi):**

| Slug                | Descrizione                          |
| ------------------- | ------------------------------------ |
| `tickets.view`      | Visualizza i ticket del proprio team |
| `tickets.create`    | Apre nuovi ticket                    |
| `tickets.assign`    | Assegna ticket a un agente           |
| `team.manage`       | Gestisce i membri del team           |
| `categories.manage` | Crea e modifica categorie            |
| `sla.manage`        | Gestisce le SLA policy               |
| `admin.dashboard`   | Accede alla dashboard admin          |

---

## 5. Registrazione Tenant

Il flusso è gestito da `TenantRegistrationService` e due Job asincroni.

### Sequenza operazioni

```
TenantRegistrationController@store
        │
        ▼
StoreTenantRequest (validazione)
- nome azienda
- sottodominio (unique su tabella domains)
- piano scelto
- email Admin
- password Admin
        │
        ▼
TenantRegistrationService
1. Verifica disponibilità sottodominio
2. INSERT tenants (DB globale)
3. INSERT domains
4. Dispatch Job: CreateTenantMysqlUser
   └── Crea il database dedicato (se piano = dedicated)
        o assegna quello condiviso (se piano = shared)
5. Dispatch Job: CreateTenantAdminUser
   └── Crea l'utente Admin nel DB tenant
        con ruolo Admin e permessi completi
6. Pubblica job su coda Redis → Go invia OTP all'Admin
        │
        ▼
Admin riceve OTP via email → verifica → tenant attivo
```

### Gestione database dedicato

Il Job `CreateTenantMysqlUser` crea un utente MySQL dedicato per il tenant con permessi limitati al solo database del tenant. Le credenziali vengono salvate cifrate nel campo `data` (JSON) della tabella `tenants`. Se il database esiste già viene lanciata l'eccezione `DatabaseAlreadyExistsException`.

---

## 6. Microservizio Mail (Go) e Redis

### Architettura della coda

Laravel pubblica un job su Redis, il microservizio Go lo consuma e invia la mail. In caso di fallimento il messaggio viene rimesso in coda fino a un massimo di 3 tentativi, dopodiché finisce nella **dead letter queue** per gestione manuale.

```
Laravel                  Redis                    Go
   │                       │                       │
   │── LPUSH mail:queue ──►│                       │
   │   {                   │◄── BRPOP ─────────────│
   │     type: "otp",      │                       │
   │     email: "...",     │           invia mail  │
   │     payload: {...},   │                       │
   │     attempts: 0       │    OK  → done         │
   │   }                   │    FAIL→ attempts++   │
   │                       │         se < 3: LPUSH │
   │                       │         se >= 3: DLQ  │
```

### Tipi di job mail supportati

| type                      | Descrizione                                        |
| ------------------------- | -------------------------------------------------- |
| `otp_registration`        | OTP verifica email nuovo utente                    |
| `otp_tenant_registration` | OTP verifica email Admin dopo registrazione tenant |
| `otp_login`               | OTP per login senza password (Flusso A)            |
| `ticket_reply`            | Notifica nuova risposta su un ticket               |
| `tenant_cancelled`        | Conferma cancellazione tenant                      |

---

## 7. Schema Database

### DB Globale

```sql
global_identities
  id, name, email (UNIQUE), password,
  email_verified_at, created_at, updated_at, deleted_at

tenants
  id (string — sottodominio, PK), name,
  plan_id (FK→plans), data (JSON), created_at, updated_at

plans
  id, name, description, price_month,
  database_type (ENUM: shared|dedicated)

tenant_memberships
  global_user_id (FK), tenant_id (FK),
  state (ENUM: pending|accepted|banned)
  PK composita (global_user_id, tenant_id)

refresh_tokens
  id, global_identity_id (FK), token (64 char, UNIQUE),
  expires_at, revoked (bool)

domains
  id, domain (UNIQUE), tenant_id (FK)

otp_codes                          -- da implementare
  id, global_identity_id (FK), code (hash),
  expires_at, attempts, used (bool)
```

### DB Tenant

```sql
users
  id, global_user_id, role_id (FK→roles),
  tenant_id, created_at, updated_at, deleted_at

roles
  id, name (UNIQUE), description, tenant_id

permissions
  id, slug (UNIQUE), description, tenant_id

permission_role
  role_id (FK), permission_id (FK), tenant_id

teams
  id, name (UNIQUE), tenant_id

team_members
  team_id (FK), user_id (FK), team_role_id
  PK composita (team_id, user_id)

categories
  id, name, parent_category_id (FK→self, nullable), tenant_id

sla_policies
  id, name (UNIQUE), priority (int),
  response_time_hours, resolution_time_hours, tenant_id

tickets
  id, title, description,
  status (ENUM: open|in_progress|waiting|resolved|closed),
  priority (ENUM: low|medium|high|critical),
  user_id_author (FK), user_id_resolver (FK, nullable),
  team_id (FK, nullable), category_id (FK, nullable),
  closed_at (nullable), tenant_id

messages
  id, ticket_id (FK CASCADE), user_id (FK),
  body, is_internal (bool), macro_id (nullable), tenant_id

attachments                        -- da implementare
  id, message_id (FK), file_name, file_path (URL cloud)

macros                             -- da implementare
  id, team_id (nullable — null = globale),
  created_by (FK→users), title, content

ticket_history                     -- da implementare
  id, ticket_id (FK), changed_by (FK→users),
  field, old_value, new_value, created_at

notifications                      -- da implementare
  id, user_id (FK), type, payload (JSON), read_at (nullable)

notification_preferences           -- da implementare
  user_id (FK), channel (ENUM: app|email),
  event_type, enabled (bool)

user_settings                      -- da implementare
  user_id (FK), key, value
```

---

## 8. Struttura Route API

Le route sono definite in `routes/api_v1.php`. Il file `routes/api.php` contiene solo la rotta Sanctum legacy (non in uso attivo).

```php
// routes/api_v1.php

// Route pubbliche tenant
Route::middleware([InitializeTenancyByDomain::class, PreventAccessFromCentralDomains::class])
    ->prefix('v1')->group(function () {

        Route::post('/auth/login',   [AuthController::class, 'login']);
        Route::post('/auth/refresh', [AuthController::class, 'refresh']);

        // Route protette da JWT
        Route::middleware('jwt.auth')->group(function () {
            Route::get('/auth/me', [AuthController::class, 'me']);
        });
    });

// Route globali (nessun middleware tenancy)
Route::prefix('v1')->group(function () {
    Route::post('/register-tenant', [TenantRegistrationController::class, 'store']);
});
```

---

## 9. Convenzioni di Sviluppo

### Struttura risposta API

Tutte le risposte usano Laravel API Resources per una struttura consistente:

```json
// Successo
{
  "data": { ... }
}

// Errore
{
  "message": "Descrizione errore",
  "errors": { ... }
}
```

### Versioning API

Tutti gli endpoint sono sotto `/api/v1/`. Le versioni future useranno `/api/v2/` ecc. senza rompere la compatibilità con i client esistenti.

### Middleware personalizzati

| Middleware          | Scopo                                                       |
| ------------------- | ----------------------------------------------------------- |
| `JwtMiddleware`     | Valida JWT e verifica tenant_id                             |
| `ForceJsonResponse` | Forza `Content-Type: application/json` su tutte le risposte |

---

## 10. Stato Implementazione

| Feature                                 | Stato                      |
| --------------------------------------- | -------------------------- |
| Multi-tenancy ibrida (stancl/tenancy)   | ✅ Completata              |
| Autenticazione JWT (login, refresh, me) | ✅ Completata              |
| Registrazione tenant                    | ✅ Completata              |
| RBAC ruoli e permessi                   | ✅ Struttura DB completata |
| Verifica email OTP                      | 🔄 In sviluppo             |
| Microservizio Go + Redis                | 🔄 In sviluppo             |
| CRUD ticket e messaggi                  | 📋 Pianificato             |
| Gestione team e categorie               | 📋 Pianificato             |
| SLA collegata ai ticket                 | 📋 Pianificato             |
| Allegati su cloud storage               | 📋 Pianificato             |
| Notifiche in-app e email                | 📋 Pianificato             |
| Cronologia ticket                       | 📋 Pianificato             |
| Macros                                  | 📋 Pianificato             |

---

_Documento v1.1 — Progetto di Informatica, quinto anno_
