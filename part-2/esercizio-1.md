## Esercizio 1 — Cluster PostgreSQL in produzione: connessione, configurazione e alta disponibilità

**Scenario:** sei il DBA responsabile di una nuova applicazione che userà PostgreSQL. Devi costruire l'intero ambiente da zero, seguendo le best practice di configurazione, esponendolo correttamente, e verificando che l'alta disponibilità funzioni davvero.

### Requisiti infrastrutturali

Crea un cluster PostgreSQL con CloudNativePG che rispetti questi requisiti:

* 3 istanze (1 Primary + 2 Replica), con `walStorage` su PVC separato dai dati
* QoS Guaranteed: `requests` e `limits` di CPU/memoria identici
* `shared_buffers`, `effective_cache_size` e `work_mem` calcolati sulla RAM del **container** (documenta il calcolo che hai fatto, non solo il valore finale)
* `max_connections` impostato assumendo che ci sarà un PgBouncer davanti (vedi sotto)
* Pod anti-affinity `required`, non il default `preferred` — le tre istanze non devono mai poter finire sullo stesso nodo
* PodDisruptionBudget con `minAvailable: 2`
* Liveness e Readiness Probe calibrate (non i valori di default troppo aggressivi — motiva i valori scelti)
* TLS abilitato e obbligatorio — nessuna connessione in chiaro deve essere accettata

### Requisiti di connessione

* Crea un `Pooler` (PgBouncer) in modalità `transaction`
* Deploya un Pod `nginx` che rappresenta il backend applicativo — deve avere il label corretto, il Secret e il ConfigMap montati come variabili d'ambiente, così da rappresentare "l'identità" di un'applicazione che si collegherebbe al database tramite il Pooler, non direttamente al Service `-rw`
> 💡 `nginx` non ha `psql` né sa parlare il protocollo Postgres — non può eseguire query reali. Il suo ruolo qui è dimostrare il plumbing Kubernetes (Secret, ConfigMap, label per la NetworkPolicy, esposizione via Ingress). Per verificare la connessione SQL vera, usa un Pod temporaneo con l'immagine `postgres` **e lo stesso label del backend** (`--labels="app=backend"`) — è il Pod temporaneo, non nginx, a dimostrare che una richiesta con l'identità giusta arriva al database
* La connection string (host, credenziali, `sslmode=verify-full`, `application_name`) va in un Secret — mai hardcoded
* La configurazione non sensibile (nome database, nome host) va in un ConfigMap separato
* Esponi un piccolo endpoint del backend tramite Ingress, raggiungibile a `backend-<namespace>.corso.local`

### Requisiti di sicurezza minimi

* Una NetworkPolicy che permetta **solo** al Pod backend di raggiungere il database sulla porta 5432 (pattern AND: label + namespace)
* Il Pod backend usa un ServiceAccount dedicato, con `automountServiceAccountToken: false`
* Un `SecurityContext` appropriato sul Pod backend (`runAsNonRoot: true`, niente privilegi non necessari)

> ⚠️ Questa impostazione va applicata **solo** al ServiceAccount del Pod backend — mai al ServiceAccount che CNPG crea automaticamente per il cluster Postgres. Il Postgres gestito da CNPG (in realtà il suo instance manager) ha bisogno di un token valido per parlare con l'API server e coordinare lo stato del cluster: togliendoglielo il Pod database non supera più la startup probe e resta in CrashLoopBackOff

### Requisiti di Alta Disponibilità

* **Failover:** elimina il Pod Primary con `kubectl delete pod`, misura il tempo impiegato dal nuovo Primary a diventare Ready, verifica che il Service `-rw` punti al nuovo Primary
* **Switchover pianificato:** usa il meccanismo dichiarativo dell'operator per promuovere una Replica specifica senza eliminare nulla — verifica che non ci sia stata perdita di dati
* **Anti-affinity:** verifica con `kubectl get pods -o wide` che le 3 istanze siano davvero su nodi diversi

### Verifiche finali

* `SELECT pg_is_in_recovery();` restituisce `f` sul Primary e `t` sulle Replica
* Una connessione con `sslmode=disable` viene esplicitamente rifiutata
* Il backend è raggiungibile da `backend-<namespace>.corso.local`
* Un Pod senza il label corretto non riesce a raggiungere il database (timeout, non connection refused)
* Il tempo di failover misurato è coerente con quanto atteso dalle slide (15-60 secondi)