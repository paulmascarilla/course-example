# <span class="rainbow">L'Écosystème CNCF 🌐</span>

<span class="spotlight">La Cloud Native Computing Foundation (CNCF) héberge les outils qui composent le stack cloud-native moderne : de Kubernetes jusqu'à l'observabilité, la sécurité et le déploiement.</span>

---

## Qu'est-ce que la CNCF ?

La CNCF est une fondation sous la Linux Foundation qui :
- **Héberge** des projets open source cloud-native (Kubernetes, Prometheus, Envoy, containerd…)
- **Définit** des standards et certifications (CKA, CKAD, CKS)
- **Publie** le [CNCF Landscape](https://landscape.cncf.io) — plus de 1000 projets

Les projets ont 3 maturités : **Sandbox** → **Incubating** → **Graduated** ✅

---

## Helm — Le gestionnaire de paquets Kubernetes

Helm empaquette des manifests Kubernetes en **charts** réutilisables et versionnés.

```bash
# Installer Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Ajouter des dépôts
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Créer un chart pour order-service

```bash
helm create order-service
```

Structure générée :

```
order-service/
├── Chart.yaml        # Métadonnées du chart
├── values.yaml       # Valeurs par défaut
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl  # Helpers Jinja/Go template
└── charts/           # Dépendances
```

```yaml
# Chart.yaml
apiVersion: v2
name: order-service
description: Order management microservice
type: application
version: 0.1.0
appVersion: "1.2.3"
dependencies:
- name: postgresql
  version: "15.x.x"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
```

```yaml
# values.yaml
replicaCount: 2

image:
  repository: ghcr.io/myorg/order-service
  tag: ""           # Utilise appVersion si vide
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: api.myapp.com
  path: /api/orders

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled: true
  auth:
    database: orders
    existingSecret: postgres-secrets

config:
  logLevel: info
  kafkaBrokers: kafka.kafka.svc.cluster.local:9092
```

```bash
# Déployer
helm upgrade --install order-service ./order-service \
  --namespace production --create-namespace \
  --values values-production.yaml

# Déployer une version spécifique
helm upgrade order-service ./order-service \
  --set image.tag=1.3.0 \
  --namespace production

# Historique et rollback
helm history order-service -n production
helm rollback order-service 2 -n production

# Supprimer
helm uninstall order-service -n production
```

---

## Prometheus & Grafana — Observabilité des métriques

### Installer le stack complet

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

### Exposer des métriques depuis Go

```go
// internal/infrastructure/http/metrics.go
package httphandler

import (
    "net/http"
    "strconv"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Namespace: "order_service",
            Name:      "http_requests_total",
            Help:      "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Namespace: "order_service",
            Name:      "http_request_duration_seconds",
            Help:      "HTTP request duration",
            Buckets:   prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )
    ordersCreatedTotal = promauto.NewCounter(prometheus.CounterOpts{
        Namespace: "order_service",
        Name:      "orders_created_total",
        Help:      "Total orders created",
    })
)

// Middleware d'instrumentation
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &responseWriter{ResponseWriter: w, status: 200}
        next.ServeHTTP(rw, r)
        duration := time.Since(start).Seconds()

        httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, strconv.Itoa(rw.status)).Inc()
        httpRequestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
    })
}

// Exposer /metrics
mux.Handle("GET /metrics", promhttp.Handler())
```

### ServiceMonitor — scraping automatique

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-service
  namespace: production
  labels:
    release: kube-prometheus-stack   # Label requis par Prometheus Operator
spec:
  selector:
    matchLabels:
      app: order-service
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
```

### PrometheusRule — alertes

```yaml
# prometheusrule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: order-service-alerts
  namespace: production
spec:
  groups:
  - name: order-service
    interval: 30s
    rules:
    - alert: OrderServiceHighErrorRate
      expr: |
        rate(order_service_http_requests_total{status=~"5.."}[5m])
        / rate(order_service_http_requests_total[5m]) > 0.05
      for: 2m
      labels:
        severity: critical
        team: backend
      annotations:
        summary: "High error rate on order-service"
        description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"

    - alert: OrderServiceHighLatency
      expr: |
        histogram_quantile(0.99,
          rate(order_service_http_request_duration_seconds_bucket[5m])
        ) > 1.0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "p99 latency above 1s on order-service"
```

---

## Jaeger — Distributed Tracing

Le tracing distribué permet de suivre une requête à travers tous les microservices.

```bash
# Installer Jaeger Operator
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/latest/download/jaeger-operator.yaml -n observability
```

```yaml
# jaeger.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: production
  collector:
    replicas: 2
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
```

### Instrumenter le service Go avec OpenTelemetry

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/trace"
)

func initTracer(ctx context.Context) (*trace.TracerProvider, error) {
    exporter, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint(os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")),
        otlptracehttp.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("order-service"),
            semconv.ServiceVersion("1.2.3"),
        )),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}

// Dans un handler
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    ctx, span := otel.Tracer("order-service").Start(r.Context(), "CreateOrder")
    defer span.End()

    // Le context propagé aux appels suivants
    order, err := h.orders.CreateOrder(ctx, cmd)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        // ...
    }
}
```

---

## cert-manager — TLS automatique

```bash
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

```yaml
# cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@myapp.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

cert-manager renouvelle automatiquement les certificats TLS 30 jours avant expiration.

---

## External Secrets Operator — Secrets sécurisés

```bash
helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace
```

```yaml
# secretstore.yaml — connexion à HashiCorp Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "order-service"
---
# externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: order-service-secrets
    creationPolicy: Owner
  data:
  - secretKey: database-url
    remoteRef:
      key: order-service/production
      property: database-url
```

---

## Vue d'ensemble du stack CNCF

<div class="pulse-glow" style="padding: 1rem; border-radius: 0.5rem; margin: 1rem 0;">

| Catégorie | Outil | Maturité |
|---|---|---|
| Orchestration | Kubernetes | ✅ Graduated |
| Packaging | Helm | ✅ Graduated |
| Container Runtime | containerd | ✅ Graduated |
| Métriques | Prometheus | ✅ Graduated |
| Tracing | Jaeger, OpenTelemetry | ✅ Graduated |
| Réseau | Cilium, Calico | ✅/⚡ |
| Service Mesh | Istio, Linkerd | ✅ Graduated |
| GitOps | ArgoCD, Flux | ✅ Graduated |
| Secrets | External Secrets Operator | ⚡ Incubating |
| TLS | cert-manager | ⚡ Incubating |

</div>

---

<span class="rainbow">🌐 Vous connaissez maintenant le stack CNCF. Dernier module : CI/CD et GitOps avec ArgoCD ! ➜</span>
