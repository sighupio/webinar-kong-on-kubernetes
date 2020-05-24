# Step 5 - Esponiamo Petstore tramite Ingress

Creiamo la `Ingress`

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

Applichiamola con:

```bash
kubectl apply -f 04.public-ingress-1.yml
```

Stiamo utilizzando l'ingress class `kong` e stiamo mappando il path `/pet` al nostro servizio petstore.

Proviamo ad accedere al path `/pet/1` tramite la nostra ingress...

http://$(minikube ip):31081/pet/1

Riceviamo un errore 404, dato che il path a cui stiamo accedendo è `/pet/1` mentre nel pod sarà `/api/pet/1`.

Procediamo al prossimo step dove configureremo le estensioni di Kong tramite CRD (Custom Resource Definition)

* [Step 6 - Estendiamo il funzionamento con le CRD di Kong](step6_demo.md)
