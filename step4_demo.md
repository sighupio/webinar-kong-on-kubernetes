# Step 4 - Installiamo l'app Petstore

Bene, cominciamo ora a creare la nostra applicazione di demo.

Posizioniamoci nella directory:

```bash
cd demo
```

Creiamo quindi il nostro namespace `petstore`:

```bash
kubectl apply -f 01.namespace.yml
```

Creiamo il nostro deployment, utilizzando l'immagine `swaggerapi/petstore:1.0.5`

```bash
kubectl apply -f 02.deployment.yml
```

Prima di esporre la nostra applicazione, vediamo come si presenta connettendoci direttamente al pod:

```bash
kubectl -n petstore port-forward $(kubectl get pods -n petstore | tail -n 1 | cut -d ' ' -f 1) 9000:8080 --address 0.0.0.0
```

Apriamo quindi la dashboard debug e posizioniamoci al path `/api/swagger.json`



http://localhost:9000/api/swagger.json

Creiamo quindi un service collegato ai nostri pod del deployment:

```bash
kubectl apply -f 03.service-1.yml
```

La nostra applicazione è ora pronta per essere collegata dall'esterno tramite `Ingress`

[Step 5 - Esponiamo Petstore tramite Ingress](step5_demo.md)
