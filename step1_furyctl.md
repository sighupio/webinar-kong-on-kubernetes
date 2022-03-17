# Step 1 - Setup usando Furyctl

Iniziamo scaricando le dipendenze tramite furyctl:

Posizioniamoci in 

```bash
cd webinar-kong-on-kubernetes/
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
furyctl distribution download --https
```

Procediamo quindi al prossimo step, installazione di prometheus e grafana

[Step 2 - Installiamo Prometheus e Grafana con KFD](step2_kustomize.md)