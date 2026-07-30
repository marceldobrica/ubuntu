# Chapter 20 — Monitoring

## Overview

Install Prometheus, Grafana, and Loki (optional) in the `monitoring` namespace for cluster and application observability. Expose Grafana at `grafana.<YOUR_DOMAIN>`.

## Prerequisites

- [Chapter 11 — Kubernetes (K3s)](./11-kubernetes-k3s.md)
- [Chapter 14 — Traefik](./14-traefik.md)
- [Chapter 15 — Cert Manager](./15-cert-manager.md)

## Goals

- [ ] Prometheus scraping metrics
- [ ] Grafana dashboards
- [ ] Loki collecting logs (optional)
- [ ] Alerts basics documented

## Steps

### 1. kube-prometheus-stack via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=<GRAFANA_ADMIN_PASSWORD> \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=20Gi
```

Add to GitOps for persistence.

### 2. Grafana ingress

IngressRoute host `grafana.<YOUR_DOMAIN>`, TLS wildcard secret.

Or port-forward for lab-only access:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-grafana 3000:80
```

### 3. Import dashboards

Grafana UI → Dashboards → Import:

- **315** — Kubernetes cluster monitoring
- **9628** — Traefik 2
- **7371** — MariaDB/MySQL (if exporter added)

### 4. Loki stack (optional)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true
```

Add Loki datasource in Grafana: `http://loki:3100`

### 5. Application metrics

Expose Symfony/Drupal metrics via exporters as needed. Start with kube-state-metrics and node-exporter (included in stack).

### 6. Basic alerts (preview)

Configure `PrometheusRule` for:

- Pod crash looping
- Node disk > 85%
- Certificate expiry < 14 days (cert-manager metrics)

## Verify

```bash
kubectl get pods -n monitoring
kubectl port-forward -n monitoring svc/kube-prometheus-kube-prometheus-prometheus 9090:9090
# Open Prometheus targets — UP status
```

Grafana login: admin / `<GRAFANA_ADMIN_PASSWORD>`

## Troubleshooting

| Problem | Fix |
|---------|-----|
| OOM on MiniPC | Reduce retention; disable some exporters |
| No data in Grafana | Check datasource URL and pod labels |
| Loki disk growth | Retention settings |

## Next

→ [Chapter 21 — Backup](./21-backup.md)
