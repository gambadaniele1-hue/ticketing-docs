# 🎫 Ticketing
### Sistema di Ticketing Multi-Tenant Ibrido

> Piattaforma SaaS per la gestione del supporto clienti, progettata per servire più aziende in modo simultaneo con isolamento completo dei dati.

---

## 📌 Panoramica

**Ticketing** è un'applicazione web che permette alle aziende di gestire le richieste di assistenza dei propri clienti attraverso un sistema di ticket strutturato, con team dedicati, priorità, SLA e comunicazione integrata.

Ogni azienda opera in un ambiente completamente isolato (tenant), mantenendo i propri dati separati da quelli di tutti gli altri clienti della piattaforma.

---

## ✨ Funzionalità principali

- 🏢 **Multi-tenancy ibrida** — ogni azienda ha il proprio ambiente e database dedicato
- 🔐 **Autenticazione JWT** a tre livelli con cookie sicuri (HttpOnly, SameSite=Strict)
- 👥 **Ruoli e permessi RBAC** — Admin, Capo Team, Agente, Cliente
- 🎫 **Ciclo di vita completo del ticket** — dalla creazione alla chiusura con cronologia immutabile
- 💬 **Chat interna al ticket** con supporto a note private visibili solo allo staff
- ⏱️ **SLA Policy** — timer di risposta e risoluzione per priorità
- 📋 **Triage** — smistamento manuale dei ticket senza categoria
- 🔔 **Notifiche** in-app e via email configurabili per evento
- 📎 **Allegati** su Cloud Storage
- ⚡ **Microservizio mail in Go** con coda Redis e gestione dei fallimenti

---

## 🛠️ Stack tecnologico

| Layer | Tecnologia |
|---|---|
| Backend API | Laravel 12 (PHP 8.2+) |
| Multi-tenancy | stancl/tenancy ^3.10 |
| Autenticazione | JWT custom (firebase/php-jwt) |
| Microservizio mail | Go |
| Coda messaggi | Redis |
| Database | MySQL |
| Storage allegati | Cloud Storage (AWS S3) |
| Frontend | Lovable |

---

## 📁 Repository del progetto

| Repository | Descrizione |
|---|---|
| [`ticketing-api`](https://github.com/gambadaniele1-hue/ticketing-api) | Backend Laravel — API REST |
| [`ticketing-mail`](https://github.com/gambadaniele1-hue/ticketing-mail) | Microservizio Go per l'invio email |
| [`ticketing-app`](https://github.com/gambadaniele1-hue/ticketing-app) | Frontend Lovable |
| [`ticketing-docs`](https://github.com/gambadaniele1-hue/ticketing-docs) | ← Sei qui — Documentazione |

---

## 📚 Documentazione

| Documento | Descrizione |
|---|---|
| [📋 Descrizione del Progetto](./01-progetto.md) | Architettura, flussi, attori, stack tecnologico |
| [🔧 Documentazione Tecnica](./02-tecnica.md) | Schema ER, endpoint API, struttura del codice *(in arrivo)* |
| [👤 Guida Utente](./03-utente.md) | Come usare la piattaforma *(in arrivo)* |

---

## 🏗️ Architettura in sintesi

```
┌──────────────────────────────────────────────────────┐
│                  PIATTAFORMA GLOBALE                 │
│                                                      │
│   ┌─────────────┐   Redis   ┌──────────────────────┐ │
│   │ Laravel API │──────────►│  Microservizio Go    │ │
│   └──────┬──────┘           └──────────────────────┘ │
└──────────┼───────────────────────────────────────────┘
           │ routing per sottodominio
   ┌───────┴────────┐
   ▼                ▼
┌──────────────┐  ┌──────────────┐
│  DB Azienda  │  │  DB Azienda  │  ...
│   "alpha"    │  │   "beta"     │
└──────────────┘  └──────────────┘
```

---

## 👤 Autore

Progetto realizzato come elaborato di quinta superiore — Informatica.

---

*Documentazione v1.0*
