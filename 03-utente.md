# Ticketing — Guida Utente

## Come usare la piattaforma

**Versione:** 1.1
**Ultimo aggiornamento:** 2025

> **Nota sullo stato del progetto:** Alcune funzionalità descritte in questa guida sono implementate e pienamente funzionanti. Altre sono disponibili nell'interfaccia grafica come anteprima delle funzionalità future, ma non ancora collegate al backend. Le sezioni interessate sono indicate con il badge **[UI dimostrativa]**.

---

## 1. Introduzione

**Ticketing** è una piattaforma web per la gestione delle richieste di assistenza. Permette ai clienti di aprire ticket di supporto e allo staff di gestirli in modo organizzato, tracciabile e rapido.

Questa guida è rivolta a tutti gli utenti della piattaforma: clienti, agenti, capi team e amministratori.

---

## 2. Primo Accesso

### 2.1 Registrazione azienda

Se la tua azienda non è ancora presente sulla piattaforma, è necessario registrarla prima di tutto.

1. Vai sulla pagina principale della piattaforma.
2. Clicca su **Registra la tua azienda**.
3. Scegli il piano (Shared o Dedicated).
4. Compila il form con:
   - Nome dell'azienda
   - Descrizione (opzionale)
   - Sottodominio desiderato (es. `nomeazienda` → `nomeazienda.ticketing.com`)
   - Email e password del primo amministratore
5. Clicca su **Registra**.
6. Riceverai un codice OTP via email: inseriscilo nella pagina di verifica.
7. La tua azienda è attiva. Puoi effettuare il login.

> **Nota:** Il sottodominio scelto non può essere modificato in seguito. Sceglilo con cura.

### 2.2 Registrazione utente

Se l'azienda esiste già e devi unirti come nuovo utente:

1. Vai direttamente sul sottodominio dell'azienda (es. `nomeazienda.ticketing.com`).
2. Clicca su **Registrati**.
3. Inserisci nome, email e password.
4. Riceverai conferma che la tua richiesta è in attesa di approvazione.
5. L'amministratore dell'azienda riceverà una notifica e dovrà approvare il tuo accesso.

---

## 3. Accesso alla Piattaforma

### 3.1 Conosco l'indirizzo della mia azienda

1. Vai direttamente su `nomeazienda.ticketing.com`.
2. Vedrai la scheda della tua azienda con nome e descrizione.
3. Inserisci email e password.
4. Accedi alla piattaforma.

### 3.2 Non ricordo l'indirizzo della mia azienda

1. Vai sulla pagina principale della piattaforma.
2. Clicca su **Trova il tuo spazio di lavoro**.
3. Inserisci la tua email.
4. Riceverai un codice OTP via email: inseriscilo.
5. Verrà mostrata la lista di tutte le aziende a cui hai accesso.
6. Clicca su **Entra** per accedere all'azienda desiderata.

### 3.3 Accesso in attesa di approvazione

Se dopo il login vedi il messaggio "Il tuo accesso è in attesa di approvazione", significa che l'amministratore non ha ancora approvato la tua richiesta. Contatta direttamente l'azienda se il tempo si prolunga.

---

## 4. Guida per il Cliente **[UI dimostrativa]**

> Le funzionalità descritte in questa sezione sono visibili nell'interfaccia ma non ancora collegate al backend. Verranno rese operative in una versione futura.

Il cliente è chi apre le richieste di assistenza.

### 4.1 Aprire un nuovo ticket

1. Dalla tua dashboard clicca su **Nuovo Ticket**.
2. Compila i campi:
   - **Titolo** — descrivi brevemente il problema
   - **Descrizione** — spiega il problema in dettaglio
   - **Categoria** — scegli quella più adatta
   - **Priorità** — indica l'urgenza
3. Allega eventuali file utili.
4. Clicca su **Invia**.

### 4.2 Seguire un ticket

Dalla tua dashboard puoi vedere tutti i tuoi ticket con il relativo stato:

| Stato              | Significato                         |
| ------------------ | ----------------------------------- |
| **Aperto**         | In attesa di essere preso in carico |
| **In lavorazione** | Un agente ci sta lavorando          |
| **In attesa**      | Lo staff aspetta una tua risposta   |
| **Risolto**        | Il problema è stato risolto         |
| **Chiuso**         | Ticket chiuso definitivamente       |

### 4.3 Rispondere a un ticket

1. Clicca sul ticket dalla tua dashboard.
2. Leggi i messaggi dello staff nella chat.
3. Scrivi la tua risposta e clicca **Invia**.

### 4.4 Chiudere un ticket

Quando il tuo problema è risolto puoi chiudere il ticket:

1. Apri il ticket.
2. Clicca su **Segna come risolto**.

---

## 5. Guida per l'Agente **[UI dimostrativa]**

> Le funzionalità descritte in questa sezione sono visibili nell'interfaccia ma non ancora collegate al backend. Verranno rese operative in una versione futura.

L'agente è il membro dello staff che risolve materialmente i ticket.

### 5.1 La coda ticket

Dalla tua dashboard vedi i ticket del tenant con i loro stati e priorità.

### 5.2 Prendere in carico un ticket

1. Dalla coda, clicca su un ticket.
2. Clicca su **Prendi in carico**.
3. Il ticket ti viene assegnato e lo stato diventa **In lavorazione**.

### 5.3 Rispondere al cliente

1. Apri il ticket.
2. Scrivi il messaggio nel campo di testo.
3. Clicca **Invia** per mandare un messaggio visibile al cliente.

### 5.4 Scrivere una nota interna

1. Nel campo di testo attiva l'opzione **Nota interna**.
2. Scrivi il messaggio e clicca **Invia**.
3. La nota è visibile solo allo staff.

### 5.5 Usare le macro **[UI dimostrativa]**

1. Nel campo di testo clicca su **Inserisci macro**.
2. Scegli dalla lista.
3. Il testo viene inserito automaticamente.

---

## 6. Guida per il Capo Team **[UI dimostrativa]**

> Le funzionalità descritte in questa sezione sono visibili nell'interfaccia ma non ancora collegate al backend. Verranno rese operative in una versione futura.

Il capo team supervisiona il lavoro del suo gruppo e ha tutte le capacità di un agente normale.

### 6.1 Riassegnare un ticket

1. Apri il ticket.
2. Clicca su **Riassegna**.
3. Scegli l'agente dalla lista dei membri del tuo team.

### 6.2 Creare macro di team **[UI dimostrativa]**

1. Vai su **Impostazioni → Macro**.
2. Clicca su **Nuova Macro**.
3. Inserisci titolo e testo.

---

## 7. Guida per l'Amministratore

L'amministratore gestisce l'intero ambiente aziendale sulla piattaforma.

### 7.1 Gestione utenti ✅

Dalla sezione **Utenti** puoi:

- Vedere tutti gli utenti che hanno richiesto accesso.
- **Approvare** gli utenti in attesa.
- **Rifiutare** l'accesso a un utente.
- **Sospendere** un utente accettato.
- **Riattivare** un utente sospeso.
- **Cambiare il ruolo** di un utente (Admin, Agent, Team Lead, Customer).

### 7.2 Gestione team ✅

1. Vai su **Team**.
2. Clicca su **Nuovo Team** e assegnagli un nome.
3. Clicca su un team per aprire il pannello di gestione:
   - Aggiungi membri scegliendo tra gli utenti disponibili
   - Assegna il ruolo nel team (Agent o Team Lead)
   - Rimuovi membri

> **Nota:** Se rimuovi l'unico Team Lead del team, il sistema ti avviserà prima di procedere.

### 7.3 Gestione categorie ✅

1. Vai su **Categorie**.
2. Crea, modifica ed elimina categorie.
3. Puoi creare sottocategorie selezionando una categoria padre.

### 7.4 Gestione SLA ✅

1. Vai su **SLA**.
2. Crea policy con nome, priorità, ore massime per la prima risposta e per la risoluzione.
3. Modifica o elimina le policy esistenti.

### 7.5 Macro ✅

Dalla sezione **Macro** puoi visualizzare le risposte preimpostate disponibili per i tuoi agenti. Le macro globali sono visibili a tutto lo staff.

> La creazione di nuove macro dall'interfaccia è pianificata per una versione futura.

### 7.6 Dashboard ✅

Dalla dashboard principale puoi vedere i numeri chiave del supporto: ticket aperti, in lavorazione, in attesa e chiusi.

---

## 8. Notifiche **[UI dimostrativa]**

> Il sistema di notifiche in-app è pianificato per una versione futura. Le notifiche email per gli eventi principali (nuovi utenti pending, OTP) sono già attive.

---

## 9. Domande Frequenti

**Non ricevo l'OTP via email. Cosa faccio?**
Controlla la cartella spam. Se non arriva entro qualche minuto, prova a richiederne uno nuovo dalla pagina di verifica.

**Non ricordo il sottodominio della mia azienda. Cosa faccio?**
Dalla pagina principale clicca su "Trova il tuo spazio di lavoro", inserisci la tua email e riceverai un codice OTP per accedere alla lista delle tue aziende.

**Non vedo il ticket che ho aperto. Dove si trova?**
La gestione ticket è in fase di sviluppo. Sarà disponibile in una versione futura.

**Posso cambiare il mio ruolo?**
No, solo l'amministratore può modificare i ruoli degli utenti dalla sezione Utenti.

**Posso riaprire un ticket chiuso?**
La gestione ticket è in fase di sviluppo. Sarà disponibile in una versione futura.

---

_Documento v1.1 — Progetto di Informatica, quinto anno_
