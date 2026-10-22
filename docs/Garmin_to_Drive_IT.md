# Garmin → Home Assistant → Google Drive

**Guida pratica per archiviare automaticamente i file FIT delle attività Garmin**  
Versione del documento: 1.0 — 22 ottobre 2026  
Installazione di riferimento collaudata: Home Assistant OS, integrazione Garmin Connect di cyberjunky e add-on Rclone Backup 3.4.1 di jcwillox.

> **Risultato finale:** dopo aver salvato e sincronizzato un'attività Garmin, Home Assistant ne rileva l'ID, scarica il relativo file `.fit` e avvia un job rclone che lo copia su Google Drive. Un secondo job ogni 15 minuti costituisce la rete di sicurezza. Non è necessario lasciare acceso un computer Windows.

## 1. Schema di funzionamento

```text
Garmin (orologio)
    │ sincronizzazione
    ▼
Garmin Connect (cloud)
    │ integrazione HACS: nuova activityId
    ▼
Home Assistant
    │ garmin_connect.download_activity
    ▼
/config/garmin_activities/activity_<id>.fit
    │ dopo 5 secondi: hassio.addon_restart
    ▼
Rclone Backup (job senza schedule, eseguito all'avvio)
    │ rclone copy
    ▼
Google Drive / Garmin / activity_<id>.fit

Rete di sicurezza: un secondo job rclone ripete il confronto ogni 15 minuti.
```

**Nota sui tempi:** il trasferimento di un piccolo FIT con rclone può richiedere meno di un secondo, ma il tempo complessivo fra salvataggio sull'orologio e disponibilità su Drive può essere di alcuni minuti per sincronizzazione e aggiornamenti dei servizi. Nel nostro collaudo l'intero flusso ha impiegato indicativamente 3–4 minuti; non è un tempo garantito.

## 2. Prerequisiti

- Home Assistant **OS o Supervised**, con accesso alla gestione delle **App/Add-on** e alle automazioni. In installazioni Container/Core il percorso può essere diverso.
- Orologio o dispositivo Garmin che sincronizzi le attività con **Garmin Connect**.
- Account Garmin Connect funzionante.
- Google Drive con spazio disponibile.
- **HACS**, per installare l'integrazione Garmin Connect personalizzata.
- Possibilità di modificare i file di configurazione HA, per esempio tramite **File Editor**.
- Add-on **Rclone Backup** e possibilità di consultarne il **Registro**.
- Facoltativo: rclone per Windows, usato **solo nella configurazione iniziale** dell'accesso Google se la procedura di autorizzazione dentro Home Assistant crea problemi.

Repository originali:

- Integrazione Garmin Connect: https://github.com/cyberjunky/home-assistant-garmin_connect
- Add-on Rclone Backup: https://github.com/jcwillox/hassio-rclone-backup
- Guida ufficiale add-on: https://github.com/jcwillox/hassio-rclone-backup/blob/main/rclone_backup/DOCS.md
- Documentazione generale rclone/Google Drive: https://rclone.org/drive/

> Le schermate, i nomi dei menu e le possibilità di accesso dipendono dalla versione di Home Assistant. La procedura è stata verificata con Rclone Backup **3.4.1**, ma non è una garanzia per tutte le versioni future.

## 3. Installare e configurare Garmin Connect in Home Assistant

1. Installare HACS seguendo la sua documentazione e verificare che sia operativo.
2. Da HACS cercare l'integrazione **Garmin Connect** di **cyberjunky** (`home-assistant-garmin_connect`). Se non compare, utilizzare la funzione repository personalizzato indicando il link GitHub qui sopra e il tipo integrazione.
3. Installare l'integrazione e riavviare Home Assistant, se richiesto.
4. Da **Impostazioni → Dispositivi e servizi → Aggiungi integrazione**, scegliere Garmin Connect e autenticare il proprio account. Completare le eventuali richieste MFA.
5. In **Strumenti per sviluppatori → Stati**, cercare l'entità `sensor.garmin_connect_last_activity` (il nome effettivo può essere diverso). Verificare che tra gli attributi ci sia **`activityId`**.

**Perché conta `activityId`:** è l'identificativo univoco dell'allenamento su Garmin Connect. Cambiando dopo la sincronizzazione di un'attività, può essere usato come trigger dell'automazione.

### Test manuale del download FIT

Da **Strumenti per sviluppatori → Azioni** cercare e provare `garmin_connect.download_activity` con l'ID di una vera attività:

```yaml
action: garmin_connect.download_activity
data:
  activity_id: 12345678901  # sostituire con il proprio activityId
  file_format: fit
```

La documentazione dell'integrazione indica che il percorso predefinito è:

```text
<config>/garmin_activities/activity_<id>.fit
```

Nel nostro Home Assistant è quindi:

```text
/config/garmin_activities/activity_<id>.fit
```

Verificare con **File Editor** che il file esista. Nel test reale il file era un FIT binario, non un semplice file di testo; il fatto che non sia leggibile direttamente in File Editor è normale.

## 4. Preparare Google Drive e la configurazione rclone

### 4.1 Cartella di destinazione

Creare su Google Drive una cartella, per esempio **`Garmin`**. Il percorso rclone userà il nome remoto `google` e la cartella `Garmin`:

```text
google:Garmin
```

`google` **non è una parola speciale**: è il nome assegnato al remote durante `rclone config`. Se lo si chiama diversamente, adattare tutti gli esempi.

### 4.2 Ottenere `rclone.conf`

Configurare un remote Google Drive mediante rclone. Se l'autorizzazione interattiva non è praticabile nell'add-on, si può configurare temporaneamente **rclone su Windows**:

1. Installare rclone sul PC e avviare `rclone config` dal terminale Windows.
2. Creare un nuovo remote di nome `google`, scegliendo il backend `drive`.
3. Seguire il flusso OAuth richiesto da Google e concedere i permessi necessari. In alcuni casi, per le limitazioni del client condiviso, può servire creare **un proprio client OAuth Desktop** in Google Cloud, abilitare la Google Drive API e inserire client ID e secret nell'installazione locale di rclone.
4. Verificare che `rclone lsd google:` (o un comando equivalente) elenchi le cartelle del Drive autorizzato.
5. Individuare il file `rclone.conf` creato sul PC e copiarlo con File Editor nella configurazione di Home Assistant, come `rclone.conf`.

**Attenzione alla sicurezza:** `rclone.conf` può contenere token OAuth, client secret e informazioni sufficienti per accedere al Drive. **Non** pubblicarlo, non allegarlo a una guida, non copiarne il contenuto in chat pubbliche, GitHub, screenshot o log condivisi. Limitare i permessi sul file. Se un secret è stato esposto, revocare e rigenerare le credenziali.

### 4.3 Differenze fra i percorsi

Nella nostra installazione il medesimo file di configurazione è visibile con percorsi diversi:

| Contesto | Percorso |
|---|---|
| Home Assistant / File Editor | `/config/rclone.conf` |
| Add-on Rclone Backup | `/homeassistant/rclone.conf` |
| Home Assistant / File Editor, cartella FIT | `/config/garmin_activities` |
| Add-on Rclone Backup, cartella FIT | `/homeassistant/garmin_activities` |

Non confondere il percorso interno di Home Assistant con quello che vede il contenitore dell'add-on. Verificare sempre i mount esposti dalla propria versione dell'add-on.

## 5. Installare Rclone Backup

1. In **Impostazioni → App/Add-on**, aggiungere il repository `https://github.com/jcwillox/hassio-rclone-backup` se non già presente.
2. Installare **Rclone Backup**, avviarlo e consultare la sua pagina **Configurazione**.
3. Impostare `config_path` a `/homeassistant/rclone.conf` (nel nostro caso).
4. Lasciare `dry_run: true` solo durante un primo test innocuo, poi **disattivarlo**: se `dry_run` resta attivo il file non verrà caricato.
5. Lasciare `run_once: false` per mantenere in servizio anche il job programmato.
6. Verificare dall'interfaccia dell'add-on o dai log che il remote `google` sia utilizzabile.

### Primo job: recupero periodico di sicurezza

Creare un job con questi campi:

| Campo | Valore |
|---|---|
| `name` | `Garmin FIT to Drive` |
| `schedule` | `*/15 * * * *` |
| `command` | `copy` |
| `source` | `/homeassistant/garmin_activities` |
| `destination` | `google:Garmin` |
| `flags` / filtri | vuoti, salvo necessità specifiche |

`*/15 * * * *` significa ogni 15 minuti. Usare **`copy`** e non `sync`: `copy` aggiunge e aggiorna i file di destinazione senza cancellare quelli remoti solo perché sono spariti dalla cartella locale.

Eseguire un test manuale del job con `dry_run: true`. Il log può riportare che il file è stato saltato a causa di `--dry-run`: è normale. Poi disattivare `dry_run` e verificare che un FIT compaia davvero in Google Drive.

### Secondo job: caricamento immediato dopo un nuovo FIT

Aggiungere un altro job:

| Campo | Valore |
|---|---|
| `name` | `Garmin FIT immediato` |
| `schedule` | **vuoto / non definito** |
| `command` | `copy` |
| `source` | `/homeassistant/garmin_activities` |
| `destination` | `google:Garmin` |
| `flags` / filtri | vuoti, salvo necessità specifiche |

La documentazione dell'add-on spiega che un job **senza `schedule` parte all'avvio dell'add-on**. Perciò, riavviando Rclone Backup, si ottiene una copia su richiesta senza configurare la sua API Remote Control.

Esempio **illustrativo** di configurazione YAML equivalente ai campi, da confrontare con l'editor e lo schema della propria versione:

```yaml
jobs:
  - name: Garmin FIT to Drive
    schedule: '*/15 * * * *'
    command: copy
    source: /homeassistant/garmin_activities
    destination: 'google:Garmin'
    flags: {}
  - name: Garmin FIT immediato
    command: copy
    source: /homeassistant/garmin_activities
    destination: 'google:Garmin'
    flags: {}
dry_run: false
run_once: false
config_path: /homeassistant/rclone.conf
```

**Importante:** non abilitare `run_once` nel nostro schema misto. La documentazione lo descrive come modalità per eseguire i job senza pianificazione e poi terminare l'add-on; ciò contrasterebbe con la permanenza attiva del job di sicurezza.

## 6. Individuare l'azione Home Assistant per riavviare Rclone Backup

Aprire **Strumenti per sviluppatori → Azioni** e selezionare:

```text
hassio.addon_restart
```

Selezionare Rclone Backup nella casella `addon`, poi passare alla **modalità YAML** per leggere l'identificativo preciso dell'add-on (*slug*). Nella nostra installazione era:

```text
19a172aa_rclone_backup
```

**Questo ID è specifico del repository/installazione: non copiarlo alla cieca**. Per una nuova installazione usare sempre quello indicato dall'interfaccia locale.

Un test manuale dell'azione può avere la forma:

```yaml
action: hassio.addon_restart
data:
  addon: 19a172aa_rclone_backup  # sostituire con lo slug effettivo
```

Controllare il **Registro** dell'add-on: un esempio di esito corretto è:

```text
[INFO]: running "Garmin FIT immediato"; /homeassistant/garmin_activities -> google:Garmin
INFO  : There was nothing to transfer
[INFO]: finished in 904ms
```

`There was nothing to transfer` è **successo**, se i file erano già sincronizzati. Se la copia è già in corso, il riavvio potrebbe interromperla: questo metodo è semplice ma non equivalente a una chiamata API transazionale.

## 7. Automazione completa di Home Assistant

Da **Impostazioni → Automazioni e scene**, creare un'automazione in modalità YAML o modificare quella esistente:

```yaml
alias: Garmin - Download automatico FIT
description: Scarica i FIT Garmin e li carica subito su Google Drive
triggers:
  - trigger: state
    entity_id: sensor.garmin_connect_last_activity
    attribute: activityId
conditions: []
actions:
  - variables:
      activity_id: >
        {{ state_attr('sensor.garmin_connect_last_activity',
                      'activityId') | int(0) }}

  - condition: template
    value_template: "{{ activity_id | int > 0 }}"

  - action: garmin_connect.download_activity
    data:
      activity_id: "{{ activity_id }}"
      file_format: fit

  - delay:
      seconds: 5

  - action: hassio.addon_restart
    data:
      addon: 19a172aa_rclone_backup  # VERIFICARE IL PROPRIO SLUG
mode: queued
max: 10
```

**Cosa fa:**

1. Il sensore Garmin aggiorna il suo attributo `activityId` quando viene rilevata una nuova attività.
2. L'automazione legge l'ID e scarta valori mancanti o non validi.
3. Esegue il download FIT nella cartella predefinita.
4. Attende 5 secondi come margine di sicurezza.
5. Riavvia l'add-on rclone, che all'avvio lancia il job senza `schedule`.
6. Rclone usa `copy` per trasferire solo ciò che manca o differisce su Drive.

`mode: queued` accoda attivazioni multiple invece di interrompere un'istanza in corso. Con alcune integrazioni, l'aggiornamento del sensore potrebbe non essere istantaneo; il job ogni 15 minuti resta una protezione aggiuntiva **solo se il FIT è stato già scaricato localmente**. Non supplisce a un download mancato da Garmin.

**Miglioramento futuro opzionale:** l'integrazione Garmin supporta `response_variable` sull'azione di download, con percorso e dimensioni del file. Un controllo basato su tale risposta può essere più robusto del solo ritardo di 5 secondi, qualora l'azione o il file system presentassero latenze particolari.

## 8. Collaudo in due fasi

### Fase A — Test manuale dell'automazione

1. Salvare l'automazione.
2. Scegliere **Esegui azioni** dalla pagina dell'automazione.
3. Controllare le tracce dell'automazione: l'azione Garmin e `hassio.addon_restart` non devono generare errori.
4. Verificare nel **Registro** di Rclone Backup che compaia `running "Garmin FIT immediato"` con l'ora del test.
5. Se l'ultimo FIT è già su Drive, è normale leggere `There was nothing to transfer`.

**Questo collauda la catena delle azioni, non il trigger su una nuova attività.**

### Fase B — Test completo automatico

1. Sul Garmin registrare un'attività breve, anche di circa un minuto.
2. Salvarla sull'orologio e sincronizzare Garmin Connect.
3. **Non** eseguire manualmente l'automazione né il job rclone.
4. Attendere il tempo necessario al rilevamento Garmin.
5. Verificare che sia comparso in `/config/garmin_activities` un nuovo file `activity_<nuovo_id>.fit`.
6. Verificare nel registro rclone che il job immediato sia partito dopo il download.
7. Verificare in Google Drive che esista **il nuovo ID**, non soltanto la vecchia attività ritoccata o risincronizzata.
8. Se si dispone di un decoder FIT, controllare data interna, sport, durata e campioni.

**Risultato del collaudo reale, 22 ottobre 2026:** è arrivato un nuovo `activity_24813451231.fit` da **7.491 byte**, relativo a una breve attività di test di circa un minuto, senza lanciare manualmente il trasferimento. L'attività era fittizia e non va conteggiata nello storico sportivo.

La data **`modified` su Google Drive non corrisponde necessariamente alla data della registrazione Garmin**: per identificarla usare l'ID Garmin e, se necessario, decodificare i metadati interni del FIT.

## 9. Risoluzione dei problemi

| Problema | Verifica / soluzione |
|---|---|
| Non appare `sensor.garmin_connect_last_activity` | Verificare login, entità abilitate, refresh dell'integrazione e nome effettivo del sensore. |
| Cambia l'attività su Garmin ma l'automazione non parte | Controllare attributo `activityId` negli Stati e le tracce dell'automazione. |
| Eseguendo manualmente l'automazione l'ID è assente | Usare la lettura dell'ID dal sensore, come nell'esempio; in esecuzione manuale non è disponibile un normale evento `trigger.to_state`. |
| Download FIT fallisce | Verificare autenticazione Garmin, disponibilità del formato, percorso e permessi della cartella. |
| `which rclone` nel Terminal & SSH non restituisce nulla | Normale: Terminal & SSH e Rclone Backup sono contenitori separati; rclone può esistere solo nell'add-on. |
| `ha apps list` non restituisce informazioni utili | Non è necessario per questa procedura: il riavvio avviene tramite `hassio.addon_restart`. |
| Rclone indica `--dry-run` o `Skipped copy` | Disattivare `dry_run` dopo il test: con dry run non carica realmente nulla. |
| Rclone scrive `There was nothing to transfer` | Nessun file nuovo o modificato da copiare: comportamento normale. |
| Log rclone mostra errore di credenziali | Ricontrollare `rclone.conf`, token OAuth e permessi Drive; se necessario rifare l'autorizzazione. |
| Job immediato non parte al riavvio | Verificare che il **secondo job abbia schedule vuoto**, e leggere il Registro dell'add-on. |
| `hassio.addon_restart` non funziona | Verificare slug locale e permessi dell'automazione; controllare le tracce. |
| La copia su Drive arriva in ritardo | Distinguere il ritardo Garmin/HA dal tempo rclone e dall'aggiornamento dell'elenco Drive. |
| Il file su Drive esiste ma ha una data `modified` recente | Controllare l'**ID nel nome** e la data interna del FIT: potrebbe trattarsi di una vecchia attività. |
| Attività di test da cancellare | Cancellarla in Garmin Connect e, se necessario, anche nelle cartelle HA e Drive: `copy` non propaga le cancellazioni. |
| Il job ogni 15 minuti lavora inutilmente | Di norma il confronto su pochi FIT è leggero, ma è un compromesso per avere una rete di sicurezza. Si può aumentare l'intervallo o eliminarlo dopo adeguate verifiche. |

## 10. Sicurezza, privacy e manutenzione

- **Non esporre l'API Remote Control di rclone**, specialmente se l'interfaccia Backend mostra `RC Enabled: true` e `NoAuth: true`. Questa guida **non richiede** accessi RC dalla LAN o da Internet.
- Il file `rclone.conf` è un segreto operativo: non condividerlo e non inserirlo in documentazione pubblica.
- I file FIT possono contenere **posizione GPS, orari, battito e altri dati personali**. La cartella Drive deve rimanere privata salvo decisione esplicita.
- **`copy` non cancella i file remoti**. È voluto; significa che eliminare un'attività su Garmin non la cancella automaticamente da Drive.
- Il riavvio dell'add-on può interrompere altri job contemporanei. Evitare di concentrare nello stesso add-on grandi trasferimenti critici senza esaminare questo comportamento.
- Ricontrollare dopo aggiornamenti dell'integrazione o dell'add-on: entità, nomi delle azioni e schemi YAML possono cambiare.
- Mantenere una copia di sicurezza della configurazione HA, separata dal file segreto `rclone.conf`, e una procedura sicura per ripristinare le credenziali.

## 11. Limitazioni e miglioramenti possibili

Questa soluzione è **event-driven per l'avvio della copia**, ma usa il riavvio dell'add-on e non una vera chiamata `rclone rc`. È stata scelta perché si integra con l'azione Home Assistant già disponibile, senza dover esporre un endpoint RC non autenticato.

Possibili evoluzioni:

- Attivare una copia mirata al **solo FIT appena scaricato**, se l'ambiente e l'add-on lo permetteranno.
- Usare una risposta verificata del download Garmin invece di affidarsi soltanto ai 5 secondi di attesa.
- Sostituire il riavvio dell'add-on con una chiamata controllata a un servizio autenticato, qualora si scelga un'architettura rclone RC sicura.
- Elaborare i FIT in JSON per alimentare una dashboard HTML portabile, senza esporre i file GPS originali.
- Mantenere nella cartella Drive anche un diario Markdown di analisi, separato dai file grezzi.

## 12. Checklist finale

- [ ] Garmin Connect sincronizza le attività dall'orologio.
- [ ] In Home Assistant esiste un sensore con attributo `activityId`.
- [ ] `garmin_connect.download_activity` crea un vero `.fit` nella cartella locale.
- [ ] L'add-on rclone riconosce `google:` e raggiunge la cartella desiderata.
- [ ] `dry_run` è `false` e `run_once` è `false`.
- [ ] Job `Garmin FIT to Drive` pianificato ogni 15 minuti.
- [ ] Job `Garmin FIT immediato` **senza schedule**.
- [ ] `hassio.addon_restart` usa lo **slug verificato localmente**.
- [ ] L'automazione completa è salvata e attiva.
- [ ] Una **nuova attività** produce autonomamente un **nuovo file FIT su Drive**.
- [ ] Credenziali e FIT rimangono protetti.

---

*Questa guida documenta una configurazione effettivamente collaudata il 22 ottobre 2026. Gli esempi sono pensati per essere adattati, non per essere incollati senza verificare gli identificativi e i percorsi della propria installazione.*
