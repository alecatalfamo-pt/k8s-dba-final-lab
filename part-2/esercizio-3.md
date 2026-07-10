## Esercizio 3 — Migrazione, Disaster Recovery e Backup per MongoDB

**Scenario:** l'azienda ha un MongoDB standalone "storico", fuori da Kubernetes (lo simuliamo con un Pod `mongo:6.0` a sé stante nel tuo namespace), che va migrato su Percona Server for MongoDB. Il nuovo cluster deve avere anche un sito di Disaster Recovery pronto e i backup configurati — tutto dentro il tuo unico namespace, usando risorse con nomi diversi per simulare i due "siti".

### Fase 1 — Il sistema legacy

```bash
# Deploya un Pod standalone che rappresenta il vecchio sistema
kubectl run mongo-legacy --image=mongo:6.0 --restart=Never
```

Popolalo con almeno una collection e una decina di documenti di esempio — questi sono "i dati di produzione" che dovrai migrare.

### Fase 2 — Migrazione verso PSMDB

* Deploya un cluster `PerconaServerMongoDB` (chiamalo `mongo-main`) nel tuo namespace
* Migra i dati dal Pod legacy al nuovo cluster usando `mongodump`/`mongorestore`
* Verifica che il conteggio dei documenti coincida tra sorgente e destinazione

### Fase 3 — Disaster Recovery, stesso namespace

* Deploya un secondo `PerconaServerMongoDB` (chiamalo `mongo-dr`) con `unmanaged: true`
* Configura `externalNodes` su entrambe le risorse in modo che `mongo-dr` entri come membro del replica set di `mongo-main`, con `priority: 0` e `votes: 0`
* Verifica con `rs.status()` sul Primary di `mongo-main` che i membri di `mongo-dr` compaiano come `SECONDARY`, non eleggibili

### Fase 4 — Backup

* Configura il backup su `mongo-main` puntato al MinIO condiviso creato nel namespace shared-storage.
* Esegui un backup on-demand e verifica che completi con successo
* Se il tempo lo permette, abilita anche il PITR (`oplogSpanMin` basso) e verifica che i chunk di oplog arrivino su MinIO

### Fase 5 — Simulazione del disastro

* Genera nuovi dati su `mongo-main`
* Simula un disastro: elimina (o rendi irraggiungibile) il Primary di `mongo-main`
* Promuovi `mongo-dr` (`unmanaged: false`), alzando se necessario `priority`/`votes` dei suoi membri
* Verifica che `mongo-dr` accetti scritture dopo la promozione

### Verifiche finali

* I documenti migrati dal sistema legacy sono presenti e corretti su `mongo-main`
* `rs.status()` mostra la topologia attesa prima della promozione (membri DR non eleggibili)
* Il backup risulta `ready`/`completed` su MinIO (verificabile anche con `mc ls`)
* Dopo la promozione, `mongo-dr` accetta scritture reali
* Sai spiegare, a voce, perché questa NON è una simulazione di un vero DR (stesso cluster K8s sottostante) e cosa cambierebbe con due cluster K8s reali
