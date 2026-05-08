# Sistema di Ticketing Multi-Tenant
## Documentazione di Progetto

**Progetto:** Sistema di Ticketing Multi-Tenant Ibrido
**Tipo documento:** Descrizione del Progetto
**Livello:** Elaborato di quinta superiore – Informatica
**Stack:** Laravel 12.0 · PHP 8.2+ · Go · MySQL · Redis

---

## 1. Introduzione e Obiettivi

Il progetto consiste nello sviluppo di un'applicazione web di **ticketing** (gestione delle richieste di supporto) progettata per servire **più aziende in modo simultaneo**, mantenendo i dati di ciascuna completamente separati e privati.

Il sistema nasce dall'esigenza reale di molte aziende di disporre di uno strumento professionale per gestire le richieste di assistenza dei propri clienti, smistando il lavoro tra team specializzati e garantendo tempi di risposta tracciabili.

### Obiettivi principali

- Consentire a più aziende di utilizzare la stessa piattaforma in modo **isolato e sicuro** (architettura multi-tenant ibrida).
- Gestire l'intero **ciclo di vita di una richiesta di supporto**, dalla creazione alla chiusura.
- Fornire strumenti di **monitoraggio e statistiche** per valutare le performance del supporto.
- Implementare un sistema di **ruoli e permessi** (RBAC) granulare, adatto a organizzazioni strutturate.

---

## 2. Stack Tecnologico

### Backend principale — Laravel 12

Il backend è sviluppato con **Laravel 12** (PHP 8.2+) ed espone API REST versioniate sotto `/api/v1/`. Il package **stancl/tenancy ^3.10** gestisce l'isolamento multi-tenant ibrido, instradando ogni richiesta al database corretto in base al sottodominio.

L'autenticazione è implementata con un sistema **JWT custom** tramite il package `firebase/php-jwt`, con tre livelli di token (descritto in dettaglio nella sezione 6).

### Microservizio Mail — Go + Redis

Un servizio indipendente scritto in **Go**, con unico scopo: inviare email transazionali in modo asincrono e affidabile.

La comunicazione tra Laravel e il microservizio avviene tramite **Redis** come coda di messaggi:

```
Laravel                  Redis (coda)              Microservizio Go
   │                          │                          │
   │── pubblica job ─────────►│                          │
   │   {email, tipo, payload} │◄──── consuma ────────────│
   │                          │                          │
   │                          │              tenta invio mail
   │                          │                          │
   │                          │     OK  → conferma consumo
   │                          │     FAIL→ rimette in coda
   │                          │           (max 3 retry)
   │                          │           poi → dead letter queue
```

La **dead letter queue** raccoglie i messaggi che hanno esaurito i tentativi, permettendo monitoraggio e gestione manuale degli errori senza perdita di dati.

Il microservizio viene invocato nei seguenti casi:

- Invio OTP per verifica email alla registrazione.
- Invio OTP per login senza password (flusso sottodominio sconosciuto).
- Notifiche email sugli aggiornamenti dei ticket.
- Email di conferma alla cancellazione di un tenant.

### Frontend — Lovable

Le interfacce utente sono sviluppate con **Lovable**, uno strumento per la generazione di UI moderne. Il frontend comunica con il backend esclusivamente tramite le API REST Laravel. Sono previste tre interfacce distinte:

- **Pannello Admin** — gestione utenti, team, categorie, SLA, statistiche.
- **Pannello Agente / Capo Team** — coda ticket, lavorazione, note interne.
- **Portale Cliente** — apertura ticket, storico, comunicazione con lo staff.

### Storage allegati — Cloud Storage

I file allegati ai messaggi dei ticket vengono salvati su un servizio di **Cloud Storage** (es. AWS S3 o compatibile). Il server applicativo non conserva file localmente; nel database viene salvato esclusivamente il riferimento al file (nome originale e URL nello storage).

### Riepilogo stack

| Componente | Tecnologia | Versione |
|---|---|---|
| Backend API | Laravel (PHP) | 12.0 / PHP 8.2+ |
| Multi-tenancy | stancl/tenancy | ^3.10 |
| Autenticazione | firebase/php-jwt | ^7.0 |
| Microservizio mail | Go | — |
| Coda messaggi | Redis | — |
| Database globale | MySQL | — |
| Database tenant | MySQL | — |
| Storage allegati | Cloud Storage (S3) | — |
| Frontend | Lovable | — |

---

## 3. Architettura Multi-Tenant Ibrida

### Che cos'è il Multi-Tenancy?

Il termine **multi-tenant** indica un'architettura software in cui una singola istanza dell'applicazione serve più clienti (detti *tenant*, cioè "inquilini"). Ogni azienda che si iscrive alla piattaforma ottiene il proprio ambiente isolato, come se avesse un'applicazione tutta sua, pur condividendo la stessa infrastruttura.

### Perché "Ibrida"?

Il progetto adotta un modello **ibrido** perché combina due approcci:

| Componente | Approccio |
|---|---|
| Identità utenti e aziende | Database globale condiviso |
| Dati operativi di ogni azienda | Database dedicato per tenant |

Questa scelta bilancia efficienza economica con sicurezza e isolamento dei dati.

### Schema dell'architettura

```
┌──────────────────────────────────────────────────────────────┐
│                      PIATTAFORMA GLOBALE                     │
│         (DB_GLOBAL: identità, tenant, abbonamenti)           │
│                                                              │
│  ┌─────────────────┐    Redis    ┌────────────────────────┐  │
│  │  Laravel API    │────────────►│  Microservizio Mail Go │  │
│  │  /api/v1/       │             └────────────────────────┘  │
│  └────────┬────────┘                                         │
└───────────┼──────────────────────────────────────────────────┘
            │ routing per sottodominio (stancl/tenancy)
    ┌────────┴─────────┐
    ▼                  ▼
┌──────────────┐   ┌──────────────┐
│  DB Azienda  │   │  DB Azienda  │  ...
│   "alpha"    │   │   "beta"     │
│  (tickets,   │   │  (tickets,   │
│   team...)   │   │   team...)   │
└──────────────┘   └──────────────┘
```

### Routing per sottodominio

Il package stancl/tenancy identifica il tenant attivo dalla richiesta HTTP. Una richiesta a `alpha.piattaforma.com/api/v1/auth/login` viene automaticamente instradata al database dell'azienda "alpha". Il middleware `InitializeTenancyByDomain` si occupa di questo instradamento su ogni richiesta tenant.

---

## 4. Attori del Sistema e Ruoli

Il sistema adotta un modello **RBAC (Role-Based Access Control)**: ogni utente ha un ruolo, ogni ruolo ha un insieme di permessi identificati da slug (es. `tickets.view`, `tickets.create`). Il metodo `User::hasPermission(slug)` viene usato dal backend per verificare i permessi su ogni operazione.

### 4.1 Amministratore (Admin)

Responsabile dell'intero ambiente aziendale. Permessi completi su tutte le risorse del tenant.

- Approva, rifiuta o sospende gli accessi degli utenti.
- Crea team, categorie, SLA policy.
- Crea macro globali disponibili a tutti gli agenti.
- Accede alle statistiche globali dell'azienda.

### 4.2 Capo Team

Supervisore di uno specifico gruppo di agenti.

- Monitora e riassegna i ticket del suo team.
- Crea macro di team visibili solo al suo gruppo.
- Accede alle statistiche del suo team.
- Ha anche tutte le capacità operative di un agente normale.

### 4.3 Agente

Operatore che risolve le richieste.

- Gestisce la coda ticket del suo team.
- Prende in carico ticket diventandone il "proprietario".
- Comunica con il cliente via chat pubblica e scrive note interne.

### 4.4 Utente Finale (Cliente)

Chi richiede assistenza.

- Apre nuove richieste di supporto.
- Vede e risponde esclusivamente ai propri ticket.

---

## 5. Flussi Principali

### 5.1 Registrazione Tenant (Azienda)

Il flusso di registrazione di una nuova azienda sulla piattaforma.

```
[1] Form registrazione      [2] Verifica               [3] Creazione
    compilato dall'Admin ──►    sottodominio        ──►    tenant nel
    futuro:                     disponibile?               DB globale.
    - Nome azienda              No → errore
    - Sottodominio              Sì → procedi
    - Piano (shared/dedicated)
    - Email Admin
    - Password Admin
                                                               │
                                                               ▼
[6] Admin inserisce     [5] Microservizio Go        [4] Creazione DB tenant
    OTP → account   ◄──    invia OTP via mail  ◄──     + utente Admin
    verificato →           all'Admin.                  nel DB tenant.
    tenant attivo.         (6 cifre, 10 min)
```

Il sistema crea in sequenza: il record tenant nel DB globale, il database dedicato (o condiviso in base al piano) tramite il job `CreateTenantMysqlUser`, e infine l'utente Admin nel nuovo DB tenant tramite il job `CreateTenantAdminUser`. Solo dopo la verifica OTP il tenant risulta pienamente attivo.

> **Nota:** In una versione futura è previsto un processo di approvazione manuale o verifica del pagamento prima dell'attivazione del tenant.

### 5.2 Registrazione e Verifica Email (Utente)

```
[1] Compilazione form        [2] Microservizio Go        [3] Utente inserisce
    registrazione       ──►      invia OTP via mail  ──►     il codice OTP
    (email, password,           (6 cifre, 10 min,           sulla pagina
    nome)                        max 3 tentativi)            di verifica.
                                                                  │
                                                                  ▼
                                                         [4] Account attivo.
                                                             Utente può
                                                             accedere.
```

L'OTP viene salvato nel DB globale in forma **hashata** con timestamp di creazione, flag `used` e contatore dei tentativi errati. Dopo 3 tentativi sbagliati l'endpoint viene bloccato temporaneamente.

### 5.3 Login — Flusso A (Sottodominio Sconosciuto)

L'utente non ricorda l'indirizzo della sua azienda e accede dalla pagina principale.

```
[1] Pagina principale    [2] Inserisce solo       [3] Go invia OTP
    piattaforma.             la propria email.         via mail.
                                                        (6 cifre, 10 min)
                                                             │
                                                             ▼
[5] Dashboard Globale:   [4] Utente inserisce
    lista aziende con        il codice OTP
    membership attiva.       correttamente.
         │
         ▼
[6] Seleziona azienda ──► [7] Sistema emette i tre token.
                              Accesso all'ambiente tenant.
```

L'OTP sostituisce completamente la password in questo flusso: rappresenta esso stesso la prova di identità.

### 5.4 Login — Flusso B (Sottodominio Noto)

L'utente conosce l'indirizzo diretto della propria azienda (es. `azienda.piattaforma.com`).

```
[1] Accesso diretto al      [2] Inserisce email      [3] Verifica credenziali.
    sottodominio aziendale.     e password.
                                                             │
                                                             ▼
                                                    [4] Sistema emette
                                                        i tre token.
                                                        Accesso al tenant.
```

### 5.5 Ciclo di Vita del Ticket

```
[1] CREAZIONE          [2] SMISTAMENTO         [3] LAVORAZIONE        [4] CHIUSURA
Il cliente apre   ──►  Categoria scelta?   ──►  Agente prende     ──►  Problema risolto,
il ticket con          Sì → coda del team       in carico il           ticket chiuso.
titolo, descriz.       No  → Triage             ticket. Chat           Cronologia
e categoria.           (assegnazione            pubblica +             immutabile
SLA applicata.         manuale)                 note interne.          salvata.
```

**Stati del ticket:**

| Stato | Descrizione |
|---|---|
| `open` | Appena creato, in attesa di presa in carico |
| `in_progress` | Un agente lo sta lavorando |
| `waiting` | In attesa di risposta dal cliente |
| `resolved` | Problema risolto |
| `closed` | Chiuso definitivamente |

### 5.6 Il Triage

Quando un ticket arriva senza categoria o team associato entra in una coda **Triage**, dove agenti o capi team lo analizzano e lo smistano manualmente.

---

## 6. Autenticazione e Gestione Sessioni

Il sistema usa **JWT custom** (`firebase/php-jwt`) con tre livelli di token, tutti trasportati via **cookie HttpOnly + Secure + SameSite=Strict** per prevenire attacchi XSS e CSRF.

### I tre token

**Identity Token**
- Durata: **15 minuti**.
- Payload: `type: identity`, `sub` (user id), `email`.
- Usato esclusivamente nel flusso OTP (Flusso A): viene emesso dopo la verifica del codice e permette all'utente di selezionare l'azienda senza inserire la password.

**Access Token**
- Durata: **1 ora**.
- Payload: `type: access`, `sub`, `tenant_id`, `role_id`.
- Autorizza tutte le operazioni all'interno del tenant. Il `tenant_id` viene verificato dal middleware `JwtMiddleware` su ogni richiesta per prevenire attacchi cross-tenant (un utente non può operare in un tenant che non gli appartiene).

**Refresh Token**
- Durata: **7 giorni**.
- Stringa da 64 caratteri, salvata nel DB globale come hash SHA-256.
- Permette di ottenere un nuovo Access Token senza ripetere il login.
- È il punto di controllo per il **ban**: revocare il refresh token significa che entro al massimo 1 ora (scadenza dell'access token corrente) l'utente perde definitivamente l'accesso.

### Nota sul tenant_id

Poiché un utente può essere membro di più aziende, ogni volta che seleziona un'azienda dalla Dashboard Globale il sistema emette un **nuovo Access Token** con il `tenant_id` specifico di quell'ambiente. Il Refresh Token rimane unico e globale.

### RBAC — Controllo permessi

Ogni ruolo ha un insieme di permessi identificati da **slug** (es. `tickets.view`, `tickets.create`, `team.manage`). Il metodo `User::hasPermission(slug)` viene invocato dal backend prima di ogni operazione sensibile.

---

## 7. Endpoint API (stato attuale)

Tutti gli endpoint sono sotto `/api/v1/`. Il routing tenant è gestito dal middleware `InitializeTenancyByDomain`.

| Metodo | Path | Descrizione | Auth |
|---|---|---|---|
| POST | `/api/v1/register-tenant` | Registrazione nuova azienda | Pubblica |
| POST | `/api/v1/auth/login` | Login con email e password | Pubblica (tenant) |
| POST | `/api/v1/auth/refresh` | Rinnovo access token | Pubblica (tenant) |
| GET | `/api/v1/auth/me` | Dati utente corrente | JWT |

Gli endpoint per ticket, messaggi, team, categorie e SLA sono pianificati nelle fasi successive dello sviluppo.

---

## 8. Sistema di Notifiche

Le notifiche sono gestite su due canali, configurabili dall'utente nelle impostazioni:

| Canale | Descrizione |
|---|---|
| **In-app** | Notifica visibile nella piattaforma, salvata nel DB tenant |
| **Email** | Inviata tramite microservizio Go via coda Redis |

**Eventi che generano notifiche:**
- Agente risponde a un ticket → notifica al cliente.
- Cliente risponde → notifica all'agente assegnato.
- Ticket riassegnato → notifica ai soggetti coinvolti.
- Ticket chiuso → notifica al cliente.

---

## 9. Ciclo di Vita del Tenant e Abbonamenti

### Piani di Abbonamento

| Tipo Database | Descrizione |
|---|---|
| **Shared** | Il tenant condivide un database con altri (economico, volumi bassi) |
| **Dedicated** | Il tenant ha un database fisicamente separato (massima sicurezza) |

### Mancato Rinnovo e Cancellazione

```
Mancato rinnovo / richiesta cancellazione
              │
              ▼
    Soft Delete attivato sul tenant
    (dati preservati, accesso sospeso)
              │
              ▼
    Email di conferma inviata all'Admin
    tramite microservizio Go
              │
              ▼
    Trascorso il periodo di grazia:
    eliminazione definitiva dei dati
    e del database dedicato.
```

---

## 10. Schema Logico del Database

### DB Globale

| Tabella | Campi principali | Note |
|---|---|---|
| `global_identities` | id, name, email, password, email_verified_at, deleted_at | Identità globale utente — SoftDeletes |
| `tenants` | id (subdomain), name, plan_id, data (json) | id è il sottodominio stesso |
| `plans` | id, name, price_month, database_type | enum: shared / dedicated |
| `tenant_memberships` | global_user_id, tenant_id, state | enum: pending / accepted / banned |
| `refresh_tokens` | global_identity_id, token (hash SHA-256), expires_at, revoked | Durata 7 giorni |
| `domains` | domain, tenant_id | Gestita da stancl/tenancy |
| `otp_codes` | user_id, code (hash), expires_at, attempts, used | Da implementare |

### DB Tenant (per ogni azienda)

| Tabella | Campi principali | Note |
|---|---|---|
| `users` | id, global_user_id, role_id, deleted_at | Profilo locale nel tenant — SoftDeletes |
| `roles` | id, name, description | RBAC |
| `permissions` | id, slug, description | es. `tickets.view` |
| `permission_role` | role_id, permission_id | Pivot M2M |
| `teams` | id, name | — |
| `team_members` | team_id, user_id, team_role_id | PK composita |
| `categories` | id, name, parent_category_id | Supporta gerarchie (self-join) |
| `sla_policies` | id, name, priority, response_time_hours, resolution_time_hours | — |
| `tickets` | id, title, description, status, priority, user_id_author, user_id_resolver, team_id, category_id, closed_at | — |
| `messages` | id, ticket_id, user_id, body, is_internal, macro_id | is_internal: note visibili solo allo staff |
| `attachments` | id, message_id, file_name, file_path | path su cloud storage — Da implementare |
| `macros` | id, team_id (null = globale), created_by, title, content | Da implementare |
| `ticket_history` | id, ticket_id, changed_by, field, old_value, new_value | Cronologia immutabile — Da implementare |
| `notifications` | id, user_id, type, payload, read_at | Da implementare |
| `notification_preferences` | user_id, channel, event_type, enabled | Da implementare |
| `user_settings` | user_id, key, value | Preferenze generali — Da implementare |

---

## 11. Considerazioni di Sicurezza

- Le **password** e i **codici OTP** vengono salvati esclusivamente come hash, mai in chiaro.
- I **Refresh Token** sono hashati con SHA-256 nel database.
- I **cookie** di sessione sono HttpOnly + Secure + SameSite=Strict, prevenendo XSS e CSRF.
- Il **tenant_id** nel JWT viene verificato su ogni richiesta per prevenire attacchi cross-tenant.
- Il **soft delete** (`deleted_at`) preserva i dati storici senza cancellazioni fisiche accidentali.
- Il **rate limiting** sull'OTP (max 3 tentativi) previene attacchi brute-force.
- Il **ban** si applica revocando il refresh token: l'utente perde l'accesso entro 1 ora senza possibilità di rinnovo.
- Gli **URL degli allegati** su cloud storage possono essere firmati con scadenza, impedendo accessi non autorizzati.
- La **dead letter queue** Redis garantisce che nessuna email transazionale venga persa silenziosamente in caso di errore del microservizio Go.

---

## 12. Stato di Sviluppo

| Area | Stato |
|---|---|
| Architettura multi-tenant ibrida | ✅ Completata |
| Autenticazione JWT (login, refresh, me) | ✅ Completata |
| Registrazione tenant | ✅ Completata |
| Verifica email OTP + microservizio Go | 🔄 In sviluppo |
| CRUD ticket e messaggi | 📋 Pianificato |
| Gestione team, categorie, SLA via API | 📋 Pianificato |
| Sistema notifiche (in-app + email) | 📋 Pianificato |
| Allegati su cloud storage | 📋 Pianificato |
| Cronologia ticket (audit trail) | 📋 Pianificato |
| Frontend Admin (Lovable) | 🔄 In sviluppo |
| Frontend Agente (Lovable) | 📋 Pianificato |
| Frontend Cliente (Lovable) | 📋 Pianificato |

---

*Documento v1.1 — Progetto di Informatica, quinto anno*
