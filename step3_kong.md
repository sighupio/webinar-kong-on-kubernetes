# Step 3 - Installiamo Kong Ingress Controller con KFD

Scommentiamo la riga `- ./vendor/katalog/kong/kong` al nostro file `kustomization.yaml` 

Se non volete faticare: 

```bash
sed -i 's/#//' kustomization.yaml
```

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