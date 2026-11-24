# Diario di bordo

Una voce per sessione: cosa è stato fatto, cosa ho capito, cosa resta aperto.
Modello in `PIANO.md`, appendice.

---

## 2026-11-13 · Missione M01

**Fatto**: installati Homebrew, git, node, VS Code e OrbStack; VS Code e OrbStack
(installati a mano) "adottati" da Homebrew con `--adopt`; creato il `Brewfile`
con formule, cask ed estensioni di VS Code; aggiunto `typeset -U path PATH` a `~/.zshrc`.

**Capito**:
- terminale ≠ shell; il PATH è un elenco ordinato di cartelle e la shell esegue
  il primo eseguibile che trova
- il PATH si legge all'avvio della shell: dopo averlo cambiato serve un terminale nuovo
- ordine di lettura: `~/.zprofile` poi `~/.zshrc`, quindi l'ultimo che aggiunge in testa vince
- i processi figli ereditano l'ambiente del padre (causa dei doppioni nel terminale di VS Code)
- su macOS i permessi sono a due livelli: Unix (`sudo`) e TCC ("Gestione app")
- il Brewfile vale solo quanto è completo; `brew bundle` è idempotente
- PATH hijacking: le prime cartelle del PATH devono essere scrivibili solo da me

**Dubbi / problemi**: `which git` mostrava il git di Apple perché il terminale era stato
aperto prima di installare Homebrew; `brew --adopt` bloccato da TCC, risolto dando
"Gestione app" al Terminale.

**Prossimo passo**: M02 · Terminale e shell.

---

## 2026-11-16 · Missione M02

**Fatto**: esercizi su percorsi, find/grep, permessi, processi e segnali, redirezioni,
variabili d'ambiente.

**Capito**:
- percorsi assoluti e relativi; lo spazio separa gli argomenti
- permessi rwx per user/group/others; sulle cartelle r = elencare, x = entrare; `.env` → 600
- i processi formano un albero (PID/PPID); SIGTERM permette di chiudere pulito, SIGKILL no:
  prima `kill`, poi `kill -9`
- variabile di shell ≠ variabile d'ambiente: solo `export` la passa ai figli (come copia)
- stdout (1) e stderr (2) sono canali separati: `>` devia solo stdout

**Dubbi / problemi**: pipe e redirezioni da consolidare (ripresa in M06).

**Prossimo passo**: M03 · Linux in una VM.

---

## 2026-11-24 · Missioni M03 e M04

**Fatto**: VM Ubuntu 26.04 (arm64) con OrbStack; utente `deploy` nel gruppo sudo;
nginx installato come servizio; esercizi su systemd, journalctl, lsof/ss, dig, curl -v,
openssl s_client, badssl.com. Diagnosi completa del problema `.orb.local`.

**Capito**:
- start ≠ enable; PID 1 = systemd; `nginx -t` prima di `restart` (verifica, poi applica)
- `sudo cmd >> file` fallisce: la redirezione la fa la mia shell → `| sudo tee -a`
- 0.0.0.0 = tutte le interfacce, 127.0.0.1 = solo la macchina stessa
- connection refused ≠ timeout ≠ could not resolve ≠ no route to host
- il TTL è un conto alla rovescia della cache; NOERROR non basta: leggere la risposta
- percorso HTTPS: DNS → TCP → TLS (date, nome, catena di fiducia) → HTTP

**Dubbi / problemi**: `.orb.local` non si risolve perché il DNS del router dichiara un SOA
per `.local` (filtro RPZ) e macOS lo preferisce all'mDNS di OrbStack. Soluzione possibile:
DNS Quad9 sul Mac; per ora uso gli IP. SSH con chiave rimandato alla M05.

**Prossimo passo**: M05 · Git e GitHub (+ SSH con chiavi).
