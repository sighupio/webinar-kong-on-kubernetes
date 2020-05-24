# Kong Ingress Controller - Webinar

Iniziamo scaricando le dipendenze tramite furyctl:

Posizioniamoci in 

```bash
cd /root/webinar-kong-on-kubernetes/
```

è già presente un file Furyfile.yml contenente 
le versioni dei pacchetti di KFD (Kubernetes Fury Distribution) che andremo ad utilizzare:

Furyfile.yml:
```yaml
bases:
  - name: monitoring/prometheus-operator
    version: "v1.6.1"
  - name: monitoring/prometheus-operated
    version: "v1.6.1"
  - name: monitoring/grafana
    version: "v1.7.0"
  - name: kong/kong
    version: "v1.0.0"
```

```bash
furyctl vendor --https
```

Procediamo quindi al prossimo step, installazione di prometheus e grafana

Nella root del progetto è già presente un file `kustomization.yaml` che useremo per installare tramite 
kustomize e kubectl i manifest di alcuni prerequisiti per la nostra demo:

* prometheus
* grafana

kustomization.yaml:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ./vendor/katalog/monitoring/prometheus-operator
  - ./vendor/katalog/monitoring/prometheus-operated
  - ./vendor/katalog/monitoring/grafana
#  - ./vendor/katalog/kong/kong

resources:
  - ./services.yml
```

Procediamo all'installazione tramite:

```bash
kustomize build . | kubectl apply -f -
```

Attendiamo che prometheus e grafana siano in stato `Running`

```bash
kubectl get pods -n monitoring
```

Procediamo quindi all'installazione del Kong Ingress Controller



Scommentiamo la riga `- ./vendor/katalog/kong/kong` al nostro file `kustomization.yaml` 

kustomization.yaml:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ./vendor/katalog/monitoring/prometheus-operator
  - ./vendor/katalog/monitoring/prometheus-operated
  - ./vendor/katalog/monitoring/grafana
  - ./vendor/katalog/kong/kong

resources:
  - ./services.yml
```

Procediamo all'installazione tramite:

```bash
kustomize build . | kubectl apply -f -
```

Attendiamo che il nostro kong ingress controller sia in stato running

```bash
kubectl get pods -n kong
```

Ora abbiamo il nostro Kong Ingress Controller pronto a servire traffico!

Procediamo quindi al prossimo step dove installeremo alcuni Ingress e configureremo un applicazione di prova



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

# Esponiamo il servizio

Creiamo quindi un service collegato ai nostri pod del deployment:

```bash
kubectl apply -f 03.service-1.yml
```

E creiamo poi la nostra ingress:

```bash
kubectl apply -f 04.public-ingress-1.yml
```

La nostra ingress è definita in questo modo 

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: public-petstore
  namespace: petstore
  annotations:
    kubernetes.io/ingress.class: 'kong'
spec:
  rules:
    - http:
        paths:
          - path: /pet
            backend:
              serviceName: petstore
              servicePort: 8080
```

Stiamo utilizzando l'ingress class `kong` e stiamo mappando il path `/pet` al nostro servizio petstore.

Proviamo ad accedere al path `/pet/1` tramite la nostra ingress...

http://yourip:31081/pet/1


Riceviamo degli errori 404, dato che cerchiamo di andare nel nostro pod al path `/pet/1` quando nel pod 
abbiamo una base path `api`.

# Kong CRD proxy

Utilizziamo una risorsa di Kong per effettuare questo proxy rewrite:

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: proxy
  namespace: petstore
  annotations:
    kubernetes.io/ingress.class: 'kong'
proxy:
  path: /api/
route:
  strip_path: false
```

Applichiamola con:

```bash
kubectl apply -f 05.kong-configuration-proxy.yml
```

... ma non funziona ancora.

Dobbiamo dire a Kong qual'è il servizio su cui fare questo proxy rewrite aggiungendo un annotation al nostro `Service`:

```yaml
...
  annotations:
    kubernetes.io/ingress.class: 'kong'
    konghq.com/override: proxy
...
```

In questa directory è già presente il manifest aggiornato, applichiamolo

```bash
kubectl apply -f 06.service-2.yml
```

Ora il nostro path `pet/1` funziona! 

http://yourip:31081/pet/1

Vediamo quindi con che facilità abbiamo esteso il funzionamento della Ingress con una CRD di kong.

# Kong CRD rate limiting

Ora vogliamo aggiungere un plugin di rate limiting alle nostre richieste:

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting
  namespace: petstore
  annotations:
    kubernetes.io/ingress.class: 'kong'
config:
  minute: 5
  limit_by: ip
  policy: local
plugin: rate-limiting
```

Applichiamo il manifest con:

```bash
kubectl apply -f 07.kong-plugin-rate-limit.yml
```

Ovviamente dobbiamo anche legare questa CRD alla nostra Ingress, annotiamola quindi così:

```yml
...
  annotations:
    kubernetes.io/ingress.class: 'kong'
    konghq.com/plugins: rate-limiting
...
```

E applichiamo il manifest:
 
```bash
kubectl apply -f 08.public-ingress-2.yml
```

Vediamo come ora la nostra ingress ha un altro comportamento aggiunto, il rate limiting!

http://yourip:31081/pet/1

# Metriche

Colleghiamoci ora a grafana e vediamo quali metriche espone il nostro Kong Ingress Controller

http://yourip:30001/

Vediamo che non è presente nulla nella nostra dashboard, per far si che si possano estrarre metriche applicative
dobbiamo aggiungere un altro plugin:

```yml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: prometheus
  namespace: petstore
  annotations:
    kubernetes.io/ingress.class: 'kong'
plugin: prometheus
```

```bash
kubectl apply -f 09.kong-plugin-prometheus.yml
```

E come abbiamo già visto, applichiamolo alla Ingress:

```yml
...
  annotations:
    kubernetes.io/ingress.class: 'kong'
    konghq.com/plugins: rate-limiting,prometheus
...
```

```bash
kubectl apply -f 10.public-ingress-3.yml
```

Vediamo che ora in grafana abbiamo correttamente i nostri dati popolati da prometheus.

http://yourip:31081/pet/1

La demo è ora completata!
