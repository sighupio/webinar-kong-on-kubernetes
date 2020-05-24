# Step 2 - Installiamo prometheus e grafana con KFD

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