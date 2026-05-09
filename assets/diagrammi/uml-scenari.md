# Ticketing — Scenari UML

## Scenari dei Casi d'Uso

**Versione:** 1.0
**Ultimo aggiornamento:** 2025

---

## 1. Login

### Scenario 1.1 — Login con credenziali corrette

**Pre-condizioni**

- L'utente si trova nel sottodominio corretto del tenant.
- L'utente ha una membership attiva in quel tenant (stato `accepted`).
- L'email dell'utente è stata verificata (`email_verified_at` non è null).
- L'utente inserisce credenziali corrette.

**Scenario**

1. Il sistema valida il formato delle credenziali inserite.
2. Il sistema verifica l'esistenza della `global_identity` tramite email.
3. Il sistema verifica che l'hash della password coincida.
4. Il sistema verifica che la membership dell'utente nel tenant sia attiva.
5. Il sistema genera e rilascia un **Refresh Token** nei cookie (HttpOnly, durata 7 giorni).
6. Il sistema genera e rilascia un **Access Token** nei cookie (HttpOnly, durata 1 ora).

**Post-condizioni**

- L'utente accede al proprio pannello.
- L'utente possiede i token nei cookie con le date di scadenza corrette.

---

### Scenario 1.2 — Login con credenziali errate

**Pre-condizioni**

- L'utente si trova nel sottodominio corretto del tenant.
- L'utente inserisce credenziali non corrette (email inesistente, password errata, membership non attiva o email non verificata).

**Scenario**

1. Il sistema valida il formato delle credenziali inserite.
2. Il sistema cerca la `global_identity` tramite email.
   - Se l'email non esiste → restituisce errore generico e termina.
3. Il sistema verifica che l'hash della password coincida.
   - Se la password non corrisponde → restituisce errore generico e termina.
4. Il sistema verifica che l'email sia stata verificata.
   - Se `email_verified_at` è null → restituisce errore specifico e termina.
5. Il sistema verifica che la membership nel tenant sia attiva.
   - Se lo stato è `pending` → restituisce errore che informa l'utente che l'accesso è in attesa di approvazione e termina.
   - Se lo stato è `banned` → restituisce errore che informa l'utente che l'accesso è stato revocato e termina.

**Post-condizioni**

- L'utente non accede al pannello.
- Nessun token viene generato o rilasciato.
- Il sistema restituisce un messaggio di errore appropriato al caso.

> **Nota di sicurezza:** Per gli errori di email e password il sistema restituisce sempre un messaggio generico (es. "Credenziali non valide") senza specificare quale dei due campi è errato, per prevenire attacchi di enumerazione utenti.

---
