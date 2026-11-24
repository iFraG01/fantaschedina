# Regolamento FantaSchedina

Questo documento è la **fonte di verità** per le regole del gioco. Se una regola non è chiara o manca, si chiede e si aggiorna questo file: le regole non si inventano nel codice.

> Il gioco è **solo a punti**: nessun denaro reale, in nessuna forma.

---

## 0. Concetti di base

- **Lega**: gruppo privato di partecipanti che giocano insieme una stagione con una formula scelta alla creazione.
- **Giornata**: una giornata di Serie A, cioè l'insieme delle sue partite.
- **Schedina**: l'insieme delle giocate di un partecipante per una giornata, in una specifica lega.
- **Giocata**: il pronostico su una singola partita (es. "1", "Over 2.5", "Marcatore: X").
- **Quota congelata**: la quota della giocata fissata al momento del congelamento, che vale sia per la scelta sia per il calcolo dei punti.
- **Admin di lega**: il partecipante che ha creato la lega (o a cui è stato trasferito il ruolo).

---

## 1. Quote e dati

1. Fonte unica dei dati: **API-Football**.
2. Bookmaker di riferimento: **Bet365**, l'unico usato, per avere uno standard senza bias. *(Da confermare con la verifica della missione M07.)*
3. Le quote si **congelano 3 giorni prima del calcio d'inizio della prima partita della giornata** e restano invariate fino alla fine della giornata.
4. Se l'orario della prima partita cambia prima del congelamento, il momento del congelamento si ricalcola.
5. Se, al momento del congelamento, per una giocata **manca la quota**, quella giocata **non è disponibile** per quella partita.
6. Un errore tecnico (API non raggiungibile) non equivale a una quota mancante: il sistema riprova per un tempo limitato prima di considerare le quote assenti.

---

## 2. La schedina

### 2.1 Giocate disponibili
- È disponibile ogni giocata che **ha una quota congelata** ed è **valutabile dal sistema**, cioè per cui esiste una regola di valutazione e i dati necessari si trovano nelle statistiche di fine partita.
- Si parte da un **nucleo** e si estende nel tempo, per famiglie di giocate:
  - 1X2
  - Doppia chance (1X, X2, 12)
  - Under/Over (2.5 e le altre soglie quotate)
  - Goal/No Goal
  - Marcatore (il giocatore segna almeno un gol)
- Obiettivo di lungo periodo: rendere disponibili **tutte** le giocate quotate, per lasciare piena libertà di scelta.

### 2.2 Struttura
- **Una giocata per ogni partita** della giornata, come in una schedina tradizionale.
- Una partita lasciata senza giocata vale 0.

### 2.3 Apertura, chiusura e modifiche
- La schedina **si apre** quando le quote vengono congelate.
- La schedina **si chiude** al calcio d'inizio della **prima partita** della giornata, per tutte le partite insieme.
- Fino alla chiusura la schedina si può modificare liberamente.
- Se la prima partita viene anticipata, la chiusura si anticipa di conseguenza.

### 2.4 Punteggio
- Giocata vinta: si somma la **quota congelata** (es. 2,10 → 2,10 punti).
- Giocata persa: 0.
- Si contano **2 decimali**. Nessun tetto massimo per giocata.
- **Punteggio di giornata** = somma dei punti delle giocate + eventuali bonus (vedi capitolo 4).

### 2.5 Valutazione delle giocate
- Al termine di ogni partita il sistema scarica statistiche ed eventi e valuta automaticamente le giocate.
- Valutazione **provvisoria** subito dopo il fischio finale.
- Valutazione **definitiva** il mattino successivo, per recepire eventuali correzioni dei dati (es. gol riassegnato come autogol). Se i punti cambiano, la modifica viene registrata.

### 2.6 Casi particolari
- **Marcatore**:
  - contano solo i gol segnati nei **tempi regolamentari** (recupero compreso);
  - l'**autogol non conta**;
  - se il giocatore **non scende in campo**, la giocata vale comunque ed è **persa**.
- **Partita non giocata** entro la fine della giornata (rinviata, sospesa, annullata): la giocata vale **0**. Se la partita viene recuperata, l'admin può **convalidarla**: i punti si calcolano sul risultato reale con le quote congelate originali.

### 2.7 Visibilità
- Le schedine degli altri partecipanti restano **nascoste fino alla chiusura**, poi diventano visibili a tutti i membri della lega.

### 2.8 Schedina non inserita
- Una schedina non inserita vale **0**.
- Se concordato, l'admin può **inserire manualmente** la schedina di un partecipante. La schedina viene segnalata come "inserita dall'admin" e l'intervento viene registrato.

---

## 3. Formule di lega

La formula si sceglie **alla creazione della lega** e **non si può cambiare** a stagione iniziata.

### 3.1 Classifica a punteggio
- Vince chi accumula più punti nell'intera stagione.
- Spareggi: 1) numero totale di giocate vinte; 2) miglior punteggio in una singola giornata.

### 3.2 Scontri diretti
- A ogni giornata ogni partecipante affronta un avversario. Vittoria **3** punti, pareggio **1**, sconfitta **0**.
- **Conversione in gol a fasce**, stile fantacalcio, con parametri configurabili alla creazione:
  - soglia del primo gol (default **5,00** punti);
  - ampiezza della fascia (default **+1 gol ogni 3,00** punti).

  | Punteggio di giornata | Gol |
  |---|---|
  | 0 – 4,99 | 0 |
  | 5,00 – 7,99 | 1 |
  | 8,00 – 10,99 | 2 |
  | 11,00 – 13,99 | 3 |
  | … | … |

  Vince chi segna più gol; stessi gol = pareggio. *I default sono provvisori e si tarano con i dati reali.*
- **Calendario** generato automaticamente con girone all'italiana (tutti contro tutti), ripetuto per tutte le giornate della lega. L'ultimo ciclo può restare incompleto.
- **Partecipanti in numero dispari**: a ogni giornata uno riposa, a turno, e affronta l'**avversario fantasma**. Il punteggio del fantasma è la media dei punteggi di giornata degli altri partecipanti, convertita in gol con le stesse fasce.
- Spareggi in classifica: 1) punti; 2) scontri diretti tra i pari merito; 3) punteggio totale della stagione.

### 3.3 Formula 1
- A ogni giornata si stila la classifica per punteggio e si assegnano punti per posizione: **25-18-15-12-10-8-6-4-2-1**. Con meno di 10 partecipanti si usano solo le prime posizioni.
- **Parità in giornata**: stessa posizione e stessi punti (es. due primi a pari merito prendono 25 a testa e il successivo è terzo).
- Spareggi in classifica: 1) punti; 2) numero di primi posti; 3) punteggio totale della stagione.

### 3.4 Durata
- La lega copre le giornate da quella di partenza scelta dall'admin fino alla fine della stagione.

### 3.5 Non previsti (per ora)
- Playoff, coppe e competizioni secondarie: non nella prima versione, ma il sistema deve poterli aggiungere in futuro.

---

## 4. Asta, rose e bonus

### 4.1 Asta facoltativa
- Alla creazione della lega si sceglie se fare l'asta.
- **Senza asta non ci sono né rose né bonus**: la lega si gioca solo con le schedine.

### 4.2 Rose
- Ogni partecipante ha una rosa di **5 giocatori** di Serie A.
- **Ruoli liberi**: nessun vincolo di composizione.
- Ogni giocatore può appartenere a **un solo partecipante** della lega.

### 4.3 Svolgimento dell'asta (live, dentro l'app)
- Ogni partecipante ha **100 crediti**.
- Ordine di chiamata **casuale**, a turno. Chi ha la rosa completa viene saltato.
- Chi chiama propone un giocatore con un'offerta iniziale (minimo **1 credito**).
- Rilanci liberi con un **timer di 10 secondi** che riparte a ogni rilancio; allo scadere il giocatore va al miglior offerente.
- **Vincolo sui crediti**: non si può offrire una cifra che impedisca di completare la rosa (bisogna tenere almeno 1 credito per ogni posto ancora libero).
- L'asta termina quando tutte le rose sono complete. I crediti avanzati **vanno persi**.
- L'admin può **mettere in pausa** l'asta. Un partecipante disconnesso al proprio turno di chiamata viene saltato.

### 4.4 Giocatori che lasciano la Serie A
- Restano nella rosa ma **non possono più generare bonus**.
- Nessuna sostituzione nella prima versione.

### 4.5 Bonus
- Il bonus si aggiunge **solo se la giocata è vinta** e riguarda un **giocatore della propria rosa** (es. "Marcatore: X" con X in rosa).
- Il bonus è un **valore fisso per tipo di giocata**, come i bonus del fantacalcio.
- I valori saranno **tarati sulla frequenza reale degli eventi**: un evento comune (es. ammonizione) vale meno di uno raro (es. doppietta). La taratura si farà analizzando le statistiche delle partite salvate (missione M28).
- **Valori di prova** fino alla taratura:

  | Giocata | Bonus |
  |---|---|
  | Marcatore | +1,00 |
  | Primo o ultimo marcatore | +1,50 |
  | Doppietta | +2,00 |
  | Assist | +0,75 |
  | Ammonizione | +0,50 |

- Con la taratura si definirà un bonus per ogni giocata disponibile che riguarda un giocatore.

---

## 5. Gestione della lega

### 5.1 Creazione e admin
- Qualunque utente registrato può creare una lega e ne diventa **admin**.
- Un solo admin per lega; il ruolo si può **trasferire** a un altro membro.

### 5.2 Partecipanti
- Si entra con un **link o codice d'invito** generato dall'admin, revocabile in qualsiasi momento.
- Partecipanti: **da 2 a 20**.
- Le iscrizioni si **chiudono all'avvio della lega**.
- Chi abbandona a metà stagione resta come **inattivo**: le sue schedine valgono 0 e il calendario non cambia.

### 5.3 Più leghe
- Un utente può partecipare a più leghe e compila **una schedina separata per ogni lega**.

### 5.4 Account
- Registrazione con email e password, con **verifica dell'email**.
- Password salvate con hashing **Argon2id**.
- Autenticazione a due fattori (TOTP) prevista in una versione successiva.
- Dati personali minimi: email e nickname.

### 5.5 Piattaforma e notifiche
- Applicazione **web responsive**, utilizzabile comodamente da smartphone.
- Notifiche (promemoria di chiusura, risultati, inizio asta): non nella prima versione; il canale è da definire.

### 5.6 Registro degli interventi dell'admin
- Ogni intervento manuale dell'admin (punteggi modificati, partite convalidate, schedine inserite, trasferimenti di ruolo, revoche di inviti) viene registrato con **chi, cosa, quando e motivazione obbligatoria**.
- Il registro è **visibile a tutti i membri** della lega.

---

## 6. Parametri configurabili alla creazione della lega

Tutti i parametri sono **bloccati all'avvio della lega**.

| Parametro | Valori | Default |
|---|---|---|
| Formula | punteggio, scontri diretti, Formula 1 | — |
| Asta | sì / no | no |
| Soglia primo gol (scontri diretti) | punti | 5,00 |
| Ampiezza fascia (scontri diretti) | punti | 3,00 |
| Giornata di partenza | giornata di Serie A | la successiva |
| Tabella bonus (se asta) | punti per tipo di giocata | valori di prova |

---

## 7. Da tarare con i dati reali

- Default delle fasce (soglia del primo gol e ampiezza).
- Valori dei bonus per tipo di giocata.
- Verifica della tempistica di congelamento rispetto alla pubblicazione delle quote marcatore.

---

## 8. Idee per il futuro (non decise)

- Pulsante "copia schedina da un'altra lega".
- Sostituzione dei giocatori ceduti all'estero.
- Playoff e coppe.
- Notifiche (canale da definire).
- Nuove famiglie di giocate valutabili.
