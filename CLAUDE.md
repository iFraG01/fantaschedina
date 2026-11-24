# CLAUDE.md — FantaSchedina

Questo file dà a Claude Code il contesto del progetto e le regole di lavoro. Va letto a ogni sessione.

## Il progetto in breve

**FantaSchedina** è una web app per leghe private tra amici. A ogni giornata di Serie A ogni partecipante compila una schedina con una giocata per partita. Il punteggio è la somma delle quote delle giocate indovinate. Le leghe si giocano con tre formule (classifica a punteggio, scontri diretti, Formula 1), con asta e bonus facoltativi.

Il progetto è prima di tutto un **percorso per imparare il DevOps da zero**: l'applicazione è il pretesto, l'obiettivo è imparare a costruirla, impacchettarla, rilasciarla, osservarla e proteggerla bene.

- Regole del gioco: `docs/REGOLAMENTO.md` (fonte di verità)
- Percorso, fasi e missioni: `docs/PIANO.md`
- Diario di bordo: `docs/DIARIO.md`
- Decisioni architetturali: `docs/adr/`

## Chi è l'utente

- Laurea magistrale in cybersecurity, **principiante assoluto nel DevOps**: va trattato come uno che parte da zero, senza dare per scontati i concetti di base.
- Lavora a ritmo libero, quando ne ha voglia: possono passare settimane tra una sessione e l'altra.
- Macchina: MacBook Air M5 (Apple Silicon, arm64), 24 GB di RAM. Tutto gira **in locale**; il cloud eventualmente più avanti.
- Lingua: **italiano** per spiegazioni, documentazione e interfaccia utente.

## Modalità di lavoro (fondamentale)

### Codice dell'applicazione (Laravel, Vue): lo scrive Claude
- Claude scrive tutto il codice applicativo (backend, frontend, test, migrazioni).
- Dopo ogni modifica **spiega cosa fa il codice e perché**, in modo comprensibile a un principiante: il flusso, le scelte, eventuali alternative scartate.
- Lavora a passi piccoli e verificabili, con test per la logica di gioco (valutazione delle giocate, punteggi, classifiche, asta).

### Attività DevOps: le svolge l'utente
Riguarda la fase 0, Docker, CI, IaC, Kubernetes, GitOps, osservabilità, sicurezza e affidabilità.
- Claude **non esegue le attività al posto dell'utente** e non scrive per lui i file DevOps (Dockerfile, compose, workflow, manifest, chart, playbook, Terraform…), salvo richiesta esplicita.
- Per ogni missione Claude:
  1. spiega il concetto (il *perché*) prima dello strumento (il *come*);
  2. dà istruzioni passo passo;
  3. rivede ciò che l'utente ha scritto o eseguito (review, output, errori) e suggerisce miglioramenti;
  4. se l'utente è bloccato, dà indizi progressivi prima della soluzione completa.
- Una missione è chiusa solo quando tutti i punti di **"✅ Fatto quando"** in `docs/PIANO.md` sono verificati.

### Inizio e fine sessione
- **Inizio**: leggere `docs/DIARIO.md` (ultime voci) e la missione in corso in `docs/PIANO.md`; riassumere in due righe dove eravamo rimasti.
- **Fine**: proporre all'utente la voce di diario della sessione (cosa fatto, cosa capito, cosa resta aperto).

### Regolamento
- `docs/REGOLAMENTO.md` è la fonte di verità per le regole del gioco.
- Se una regola è ambigua o manca, **chiedere all'utente** e poi aggiornare il regolamento: mai inventarla nel codice.

## Regole non negoziabili (sicurezza)

1. **I segreti sono sacri**: chiavi API, password e token non finiscono mai nel codice né su Git. Si usa `.env` (in `.gitignore`) con un `.env.example` contenente solo segnaposto. Più avanti: Gitleaks in pre-commit e in CI, poi SOPS.
2. **API-Football non viene mai chiamata durante una richiesta utente.** Solo job pianificati sincronizzano i dati nel database; gli utenti leggono solo dal database. Il budget del piano gratuito è di 100 richieste al giorno.
3. **Ogni intervento manuale dell'admin di lega** (punteggi, schedine inserite a mano, partite convalidate) va registrato nel registro degli interventi con chi, cosa, quando e motivazione, ed è visibile a tutti i membri.
4. Password con hashing **Argon2id**; verifica email obbligatoria; dati personali minimi (email e nickname).
5. Container non root, immagini con versioni fissate, permessi minimi ovunque (least privilege).
6. Nessun denaro reale: il gioco è solo a punti.

## Stack

| Area | Tecnologia |
|---|---|
| Backend | Laravel (PHP) con Sanctum (autenticazione SPA via cookie di sessione, protezione CSRF) |
| Frontend | Vue 3 + Vite, Vue Router, Pinia (SPA) |
| Database | PostgreSQL |
| Cache, code, broadcast | Redis |
| Job in background | Laravel Queues (driver Redis), eventualmente Horizon |
| Job pianificati | Laravel Scheduler |
| Tempo reale (asta, v4) | Laravel Reverb + Laravel Echo |
| Test e qualità | Pest/PHPUnit, PHPStan (Larastan), Pint, ESLint, Vitest |
| Dati calcio | API-Football (piano gratuito), quote Bet365 |

Strumenti DevOps: Homebrew, Git/GitHub (repository privato), OrbStack, GitHub Actions, Gitleaks, Semgrep, Trivy, Ansible, Terraform, LocalStack, k3d, Helm, Argo CD, Prometheus, Grafana, Loki, SOPS, Kyverno, Cosign, k6.

## Struttura del repository

```
fantaschedina/
├── CLAUDE.md
├── README.md
├── backend/             # applicazione Laravel
├── frontend/            # applicazione Vue 3
├── scripts/             # script di supporto (bash)
├── docs/
│   ├── PIANO.md
│   ├── REGOLAMENTO.md
│   ├── DIARIO.md
│   └── adr/             # 0001-fonte-dati.md, 0002-stack.md, ...
├── deploy/
│   ├── docker/          # Dockerfile e configurazioni dei container
│   ├── compose/         # docker compose
│   ├── k8s/             # manifest Kubernetes (fase 5)
│   └── helm/            # chart Helm (fase 5+)
├── infra/
│   ├── ansible/
│   └── terraform/
└── .github/workflows/   # pipeline CI
```

## Convenzioni

- Codice, identificatori, nomi di tabelle e commit message in **inglese**; documentazione, commenti esplicativi e interfaccia utente in **italiano**.
- Commit nel formato **Conventional Commits** (`feat:`, `fix:`, `docs:`, `chore:`, `ci:`…).
- Un branch per missione o funzionalità (`missione/07-verifica-api`, `feat/schedina`), merge su `main` tramite pull request. `main` è protetto.
- Le decisioni importanti si documentano in un ADR breve (contesto, decisione, conseguenze).

## Comandi utili

*(Da completare man mano che il progetto cresce: avvio locale, test, build, deploy.)*
