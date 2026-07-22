## Esercizio 2 - ReplicaSet MongoDB con Percona Operator

**Scenario:** un terzo team usa MongoDB per dati non strutturati. Devi configurare un ReplicaSet completo con Percona Operator for MongoDB.

### Requisiti infrastrutturali

Installa Percona Operator for MongoDB nel tuo namespace e crea un ReplicaSet con questi requisiti:

* 3 membri (1 Primary + 2 Secondary)
* QoS Guaranteed
* Un PVC dedicato per ogni membro
* Credenziali gestite tramite Secret con tutti gli utenti richiesti dall'Operator

### Requisiti applicativi

Simula un'applicazione che usa MongoDB:

* Deploya un Pod `mongo:6.0` come client applicativo — questa immagine include `mongosh`, quindi qui la verifica di connessione reale è possibile direttamente da questo Pod, senza bisogno di un Pod temporaneo separato
* Il Pod deve connettersi al ReplicaSet usando la connection string che sfrutti l'headless service. 
* La connection string deve essere in un Secret
* Configura `readPreference=secondaryPreferred` e `w=majority`
* Il Pod client deve avere un Init Container che aspetta che il ReplicaSet sia disponibile prima di avviarsi — l'Init Container deve usare un'immagine con gli strumenti necessari per verificare la connessione (es. `mongo:6.0` stessa, o `busybox` se verifichi solo la raggiungibilità TCP con `nc`)
* Deploya un Pod `nginx` come frontend e esponilo tramite Ingress a `mongo-frontend-<namespace>.corso.local`

### Requisiti di sicurezza

* NetworkPolicy che permette solo al Pod client di connettersi al database sulla porta 27017
* Il Pod frontend deve raggiungere il Pod client ma non il database direttamente
* Applica un SecurityContext appropriato al Pod client
* ServiceAccount dedicato per il Pod client senza token API montato

> 💡 A differenza di CNPG, qui il Pod "client" è un'applicazione a sé (non il database gestito dall'operator) — quindi togliergli il token API è corretto e sicuro: non ha alcuna ragione di parlare con l'API di Kubernetes. Il ServiceAccount usato dai Pod del ReplicaSet MongoDB gestiti dall'operator, invece, non va toccato con questa impostazione, per lo stesso motivo visto per CNPG nell'Esercizio 1.

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
