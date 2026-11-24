# Piano del progetto FantaSchedina

Percorso per imparare il DevOps da zero costruendo un'applicazione reale. Questo documento descrive il **come** (architettura, fasi, missioni); le regole del gioco sono in `REGOLAMENTO.md`.

---

## 1. Contesto e principi

### Contesto
- **Obiettivo**: imparare tutti i concetti fondamentali del DevOps, per curiosità, con un progetto completo.
- **Ritmo**: libero, senza scadenze. Il percorso è diviso in **missioni** autonome, ognuna con un criterio di "fatto".
- **Ambiente**: tutto in locale su MacBook Air M5 (Apple Silicon, 24 GB di RAM). Il passaggio al cloud è un'opzione futura.
- **Livello di partenza**: da zero. Ogni concetto si vede prima in teoria (perché), poi in pratica (come).
- **Divisione dei compiti**: Claude scrive il codice dell'applicazione e lo spiega; l'utente svolge le attività DevOps guidato da Claude.

### Principi
1. **Tutto come codice**: niente configurazioni fatte a mano "una volta sola".
2. **Iterazioni piccole**: ogni missione si chiude con qualcosa che funziona e un commit pulito.
3. **Documentare le scelte**, non solo il risultato: gli ADR spiegano il perché.
4. **Distruggi e ricrea**: se non si ricostruisce tutto da zero con pochi comandi, non è finito.
5. **Sicurezza fin dall'inizio**, non aggiunta alla fine.
6. **Il diario**: a fine sessione si annota in `DIARIO.md` cosa è stato fatto, cosa si è capito e cosa resta aperto.

---

## 2. Stack e strumenti

### Applicazione

| Componente | Tecnologia | Ruolo |
|---|---|---|
| Backend | Laravel + Sanctum | API REST, logica di gioco, autenticazione SPA via cookie di sessione |
| Frontend | Vue 3 + Vite, Vue Router, Pinia | Interfaccia web responsive (SPA) |
| Database | PostgreSQL | Dati di leghe, utenti, partite, quote, schedine, punteggi |
| Redis | Redis | Cache, coda dei job, broadcast in tempo reale |
| Worker | Laravel Queues (eventualmente Horizon) | Esecuzione dei job in background |
| Scheduler | Laravel Scheduler | Avvio dei job pianificati (sincronizzazioni, congelamento, valutazioni) |
| Tempo reale | Laravel Reverb + Echo | Asta live (v4) |
| Qualità | Pest/PHPUnit, PHPStan (Larastan), Pint, ESLint, Vitest | Test e analisi statica |

### Strumenti DevOps

| Area | Strumenti |
|---|---|
| Base | Homebrew, zsh, Git, GitHub (repo privato), VS Code |
| Container e VM | OrbStack (Docker + VM Linux) |
| CI | GitHub Actions, GitHub Container Registry |
| Sicurezza della supply chain | Gitleaks, Semgrep, Trivy, Dependabot, Cosign |
| Infrastructure as Code | Ansible, Terraform, LocalStack |
| Kubernetes | k3d, kubectl, Helm |
| GitOps | Argo CD |
| Osservabilità | Prometheus, Grafana, Loki, Grafana Alloy |
| Sicurezza avanzata | SOPS + age, Kyverno |
| Affidabilità | k6, backup su storage S3 (LocalStack) |

---

## 3. Architettura

```
                    ┌─────────────────────────────┐
  Browser  ───────▶ │  Reverse proxy (nginx)      │
  (Vue SPA)         │  /      → frontend statico  │
                    │  /api   → Laravel           │
                    │  /ws    → Reverb (v4)       │
                    └──────────────┬──────────────┘
                                   │
          ┌────────────────────────┼─────────────────────────┐
          ▼                        ▼                         ▼
   ┌─────────────┐         ┌──────────────┐          ┌──────────────┐
   │ API Laravel │         │ Reverb (v4)  │          │  Scheduler   │
   └──────┬──────┘         └──────┬───────┘          └──────┬───────┘
          │                       │                         │ accoda job
          ▼                       ▼                         ▼
   ┌─────────────┐         ┌──────────────┐          ┌──────────────┐
   │ PostgreSQL  │◀────────│    Redis     │◀─────────│    Worker    │──▶ API-Football
   └─────────────┘         └──────────────┘          └──────────────┘    (solo dal worker)
```

**Principio chiave**: API-Football viene chiamata **solo dal worker**, mai durante una richiesta utente. Gli utenti leggono solo dal nostro database. Così si rispettano i limiti del piano gratuito, l'app è veloce e resta funzionante anche se l'API esterna non risponde.

**Autenticazione**: Sanctum in modalità SPA, con frontend e API sotto lo stesso dominio (tramite reverse proxy) e cookie di sessione `HttpOnly` + `SameSite` con protezione CSRF. Niente token salvati nel browser.

### Job pianificati

| Job | Quando | Cosa fa |
|---|---|---|
| Sincronizzazione calendario | Ogni giorno, più spesso nei giorni di partita | Aggiorna giornate, partite, orari, stati (rinvii) |
| Congelamento quote | 3 giorni prima della prima partita della giornata | Salva le quote Bet365 e segna le giocate non disponibili; apre le schedine |
| Sincronizzazione risultati | Dopo la fine di ogni partita | Scarica risultato, eventi e statistiche |
| Valutazione provvisoria | Subito dopo la sincronizzazione dei risultati | Valuta le giocate e calcola i punti |
| Valutazione definitiva | Il mattino successivo | Riscarica i dati, applica eventuali correzioni e registra le modifiche |
| Aggiornamento classifiche | Dopo la valutazione definitiva | Calcola le classifiche per ogni formula |
| Sincronizzazione rose | Settimanale e nelle finestre di mercato | Aggiorna giocatori e squadre |

### Motore di valutazione delle giocate
Le giocate si raggruppano in **famiglie** con regole parametriche (es. "Over/Under" vale per gol, angoli e cartellini cambiando statistica e soglia). Le statistiche grezze di ogni partita si conservano: aggiungere una nuova famiglia permette anche di ricalcolare le giornate passate. Ogni famiglia ha i suoi test.

### Budget delle chiamate API (piano gratuito: 100 al giorno)
Stima indicativa, da confermare nella missione M07: poche chiamate al giorno per il calendario, una decina al congelamento delle quote, circa 1–3 chiamate per partita a fine gara, una ventina per le rose a inizio stagione. Si resta abbondantemente sotto il limite. La quota residua si monitora (fase 7).

---

## 4. Versioni dell'applicazione

| Versione | Contenuto | Sviluppata durante |
|---|---|---|
| **v1 · Il cuore** | Account (registrazione, verifica email, login), leghe e inviti, sincronizzazione dati, congelamento quote, schedina, motore di valutazione (nucleo di giocate), classifica a punteggio, registro degli interventi dell'admin | Fase 1 |
| **v2 · Le formule** | Scontri diretti (calendario, fasce, avversario fantasma), Formula 1, spareggi | Fase 3 |
| **v3 · Rose e bonus** | Rose assegnate manualmente dall'admin, bonus sulle giocate | Fase 5 |
| **v4 · Asta live** | Asta in tempo reale con Reverb | Fase 6 |
| **Extra** | Autenticazione a due fattori; nuove famiglie di giocate; taratura di fasce e bonus | Fasi 7–8 |

---

## 5. Fasi e missioni

Ogni missione ha: **obiettivo**, **concetti**, **attività** e **✅ Fatto quando** (la checklist che la chiude). Le missioni contrassegnate con 🧑‍💻 sono di sviluppo dell'app: il codice lo scrive Claude e l'utente lo rivede, lo esegue e lo prova.

---

### Fase 0 · Fondamenta

#### M01 · Preparare il Mac
- **Obiettivo**: un ambiente di lavoro pulito e riproducibile.
- **Concetti**: package manager, `PATH`, differenza tra terminale e shell.
- **Attività**: installare Homebrew, Git, VS Code e OrbStack; configurare l'identità di Git; creare un `Brewfile` con gli strumenti installati.
- ✅ **Fatto quando**:
  - [ ] `brew`, `git` e `docker` rispondono con la loro versione
  - [ ] esiste un `Brewfile` e `brew bundle` reinstalla tutto
  - [ ] so spiegare cos'è il `PATH` e come la shell trova un comando

#### M02 · Terminale e shell
- **Obiettivo**: muoversi nel terminale con sicurezza.
- **Concetti**: filesystem, percorsi assoluti e relativi, permessi (`rwx`, utente/gruppo/altri), processi, variabili d'ambiente, pipe e redirezioni.
- **Attività**: esercizi su `ls`, `cd`, `cp`, `mv`, `rm`, `find`, `grep`, `chmod`, `ps`, `kill`, `env`, `|`, `>`, `>>`.
- ✅ **Fatto quando**:
  - [ ] so trovare un file per nome e per contenuto
  - [ ] so leggere e modificare i permessi di un file
  - [ ] so trovare e terminare un processo
  - [ ] so concatenare comandi con le pipe e salvare l'output in un file

#### M03 · Linux in una VM
- **Obiettivo**: amministrare un server Linux di base.
- **Concetti**: utenti e gruppi, `sudo`, gestore di pacchetti (`apt`), servizi (`systemd`), log (`journalctl`), SSH con chiavi.
- **Attività**: creare una VM Ubuntu con OrbStack; creare un utente; installare nginx come servizio; accedere via SSH con chiave; disabilitare l'accesso con password.
- ✅ **Fatto quando**:
  - [ ] nginx parte all'avvio della VM ed è raggiungibile dal Mac
  - [ ] so leggere i log di un servizio
  - [ ] accedo via SSH solo con chiave (login con password disabilitato)

#### M04 · Networking pratico
- **Obiettivo**: capire cosa succede quando si apre un URL.
- **Concetti**: IP, porte, DNS, TCP, HTTP (metodi, status code, header), TLS e certificati.
- **Attività**: usare `curl -v`, `dig`, `lsof -i`, `openssl s_client` su siti reali e sulla VM.
- ✅ **Fatto quando**:
  - [ ] so raccontare passo passo il percorso di una richiesta HTTPS
  - [ ] so scoprire quale processo è in ascolto su una porta
  - [ ] so leggere un certificato TLS (emittente, scadenza, dominio)

#### M05 · Git e GitHub
- **Obiettivo**: lavorare con Git come in un team.
- **Concetti**: commit, branch, merge, rebase, conflitti, pull request, `.gitignore`, Conventional Commits.
- **Attività**: account GitHub con 2FA e chiave SSH; repository **privato** `fantaschedina`; caricare `CLAUDE.md` e i file di `docs/`; proteggere `main`; aprire e unire la prima pull request; creare `docs/DIARIO.md`.
- ✅ **Fatto quando**:
  - [ ] il repository privato contiene la documentazione
  - [ ] `main` è protetto e si modifica solo tramite pull request
  - [ ] ho risolto almeno un conflitto di merge in un esercizio
  - [ ] il `.gitignore` esclude `.env` fin dal primo commit

#### M06 · Scripting con Bash
- **Obiettivo**: automatizzare le operazioni ripetitive.
- **Concetti**: variabili, condizioni, cicli, exit code, `set -euo pipefail`, JSON con `jq`.
- **Attività**: scrivere `scripts/doctor.sh`, che controlla gli strumenti installati e le loro versioni e segnala cosa manca.
- ✅ **Fatto quando**:
  - [ ] lo script termina con exit code diverso da 0 se manca qualcosa
  - [ ] so spiegare cosa fa ogni opzione di `set -euo pipefail`

#### M07 · Missione speciale: verifica di API-Football
- **Obiettivo**: confermare con dati reali le scelte sui dati.
- **Concetti**: API REST, autenticazione con chiave, rate limit, JSON.
- **Attività**: registrarsi ad API-Football; salvare la chiave in `.env` (mai su Git); con `curl` e `jq` interrogare stato e quota residua, campionato e stagione di Serie A, partite della prossima giornata, bookmaker disponibili, quote di una partita, eventi e giocatori di una partita già giocata; scrivere `scripts/api-check.sh`.
- ✅ **Fatto quando**, documentato in `docs/adr/0001-fonte-dati.md`:
  - [ ] il piano gratuito copre la stagione in corso della Serie A
  - [ ] Bet365 è disponibile, con quali mercati
  - [ ] le quote marcatore esistono e **quanti giorni prima** della partita compaiono (osservare per qualche giorno)
  - [ ] gli identificativi dei giocatori coincidono tra rose ed eventi
  - [ ] stima reale delle chiamate per giornata
  - [ ] la chiave non compare in nessun commit

---

### Fase 1 · Applicazione (v1)

#### M08 · Ambiente di sviluppo locale
- **Obiettivo**: far girare l'applicazione sul Mac senza container (per apprezzarli dopo).
- **Concetti**: dipendenze e versioni, servizi locali, variabili d'ambiente.
- **Attività**: installare PHP, Composer, Node, PostgreSQL e Redis con Homebrew; avviare i servizi; Claude crea lo scheletro di Laravel e Vue.
- ✅ **Fatto quando**:
  - [ ] backend e frontend partono in locale e comunicano
  - [ ] Laravel è collegato a PostgreSQL e Redis
  - [ ] esiste un `.env.example` con soli segnaposto

#### M09 · Architettura e decisioni
- **Obiettivo**: capire come è fatta l'app prima di costruirla.
- **Concetti**: componenti, API REST, code e job, ADR.
- **Attività**: leggere e discutere con Claude l'architettura (sezione 3); scrivere `0002-stack.md` e `0003-job-e-dati.md`.
- ✅ **Fatto quando**:
  - [ ] so spiegare il ruolo di ogni componente
  - [ ] so spiegare perché l'API esterna si chiama solo dal worker

#### M10 · 🧑‍💻 v1, parte 1: fondamenta dell'app
- **Contenuto**: modello dati e migrazioni; registrazione, verifica email, login con Sanctum, Argon2id; leghe, inviti, ruoli; registro degli interventi dell'admin.
- **Ruolo dell'utente**: rivedere ogni pull request con la spiegazione di Claude, eseguire i test, provare a mano.
- ✅ **Fatto quando**:
  - [ ] posso registrarmi, verificare l'email, creare una lega e invitare un secondo utente
  - [ ] i test sono verdi

#### M11 · 🧑‍💻 v1, parte 2: il gioco
- **Contenuto**: job di sincronizzazione; congelamento quote; schedina (apertura, chiusura, modifiche, visibilità); motore di valutazione con il nucleo di giocate; valutazione provvisoria e definitiva; classifica a punteggio; interventi dell'admin (convalida partita, schedina manuale).
- ✅ **Fatto quando**:
  - [ ] una giornata reale completa funziona dall'inizio alla fine con dati veri
  - [ ] ogni regola del capitolo 2 del regolamento ha almeno un test
  - [ ] il `README.md` spiega come avviare il progetto

#### M12 · Test e qualità
- **Obiettivo**: capire perché e come si controlla il codice in automatico.
- **Concetti**: piramide dei test, analisi statica, formattazione, `Makefile`.
- **Attività**: eseguire Pest, PHPStan, Pint, ESLint e Vitest; creare un `Makefile` con `make check` che li esegue tutti.
- ✅ **Fatto quando**:
  - [ ] `make check` esegue tutti i controlli e fallisce se uno fallisce
  - [ ] so spiegare la differenza tra test unitari, di integrazione ed end-to-end

---

### Fase 2 · Container

#### M13 · Il primo container
- **Concetti**: container e VM a confronto, immagini, layer, registry, porte, volumi.
- **Attività**: scrivere il Dockerfile del backend (PHP-FPM) ed eseguirlo collegato al database locale.
- ✅ **Fatto quando**:
  - [ ] l'immagine si costruisce e il container risponde
  - [ ] so spiegare cosa succede a ogni istruzione del Dockerfile

#### M14 · Immagini piccole e sicure
- **Concetti**: build multi-stage, utente non root, `.dockerignore`, versioni fissate, healthcheck, scansione delle vulnerabilità.
- **Attività**: ottimizzare il Dockerfile del backend; scrivere quello del frontend (build Vite + nginx); scansionare le immagini con Trivy.
- ✅ **Fatto quando**:
  - [ ] ho misurato la dimensione delle immagini prima e dopo
  - [ ] i container girano come utente non root
  - [ ] Trivy non segnala vulnerabilità critiche

#### M15 · Docker Compose
- **Concetti**: servizi, reti, volumi, variabili d'ambiente, dipendenze e healthcheck.
- **Attività**: comporre reverse proxy, frontend, API, worker, scheduler, PostgreSQL e Redis.
- ✅ **Fatto quando**:
  - [ ] da un clone pulito `docker compose up` avvia tutta l'app
  - [ ] i dati del database sopravvivono al riavvio
  - [ ] il README è aggiornato

---

### Fase 3 · Integrazione continua (+ v2)

#### M16 · La prima pipeline
- **Concetti**: CI, workflow, job, runner, servizi nella pipeline.
- **Attività**: GitHub Actions su ogni pull request: lint, analisi statica, test (con PostgreSQL come servizio), build del frontend. La protezione di `main` richiede la pipeline verde.
- ✅ **Fatto quando**:
  - [ ] una pull request con un test rotto non si può unire

#### M17 · Sicurezza nella pipeline
- **Concetti**: shift-left, SAST, scansione dei segreti e delle dipendenze, SBOM.
- **Attività**: Gitleaks (anche come hook pre-commit), Semgrep, `composer audit` e `npm audit`, Trivy sulle immagini, generazione dell'SBOM, Dependabot.
- ✅ **Fatto quando**:
  - [ ] la pipeline fallisce se trova un segreto o una vulnerabilità critica
  - [ ] l'SBOM viene salvato tra gli artefatti della pipeline

#### M18 · Build e pubblicazione delle immagini
- **Concetti**: tag delle immagini (SHA del commit, versioni semantiche), registry, cache di build.
- **Attività**: costruire e pubblicare le immagini su GitHub Container Registry (privato) a ogni merge su `main`.
- ✅ **Fatto quando**:
  - [ ] ogni merge produce immagini taggate e rintracciabili fino al commit

#### 🧑‍💻 Sviluppo v2 in questa fase
Scontri diretti (calendario, fasce, avversario fantasma), Formula 1, spareggi. Ogni pull request passa dalla pipeline appena costruita.

---

### Fase 4 · Infrastructure as Code

#### M19 · Ansible
- **Concetti**: configurazione dichiarativa, inventory, playbook, ruoli, idempotenza.
- **Attività**: trattare una VM OrbStack come un "server": playbook che la mette in sicurezza (SSH, firewall, aggiornamenti automatici, utente di deploy), installa Docker e avvia l'app con Compose.
- ✅ **Fatto quando**:
  - [ ] da una VM nuova il playbook porta l'app online
  - [ ] rieseguendo il playbook non cambia nulla (idempotenza)

#### M20 · Terraform
- **Concetti**: provider, risorse, stato, `plan` e `apply`, variabili, moduli.
- **Attività**: gestire con Terraform le impostazioni del repository GitHub (protezione dei branch, ecc.); creare con LocalStack un bucket S3 che servirà per i backup.
- ✅ **Fatto quando**:
  - [ ] le impostazioni del repository sono definite come codice
  - [ ] il file di stato non è su Git
  - [ ] so spiegare cosa succede se lo stato si perde

---

### Fase 5 · Kubernetes (+ v3)

#### M21 · Concetti e primo cluster
- **Concetti**: cluster, nodi, pod, deployment, service, namespace, `kubectl`.
- **Attività**: creare un cluster con k3d; fare il deploy dell'API a mano con i manifest.
- ✅ **Fatto quando**:
  - [ ] so spiegare cosa succede quando cancello un pod
  - [ ] l'API risponde dal cluster

#### M22 · L'app completa su Kubernetes
- **Concetti**: ConfigMap, Secret, Ingress, probe (liveness e readiness), richieste e limiti di risorse, StatefulSet e volumi persistenti, Job per le migrazioni.
- **Attività**: portare nel cluster tutti i componenti (proxy, frontend, API, worker, scheduler, PostgreSQL, Redis).
- ✅ **Fatto quando**:
  - [ ] l'app è raggiungibile su `fantaschedina.localhost`
  - [ ] i dati sopravvivono al riavvio dei pod
  - [ ] le migrazioni girano come Job prima del rilascio

#### M23 · Helm
- **Concetti**: chart, template, values, release, rollback.
- **Attività**: impacchettare l'app in un chart con valori separati per gli ambienti (es. `dev` e `prod` locale).
- ✅ **Fatto quando**:
  - [ ] installo, aggiorno e faccio rollback dell'app con Helm

#### 🧑‍💻 Sviluppo v3 in questa fase
Rose assegnate dall'admin e bonus sulle giocate.

---

### Fase 6 · GitOps (+ v4)

#### M24 · Argo CD
- **Concetti**: GitOps, stato desiderato e stato reale, sincronizzazione, deriva della configurazione.
- **Attività**: installare Argo CD; collegarlo al repository; la pipeline aggiorna il tag dell'immagine e Argo CD rilascia.
- ✅ **Fatto quando**:
  - [ ] un merge su `main` porta la nuova versione nel cluster senza comandi manuali
  - [ ] un `git revert` riporta indietro il rilascio
  - [ ] una modifica fatta a mano nel cluster viene annullata da Argo CD

#### M25 · 🧑‍💻 Asta live e scalabilità
- **Contenuto**: Claude sviluppa la v4 (asta con Reverb); l'utente la porta nel cluster.
- **Concetti**: WebSocket, stato condiviso, repliche multiple, pub/sub con Redis, Ingress con WebSocket.
- **Attività**: avviare 2 repliche di API e Reverb; osservare cosa si rompe e capire perché; applicare la soluzione.
- ✅ **Fatto quando**:
  - [ ] un'asta con 3 browser funziona con 2 repliche
  - [ ] so spiegare il problema incontrato e come è stato risolto

---

### Fase 7 · Osservabilità

#### M26 · Metriche e dashboard
- **Concetti**: metriche, time series, scraping, i "quattro segnali d'oro" (latenza, traffico, errori, saturazione).
- **Attività**: installare Prometheus e Grafana; esporre metriche dell'app (lunghezza della coda, job falliti, **quota residua di API-Football**); creare una dashboard.
- ✅ **Fatto quando**:
  - [ ] la dashboard mostra lo stato dell'app e del budget API

#### M27 · Log e alert
- **Concetti**: log strutturati, aggregazione dei log, alert, SLO.
- **Attività**: log in JSON; Loki e Grafana Alloy; alert per job di sincronizzazione falliti, congelamento quote non avvenuto, quota API oltre l'80%.
- ✅ **Fatto quando**:
  - [ ] trovo in Grafana i log di un job specifico
  - [ ] simulando un guasto, l'alert scatta

#### M28 · 🧑‍💻 Analisi dati per la taratura
- **Contenuto**: analisi delle statistiche salvate per misurare la frequenza degli eventi e i punteggi tipici di giornata.
- **Attività**: proporre i valori dei bonus e i default delle fasce; documentare la decisione in un ADR e aggiornare `REGOLAMENTO.md`.
- ✅ **Fatto quando**:
  - [ ] bonus e fasce hanno valori motivati dai dati

---

### Fase 8 · Sicurezza avanzata

#### M29 · Gestione dei segreti
- **Concetti**: cifratura dei segreti nel repository, rotazione delle chiavi.
- **Attività**: cifrare i segreti con SOPS e age; integrarli nel deploy; esercitarsi a ruotare la chiave di API-Football.
- ✅ **Fatto quando**:
  - [ ] nel repository non ci sono segreti in chiaro
  - [ ] so ruotare una chiave senza interrompere l'app

#### M30 · Policy e supply chain
- **Concetti**: policy as code, Pod Security Standards, firma delle immagini, network policy.
- **Attività**: policy Kyverno (niente root, niente tag `latest`, limiti di risorse obbligatori, solo immagini firmate); firma delle immagini con Cosign in pipeline; network policy "nega tutto" con eccezioni esplicite.
- ✅ **Fatto quando**:
  - [ ] un'immagine non firmata viene rifiutata dal cluster
  - [ ] un pod non può parlare con un servizio non autorizzato

#### M31 · Threat model e 2FA
- **Contenuto**: Claude sviluppa l'autenticazione a due fattori (TOTP).
- **Attività**: l'utente scrive il threat model dell'app (es. con STRIDE): qui la tua formazione in cybersecurity è protagonista.
- ✅ **Fatto quando**:
  - [ ] il threat model è in `docs/` con le contromisure e il loro stato
  - [ ] la 2FA funziona

---

### Fase 9 · Affidabilità

#### M32 · Backup e ripristino
- **Concetti**: RPO e RTO; un backup mai ripristinato non è un backup.
- **Attività**: backup automatici di PostgreSQL sul bucket S3 di LocalStack; prova di ripristino completa.
- ✅ **Fatto quando**:
  - [ ] ho cancellato il database e l'ho ripristinato dal backup

#### M33 · Carico e autoscaling
- **Concetti**: test di carico, autoscaling orizzontale.
- **Attività**: test di carico con k6 (es. tutti i partecipanti che inviano la schedina all'ultimo minuto); HPA sull'API.
- ✅ **Fatto quando**:
  - [ ] sotto carico le repliche aumentano e poi tornano al minimo

#### M34 · Rompi e ripara
- **Concetti**: chaos engineering, runbook, postmortem.
- **Attività**: simulare guasti (pod terminati, Redis giù, API-Football irraggiungibile, chiave scaduta) e scrivere un runbook per ciascuno; scrivere un postmortem su uno di essi.
- ✅ **Fatto quando**:
  - [ ] ogni scenario ha un runbook testato
  - [ ] esiste almeno un postmortem

---

## 6. Dopo il percorso

- Portare il progetto su un cloud (AWS, Azure o GCP), riusando Terraform e Helm.
- Notifiche (canale da definire).
- Nuove famiglie di giocate.
- Le idee future elencate in `REGOLAMENTO.md`.

---

## Appendice · Modello per il diario (`docs/DIARIO.md`)

```markdown
## AAAA-MM-GG · Missione MXX

**Fatto**: …
**Capito**: …
**Dubbi / problemi**: …
**Prossimo passo**: …
```
