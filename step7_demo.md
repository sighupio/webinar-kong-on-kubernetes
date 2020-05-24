# Step 7 - Raccogliamo le metriche con Prometheus

Colleghiamoci ora a grafana e vediamo quali metriche espone il nostro Kong Ingress Controller

http://$(minikube ip):30001/

Vediamo che non è presente nulla nella nostra dashboard, per far si che si possano estrarre metriche applicative
dobbiamo aggiungere un altro plugin, e in questo caso lo aggiungeremo globalmente in tutte le ingress gestite da Kong tramite la label `global=true`:

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: prometheus
  namespace: petstore
  labels:
    global: "true"
  annotations:
    kubernetes.io/ingress.class: 'kong'
plugin: prometheus
```

```bash
kubectl apply -f 20.kong-plugin-prometheus.yml
```

Vediamo che ora in grafana abbiamo correttamente i nostri dati popolati da prometheus.

http://$(minikube ip):31081/pet/1


