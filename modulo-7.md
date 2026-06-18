# Laboratorio Finale — Modulo 7
## Kubernetes per il DBA
### Ambiente: cluster Rancher, namespace dedicato per studente

---

## Istruzioni generali

Ogni esercizio è uno scenario completo e indipendente. Leggi tutti i requisiti prima di iniziare — alcuni si costruiscono sulle fondamenta dei precedenti.

Usa solo immagini ufficiali — `postgres:15`, `mysql:8.0`, `mongo:6.0`, `nginx`, `busybox`, ecc. Nessuna immagine custom.

Il formato per gli Ingress è `<nome-servizio>-<namespace>.corso.local`.

Documenta ogni step con i comandi usati e l'output ottenuto.

---

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

* Deploya un Pod `nginx` che rappresenta il backend applicativo
* Il Pod deve connettersi al cluster PostgreSQL tramite il Service corretto
* Le credenziali del database devono essere gestite tramite Secret — non hardcoded
* La configurazione non sensibile (nome database, host) deve essere in un ConfigMap
* Esponi il backend tramite un Ingress raggiungibile a `backend-<namespace>.corso.local`

### Requisiti di sicurezza

* Il Pod backend deve poter raggiungere il database sulla porta 5432
* Nessun altro Pod nel namespace deve poter raggiungere il database
* Il Pod database deve avere un ServiceAccount dedicato senza token API montato
* Applica un SecurityContext appropriato al Pod backend

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
* Verifica che il Pod backend raggiunga nginx tramite Ingress da `backend-<namespace>.corso.local`
* Verifica che un Pod senza il label corretto non riesca a connettersi al database
* Verifica che gli alert siano visibili in Prometheus UI → Alerts

---

## Esercizio 2 — Stack MySQL con Percona Operator

**Scenario:** un secondo team della tua azienda usa MySQL. Devi installare Percona Operator nel tuo namespace e configurare un cluster MySQL completo.

### Requisiti infrastrutturali

Installa Percona Operator for MySQL nel tuo namespace e crea un cluster con questi requisiti:

* 3 istanze con Group Replication
* QoS Guaranteed
* PVC dedicati per dati e binary log — usa StorageClass separate se disponibili
* Credenziali gestite tramite Secret

### Requisiti applicativi

Simula un'applicazione che usa MySQL in lettura e scrittura:

* Deploya due Pod `busybox` — uno che simula scritture sul Primary, uno che simula letture sulla Replica
* Il Pod di scrittura deve connettersi al Service Primary
* Il Pod di lettura deve connettersi al Service Replica
* Entrambi devono leggere le credenziali da Secret
* Deploya un Pod `nginx` come frontend e esponilo tramite Ingress a `frontend-<namespace>.corso.local`

### Requisiti di sicurezza

* NetworkPolicy che permette ai due Pod applicativi di connettersi al database sulla porta 3306
* Nessun altro Pod deve raggiungere il database
* I due Pod applicativi non devono potersi raggiungere tra loro
* Il Pod frontend deve poter raggiungere i Pod applicativi ma non il database direttamente

### Requisiti di monitoring

* Installa mysql_exporter e configuralo per scrapare il cluster MySQL
* Crea un ServiceMonitor per il mysql_exporter con il label corretto per kube-prometheus-stack
* Verifica che le metriche `mysql_*` siano disponibili in Prometheus UI
* Crea una PrometheusRule con almeno due alert:
  * Replication lag superiore a 30 secondi
  * Connessioni attive superiori a 100
* Importa la dashboard MySQL (ID 7362) in Grafana e verifica che mostri le metriche del tuo cluster
* Verifica in Grafana che il buffer pool hit ratio sia sopra il 95%

### Verifiche finali

* Verifica lo stato del Group Replication connettendoti al Primary:
  ```sql
  SELECT * FROM performance_schema.replication_group_members;
  ```
* Verifica che il Pod di lettura sia connesso alla Replica e non al Primary
* Verifica che i due Pod applicativi non possano comunicare tra loro
* Verifica che il frontend sia raggiungibile tramite Ingress
* Verifica che il frontend non possa raggiungere direttamente il database
* Verifica le metriche MySQL in Prometheus UI e Grafana

---

## Esercizio 3 — ReplicaSet MongoDB con Percona Operator

**Scenario:** un terzo team usa MongoDB per dati non strutturati. Devi configurare un ReplicaSet completo con Percona Operator for MongoDB.

### Requisiti infrastrutturali

Installa Percona Operator for MongoDB nel tuo namespace e crea un ReplicaSet con questi requisiti:

* 3 membri (1 Primary + 2 Secondary)
* QoS Guaranteed
* Un PVC dedicato per ogni membro
* Credenziali gestite tramite Secret con tutti gli utenti richiesti dall'Operator

### Requisiti applicativi

Simula un'applicazione che usa MongoDB:

* Deploya un Pod `mongo:6.0` come client applicativo
* Il Pod deve connettersi al ReplicaSet usando la connection string completa con tutti i membri
* La connection string deve essere in un Secret
* Configura `readPreference=secondaryPreferred` e `w=majority`
* Il Pod client deve avere un Init Container che aspetta che il ReplicaSet sia disponibile prima di avviarsi
* Deploya un Pod `nginx` come frontend e esponilo tramite Ingress a `mongo-frontend-<namespace>.corso.local`

### Requisiti di sicurezza

* NetworkPolicy che permette solo al Pod client di connettersi al database sulla porta 27017
* Il Pod frontend deve raggiungere il Pod client ma non il database direttamente
* Applica un SecurityContext appropriato al Pod client
* ServiceAccount dedicato per il Pod client senza token API montato

### Requisiti di monitoring

* Installa mongodb_exporter e configuralo per scrapare il ReplicaSet
* Crea un ServiceMonitor con il label corretto per kube-prometheus-stack
* Verifica che le metriche `mongodb_*` siano disponibili in Prometheus UI
* Crea una PrometheusRule con almeno due alert:
  * Connessioni attive superiori a 50
  * Membro del ReplicaSet non in stato PRIMARY o SECONDARY
* Importa la dashboard MongoDB (ID 2583) in Grafana
* Crea un pannello custom in Grafana che mostri il numero di operazioni al secondo per tipo (insert, query, update, delete)

### Verifiche finali

* Verifica lo stato del ReplicaSet dall'interno del Pod client:
  ```javascript
  rs.status()
  ```
* Verifica che le scritture vadano sul Primary e le letture sui Secondary
* Verifica che l'Init Container abbia aspettato correttamente prima dell'avvio del Pod client
* Verifica che il frontend sia raggiungibile tramite Ingress
* Verifica che il frontend non possa raggiungere direttamente MongoDB
* Verifica le metriche MongoDB in Prometheus UI e nella dashboard Grafana

---

## Esercizio 4 — Scenario completo multi-database

**Scenario:** costruisci un ambiente completo che simula un'applicazione reale con tre tier — frontend, backend e database. L'applicazione usa PostgreSQL come database principale e MongoDB come store per dati non strutturati.

### Architettura richiesta

```
Internet
    ↓
[Ingress Controller]
    ↓ frontend-<namespace>.corso.local
[Frontend - nginx]
    ↓ porta 80
[Backend - nginx]
    ↓                         ↓
[PostgreSQL - CNPG]      [MongoDB - Percona]
```

### Requisiti del tier frontend

* Deploya un Pod `nginx` come frontend
* Esponi il frontend tramite Ingress a `frontend-<namespace>.corso.local`
* Deve poter raggiungere il backend sulla porta 80
* Non deve poter raggiungere direttamente i database
* Deve avere un ConfigMap con la configurazione nginx

### Requisiti del tier backend

* Deploya un Pod `nginx` come backend — non esposto tramite Ingress
* Deve poter raggiungere PostgreSQL sulla porta 5432
* Deve poter raggiungere MongoDB sulla porta 27017
* Deve essere raggiungibile dal frontend solo sulla porta 80
* Le credenziali di entrambi i database devono essere in Secret separati
* La configurazione non sensibile deve essere in ConfigMap
* Deve avere un Init Container che verifica la disponibilità di entrambi i database prima di avviarsi
* ServiceAccount dedicato senza token API montato

### Requisiti PostgreSQL

* Cluster CloudNativePG con 2 istanze
* QoS Guaranteed
* TLS obbligatorio
* WAL su PVC separato
* ServiceAccount dedicato senza token API montato

### Requisiti MongoDB

* ReplicaSet Percona con 3 membri
* QoS Guaranteed
* Un PVC per membro
* Credenziali in Secret

### Requisiti di sicurezza complessivi

* NetworkPolicy che implementa esattamente l'architettura descritta:
  * Frontend → Backend (porta 80) ✅
  * Frontend → PostgreSQL ❌
  * Frontend → MongoDB ❌
  * Backend → PostgreSQL (porta 5432) ✅
  * Backend → MongoDB (porta 27017) ✅
  * Nessun Pod → altri namespace ❌
* ServiceAccount dedicati per ogni tier senza token API montato
* SecurityContext appropriato su tutti i Pod

### Requisiti di monitoring

* postgres_exporter per PostgreSQL con ServiceMonitor
* mongodb_exporter per MongoDB con ServiceMonitor
* PrometheusRule con alert per entrambi i database:
  * Replication lag PostgreSQL > 30s
  * Connessioni MongoDB > 100
  * Memoria usata da qualsiasi Pod database oltre l'80% del limit
* Importa in Grafana le dashboard 9628 (PostgreSQL) e 2583 (MongoDB)
* Importa la dashboard K8s Pods (ID 6417) e filtrala per il tuo namespace
* Crea un pannello custom in Grafana che mostri su un unico grafico:
  * Connessioni attive PostgreSQL
  * Connessioni attive MongoDB

### Verifiche finali

* Verifica che il frontend sia raggiungibile tramite Ingress da `frontend-<namespace>.corso.local`
* Verifica che il frontend raggiunga il backend sulla porta 80
* Verifica che il frontend NON raggiunga direttamente PostgreSQL o MongoDB
* Verifica che il backend raggiunga entrambi i database
* Verifica che l'Init Container del backend abbia aspettato entrambi i database
* Verifica TLS su PostgreSQL — una connessione in chiaro deve essere rifiutata
* Verifica lo stato del ReplicaSet MongoDB da dentro il Pod backend
* Verifica che tutti gli alert siano visibili in Prometheus UI
* Verifica che le dashboard Grafana 9628 e 2583 mostrino metriche reali
* Verifica nel pannello custom che entrambe le metriche di connessione siano visibili