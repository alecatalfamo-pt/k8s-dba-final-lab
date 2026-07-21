## Esercizio 1 — Cluster PostgreSQL in Alta Disponibilità

**Scenario:** sei il DBA responsabile di un'applicazione web che usa PostgreSQL. Devi costruire un ambiente completo e sicuro nel tuo namespace.

### Requisiti infrastrutturali

Crea un cluster PostgreSQL con CloudNativePG che rispetti questi requisiti:

* 3 istanze (1 Primary + 2 Replica)
* QoS Guaranteed con risorse appropriate
* WAL su PVC separato
* TLS abilitato e obbligatorio — nessuna connessione in chiaro deve essere accettata
* Le istanze non devono mai stare sullo stesso nodo

### Requisiti applicativi

Simula un'applicazione backend che si connette al database:

* Deploya un Pod `nginx` che rappresenta il backend applicativo — deve avere il label corretto, il Secret e il ConfigMap montati come variabili d'ambiente, così da rappresentare "l'identità" di un'applicazione che si collegherebbe al database
* Le credenziali del database devono essere gestite tramite Secret — non hardcoded
* La configurazione non sensibile (nome database, host) deve essere in un ConfigMap
* Esponi il backend tramite un Ingress raggiungibile a `backend-<namespace>.corso.local`

### Requisiti di sicurezza

* Il Pod backend deve poter raggiungere il database sulla porta 5432
* Nessun altro Pod nel namespace deve poter raggiungere il database
* Il Pod **backend** deve avere un ServiceAccount dedicato senza token API montato
* Applica un SecurityContext appropriato al Pod backend

> ⚠️ Non applicare `automountServiceAccountToken: false` al ServiceAccount del cluster PostgreSQL gestito da CNPG — il suo instance manager deve poter parlare con l'API server per coordinare Primary/Replica e leggere lo stato del `Cluster`. Togliergli il token produce un CrashLoopBackOff (il processo non riesce più a verificare lo stato del cluster). Il ServiceAccount senza token va applicato solo al Pod applicativo (backend), che non ha alcuna ragione di parlare con l'API di Kubernetes.

### Requisiti di monitoring

* Installa postgres_exporter come Pod separato nel namespace
* Verifica che le metriche `pg_*` siano disponibili in Prometheus UI
* Crea una PrometheusRule con almeno due alert:
  * Replication lag superiore a 30 secondi
  * Numero di connessioni attive superiore a 50
* Importa la dashboard PostgreSQL (ID 9628) in Grafana e verifica che mostri le metriche del tuo cluster
* Crea un pannello custom in Grafana che mostri il replication lag nel tempo

### Verifiche finali

* Connettiti al Primary tramite il Service corretto e verifica con `SELECT pg_is_in_recovery();`
* Connettiti alla Replica tramite il Service read-only e verifica che sia in recovery
* Verifica che una connessione senza TLS venga rifiutata
* Verifica che il backend sia raggiungibile tramite Ingress da `backend-<namespace>.corso.local`
* Usa un Pod temporaneo con immagine `postgres:15` **e lo stesso label del backend** per verificare che una richiesta con quell'identità raggiunga davvero il database sulla porta 5432
* Usa un Pod temporaneo **senza** il label corretto per verificare che non riesca a connettersi al database (deve andare in timeout, non connection refused)
* Verifica che gli alert siano visibili in Prometheus UI → Alerts
