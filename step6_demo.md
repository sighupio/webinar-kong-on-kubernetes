# Step 6 - Estendiamo il funzionamento con le CRD di Kong

Procediamo quindi a estendere il funzionamento tramite le Kong CRD.

### Kong CRD proxy

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
```

Applichiamola con:

```bash
kubectl apply -f 05.kong-configuration-proxy.yml
```

... ma non funziona ancora.

Dobbiamo instrumentare Kong sul corretto proxy rewrite collegando la configurazione appena creata al servizio k8s
tramite un annotation:

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

http://$(minikube ip):31081/pet/1

Vediamo quindi con che facilità abbiamo esteso il funzionamento della Ingress con una CRD di kong.

### Kong CRD rate limiting

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

```yaml
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

http://$(minikube ip):31081/pet/1

### Kong CRD REST method filtering

Tramite la risorsa `KongIngress` è possibile anche estendere ulteriormente il funzionamento della nostra Ingress.

Per esempio, possiamo agire sui metodi disponibili rendendo disponibile solamente il metodo `GET`.

Nel prossimo esempio, proviamo prima a rendere disponibile solamente il metodo `POST`, e verifichiamo che la richiesta venga negata.

Successivamente applichiamo il metodo corretto `GET`.

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: methods-customization
  namespace: petstore
  annotations:
    kubernetes.io/ingress.class: 'kong'
route:
  methods:
    - POST
```

Applichiamo il manifest con:

```bash
kubectl apply -f 09.kong-configuration-methods.yml
```

Colleghiamolo alla nostra ingress:

```yaml
...
  annotations:
    kubernetes.io/ingress.class: 'kong'
    konghq.com/plugins: rate-limiting
    konghq.com/override: methods-customization
...
```

Applichiamo la nostra ingress aggiornata:

```bash
kubectl apply -f 10.public-ingress-3.yml
```

Accediamo al nostro url

http://$(minikube ip):31081/pet/1

Vediamo che non è stata trovata una rotta corrispondente, dato che ci siamo connessi tramite un metodo `GET`.

Applichiamo il manifest aggiornato con il method `GET`:

```bash
kubectl apply -f 11.kong-configuration-methods.yml
```

Ora la nostra ingress è ritornata funzionante!

http://$(minikube ip):31081/pet/1

### Kong CRD basic auth

Come abbiamo visto possiamo aggiungere molte funzionalità, vediamo ora come aggiungere un metodo di autenticaizone alla 
Ingress.

Questo esempio userà una semplice autenticazione basic auth, ma vedremo come sia possibile aggiungere più utenti in maniera dinamica.

Creiamo il nostro plugin per la basic auth


```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: basic-auth
  namespace: petstore
  annotations:
    kubernetes.io/ingress.class: 'kong'
config:
  hide_credentials: true
plugin: basic-auth
```

Applichiamo il manifest con:

```bash
kubectl apply -f 12.kong-plugin-basic-auth.yml
```

Associamo la basic auth alla nostra ingress:

```yaml
...
  annotations:
    kubernetes.io/ingress.class: 'kong'
    konghq.com/plugins: rate-limiting,basic-auth
    konghq.com/override: methods-customization
...
```

Applichiamo il manifest con:

```bash
kubectl apply -f 13.public-ingress-4.yml
```

Proviamo ora a collegaci, riceviamo una richiesta di username e password, che non abbiamo ancora definito.

http://$(minikube ip):31081/pet/1

Creiamo i nostri utenti. Per fare ciò, dobbiamo creare un secret e una risorsa KongConsumer ed un Secret.

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: user1
  namespace: petstore
  annotations:
    kubernetes.io/ingress.class: 'kong'
username: user
credentials:
  - user1
---
apiVersion: v1
kind: Secret
metadata:
  name: user1
  namespace: petstore
stringData:
  kongCredType: basic-auth
  password: "123456"
  username: "user1"
```

Applichiamo il manifest con:

```bash
kubectl apply -f 14.kong-consumer-user1.yml
```

Proviamo ora a collegarci con la nostra utenza

http://$(minikube ip):31081/pet/1

Vediamo che le credenziali sono accettate!

Proviamo ora ad aggiungere un altro utente:

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: user2
  namespace: petstore
  annotations:
    kubernetes.io/ingress.class: 'kong'
username: user
credentials:
  - user2
---
apiVersion: v1
kind: Secret
metadata:
  name: user2
  namespace: petstore
stringData:
  kongCredType: basic-auth
  password: "654321"
  username: "user2"
```

Applichiamo il manifest con:

```bash
kubectl apply -f 15.kong-consumer-user2.yml
```

E anche queste credenziali sono accettate! 

Ovviamente la basic auth è la modalità più semplice per effettuare un autenticazione HTTP, possiamo anche configurare altri plugin
che utilizzano altre metodologie.