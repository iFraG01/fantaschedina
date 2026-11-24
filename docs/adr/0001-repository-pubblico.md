# ADR 0001 · Repository pubblico su GitHub

- **Data**: 2026-11-24
- **Stato**: accettata
- **Missione**: M05 · Git e GitHub

## Contesto

Il piano prevedeva un repository **privato** su un account GitHub gratuito, con il branch
`main` protetto (modifiche solo tramite pull request e, dalla M16, solo con la pipeline verde).

Durante la M05 è emerso che, con il piano gratuito, le regole di protezione dei branch
**non vengono applicate** ai repository privati: si possono creare, ma GitHub non le fa
rispettare. Senza protezione non si possono verificare due criteri del piano:

- M05: "`main` è protetto e si modifica solo tramite pull request";
- M16: "una pull request con un test rotto non si può unire".

## Decisione

Il repository `fantaschedina` diventa **pubblico**.

Prima della pubblicazione è stato fatto un controllo della storia:

- tutti i commit usano l'indirizzo email `noreply` di GitHub;
- nessun segreto presente nella storia;
- `.env` escluso dal `.gitignore` fin dal primo commit;
- informazioni personali rimosse da `CLAUDE.md` e `PIANO.md`.

## Conseguenze

**Positive**

- La protezione di `main` funziona davvero (verificato: un push diretto viene rifiutato).
- Secret scanning e push protection di GitHub gratuiti: un primo livello di difesa contro
  i segreti, in attesa di Gitleaks (M17).
- Minuti di GitHub Actions senza limiti per i repository pubblici.
- Il progetto diventa un portfolio visibile.

**Negative e mitigazioni**

- Codice e documentazione sono visibili a tutti → nel repository non entrano mai segreti
  né dati personali (regola già presente in `CLAUDE.md`, rafforzata da Gitleaks e SOPS).
- Ciò che entra nella storia resta pubblico anche se poi viene cancellato → un segreto
  finito per errore su Git si considera compromesso e si **ruota** subito.
- Le immagini sul GitHub Container Registry (M18) potranno essere pubbliche → nelle
  immagini non si inseriscono mai segreti: la configurazione arriva a runtime.

## Alternative considerate

- **GitHub Pro**: il repository resta privato con tutte le protezioni, ma a pagamento.
- **Repository privato con protezione "di disciplina"**: un hook Git locale (`pre-push`)
  che blocca i push su `main`. Gratuito, ma aggirabile e non applicabile alle pull request.
