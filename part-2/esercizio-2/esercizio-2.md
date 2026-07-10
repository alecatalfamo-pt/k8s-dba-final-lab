## Esercizio 2 — Troubleshooting: un cluster MySQL ereditato pieno di anti-pattern

**Scenario:** sei subentrato come DBA su un cluster Percona XtraDB Cluster lasciato da un collega che ha ignorato ogni best practice di configurazione. Verrà fornito il manifest del cluster così com'è — **non sai in anticipo quali problemi contenga**. Il cluster ha diversi problemi, alcuni attivi (Pod non partono, applicazione non si connette) e altri latenti (configurazione scorretta che esploderà al primo carico serio). Il tuo compito è applicare un metodo rigoroso per diagnosticare ciascun problema, e le linee guida corrette per risolverlo.

### Il cluster di partenza

Applica il manifest fornito (`cluster-problematico-esercizio-2.yaml`) nel tuo namespace:

```bash
kubectl apply -f cluster-problematico-esercizio-2.yaml -n <il-tuo-namespace>
```

Non guardare il contenuto del file riga per riga prima di iniziare — trattalo come troveresti un cluster reale lasciato da un collega: parti osservando i sintomi (`kubectl get pods`, stato dell'applicazione), non leggendo la configurazione a priori.

### Fase di diagnosi

Il cluster nasconde diversi problemi — alcuni si manifestano subito (Pod Pending, CrashLoopBackOff, OOMKill), altri restano silenziosi finché non provi a connetterti o non ispezioni la configurazione con attenzione. Non fermarti al primo problema che risolvi: quando pensi di aver finito, ricontrolla `kubectl get pods`, `kubectl describe`, la configurazione applicativa e le policy di rete — potrebbero esserci più problemi indipendenti tra loro.

Per ogni problema che trovi:

1. Segui l'approccio metodico: stato Pod → eventi → log (incluso `--previous` dove serve) → risorse correlate
2. Documenta il comando esatto che ti ha rivelato la causa e l'output ottenuto
3. Distingui esplicitamente, per il problema di rete, se il sintomo è un timeout o un connection refused, e cosa questo implica

### Fase di correzione

Per ogni problema che hai diagnosticato, applica il fix corretto secondo le best practice — usa questa come lista di verifica generale per un cluster "ereditato", non come indicazione di cosa sia effettivamente rotto in questo specifico cluster:

```
□ I parametri di memoria (buffer pool, cache) sono calcolati sui
  limits.memory del container, non sulla RAM del nodo?
□ I timeout di connessione sono ragionevoli per un ambiente K8s
  dove i Pod possono sparire in qualsiasi momento?
□ Il Pod ha QoS Guaranteed (requests = limits)?
□ Le probe sono calibrate per tollerare un carico normale senza
  riavvii inutili?
□ Le credenziali vivono in un Secret, mai in un ConfigMap o hardcoded?
□ Esiste un PodDisruptionBudget coerente con il numero di istanze?
□ Le NetworkPolicy permettono tutto il traffico necessario
  (incluso il DNS in uscita, se serve)?
□ Le StorageClass referenziate esistono davvero nel cluster?
```

### Verifiche finali

* Tutti i Pod del cluster sono `Running` e pronti (3/3 o equivalente)
* `kubectl describe pod` mostra QoS Guaranteed
* Il PDB esiste con `ALLOWED DISRUPTIONS` coerente con il numero di istanze
* Il Pod applicativo (nginx) si connette correttamente al cluster MySQL
* `SHOW REPLICA STATUS\G` (o l'equivalente per PXC) mostra `Seconds_Behind_Source: 0`
* Nessuna credenziale è più visibile in chiaro in un ConfigMap (`kubectl get configmap -o yaml`)
