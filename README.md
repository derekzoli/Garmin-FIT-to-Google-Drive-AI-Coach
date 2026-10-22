# Garmin FIT → Google Drive → AI Coach 🏃‍♂️

**Italiano** · [English](#english)

> **Una guida pratica, senza sviluppare una propria API:** sincronizza automaticamente le attività Garmin in formato FIT su una cartella Google Drive, per poterle esaminare con un assistente AI che abbia accesso autorizzato alla stessa cartella.

## Italiano

### L’idea

Trasformare un chatbot, come **ChatGPT o Claude**, in un possibile **coach di allenamento** alimentato dai propri dati Garmin. Dopo una corsa, l'orologio sincronizza Garmin Connect; Home Assistant scarica il FIT; rclone lo copia su Drive. In una chat dotata di un **connettore Google Drive compatibile con il recupero dei FIT**, puoi chiedere: *«Leggi l’ultimo allenamento, confrontalo con il precedente e suggeriscimi la prossima seduta»*.

**Senza API personalizzate** significa che non è necessario sviluppare né ospitare una propria integrazione API, un server o un bot dedicato. Sono comunque utilizzati l'integrazione Garmin Connect per Home Assistant, rclone e le normali autorizzazioni Google; l'accesso di ChatGPT/Claude a Drive dipende da funzionalità, piano e connettori disponibili. **Il solo collegamento della cartella non garantisce che il chatbot sappia decodificare file FIT binari.** Verificalo nel tuo ambiente.

### Come funziona

```text
Garmin → Garmin Connect → Home Assistant → FIT locale
                                        ↓
                              Rclone Backup → Google Drive / Garmin
                                                      ↓
                                  Assistente AI con accesso autorizzato
                                                      ↓
                                 Analisi, confronti e coaching in chat
```

- **Evento:** una nuova `activityId` attiva il download del FIT.
- **Upload:** Home Assistant riavvia l'add-on Rclone Backup; un job privo di `schedule` parte all'avvio.
- **Sicurezza:** un secondo job `copy` ogni 15 minuti funge da recupero per file già presenti localmente.
- **Privacy:** credenziali e FIT originali restano privati; i FIT possono contenere GPS e dati fisiologici.

### Requisiti

Home Assistant OS/Supervised, HACS, l'integrazione [Garmin Connect di cyberjunky](https://github.com/cyberjunky/home-assistant-garmin_connect), l'add-on [Rclone Backup di jcwillox](https://github.com/jcwillox/hassio-rclone-backup), un account Garmin e un account Google Drive. Un chatbot con accesso consentito a Drive è facoltativo per il solo backup, necessario per l'uso AI qui descritto.

### Inizia da qui

1. Segui la **[guida completa in italiano](docs/Garmin_to_Drive_IT.md)**.
2. Personalizza l'[automazione YAML di esempio](examples/garmin_to_drive_automation.yaml), soprattutto il **nome del sensore e lo slug dell'add-on**.
3. Verifica il trasferimento con una nuova attività di test e controlla l'ID del file su Drive.
4. Autorizza, se disponibile, un chatbot a leggere la cartella privata; verifica che possa ottenere e interpretare i `.fit`.
5. Conserva, facoltativamente, un `COACH_MEMORY.md` privato nella cartella Drive come sintesi delle analisi e degli obiettivi, **senza pubblicarlo nel repository**.

**Stato:** procedura di trasferimento provata su Home Assistant con Rclone Backup **3.4.1** (ottobre 2026). Non è un prodotto ufficiale Garmin, Google, Home Assistant, rclone, OpenAI o Anthropic, né un servizio di coaching medico o sportivo certificato.

### Limiti importanti

- I ritardi end-to-end possono essere di alcuni minuti: non è una sincronizzazione istantanea garantita.
- Il job su evento è avviato **riavviando l'add-on**, non attraverso una vera chiamata `rclone rc`; può interrompere un trasferimento concorrente.
- Il job periodico recupera FIT già scaricati localmente, ma non sostituisce il trigger Garmin quando il download non è mai avvenuto.
- I chatbot possono non accedere ai file binari, non comprendere il formato FIT o non supportare il connettore nel proprio piano.
- Non caricare sul repository `rclone.conf`, token, password, file `.fit` reali, tracce GPS o `COACH_MEMORY.md` personale.

## English

### Purpose

**Turn your Garmin workout history into an AI-assisted coaching workflow without building a custom API.** Sync Garmin activity `.fit` files to a private Google Drive folder using Home Assistant and rclone. Then, when your AI assistant (for example **ChatGPT or Claude**) has **authorized, compatible access to that folder and can process FIT binaries**, ask it to analyze a run, compare sessions and help plan the next workout.

“No custom API” means **no custom API application, hosted backend or bot integration is needed**. Existing Garmin/Home Assistant/rclone/Google integrations and OAuth authorizations are still used. AI connector access, binary-file retrieval and FIT decoding **vary by product, subscription and configuration**. They are not guaranteed merely by sharing a folder.

### Workflow

```text
Garmin → Garmin Connect → Home Assistant → Local .fit file
                                       ↓
                         Rclone Backup → Private Google Drive folder
                                                     ↓
                                  Authorized AI assistant / chatbot
                                                     ↓
                                Workout analysis and training insights
```

### What is included

- A tested Home Assistant automation watching the Garmin activity ID.
- A scheduled fallback and an on-start rclone copy job triggered by restarting the add-on.
- Step-by-step setup and troubleshooting in the [Italian setup guide](docs/Garmin_to_Drive_IT.md).
- A reusable [Home Assistant YAML automation](examples/garmin_to_drive_automation.yaml).
- Documentation and safety notes; **no credentials, personal fitness records or proprietary Garmin SDK**.

### Quick start

1. Install and configure [cyberjunky’s Garmin Connect integration](https://github.com/cyberjunky/home-assistant-garmin_connect) through HACS.
2. Configure [jcwillox’s Rclone Backup add-on](https://github.com/jcwillox/hassio-rclone-backup) with an authenticated Google Drive remote.
3. Configure one `copy` job scheduled every 15 minutes and a second `copy` job **without a schedule** (runs when the add-on starts).
4. Configure the [Home Assistant automation](examples/garmin_to_drive_automation.yaml), replacing the integration entity and add-on slug with values from **your** installation.
5. Record a short test activity, sync Garmin Connect and verify that a **new FIT activity ID** reaches Drive without manual intervention.
6. Optionally connect a compatible AI assistant to your **private** Drive folder and test actual FIT parsing before relying on it as a coach.

See the [full Italian instructions](docs/Garmin_to_Drive_IT.md) for configuration details, testing and troubleshooting. Contributions and an English full translation are welcome.

### Safety and limitations

FIT files can contain GPS routes, exact times, heart-rate measurements and other personal data. Do not publish these, OAuth credentials or `rclone.conf`. Restarting the add-on can interrupt ongoing jobs. Background sync can take several minutes. AI insights are supplementary and are not medical advice.

## Credits & license

This is an **independent integration guide** combining existing open-source projects; credit goes to their maintainers. Documentation is made available under [CC BY 4.0](LICENSE), subject to the licenses of any third-party material linked here. Contributions: [CONTRIBUTING.md](CONTRIBUTING.md).
