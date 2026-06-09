<span class="spotlight">Les ressources avancées de Kubernetes transforment un simple orchestrateur de conteneurs en plateforme production-grade : routage intelligent, scalabilité automatique, et stockage persistant.</span>

---

## Ingress — Routage HTTP/HTTPS

Un **Ingress** expose plusieurs Services via un seul point d'entrée HTTP/S avec routage basé sur le path et le hostname.

```
Internet → [Ingress Controller] → /api/orders → order-service
                                → /api/users  → user-service
                                → /api/products → product-service
```

### Ingress avec NGINX Ingress Controller

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    # NGINX Ingress Controller
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    # TLS via cert-manager
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.myapp.com
    secretName: api-tls-cert
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /api/orders(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /api/users(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
```

### Installer NGINX Ingress Controller (via Helm)

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.metrics.enabled=true \
  --set controller.metrics.serviceMonitor.enabled=true
```

---

## HorizontalPodAutoscaler (HPA)

L'HPA ajuste automatiquement le nombre de réplicas selon la charge.

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
  # CPU : scale si > 70% des requests
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Mémoire : scale si > 80% des requests
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Métrique custom : requêtes/sec (via Prometheus Adapter)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # Attendre 1min avant de scaler up
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Attendre 5min avant de scaler down
      policies:
      - type: Percent
        value: 25
        periodSeconds: 60
```

```bash
# Surveiller l'HPA
kubectl get hpa -n production --watch
kubectl describe hpa order-service-hpa -n production
```

---

## VerticalPodAutoscaler (VPA)

Le VPA ajuste automatiquement les `requests` et `limits` selon l'historique d'utilisation.

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: "Off"    # "Off" = recommandations seulement, pas d'application auto
  resourcePolicy:
    containerPolicies:
    - containerName: order-service
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 2
        memory: 1Gi
```

```bash
# Voir les recommandations
kubectl describe vpa order-service-vpa -n production
```

---

## Volumes et Stockage Persistant

### PersistentVolume & PersistentVolumeClaim

```yaml
# pvc.yaml — Demande de stockage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce     # Un seul nœud peut monter en écriture
  storageClassName: fast-ssd   # Défini par le cloud provider
  resources:
    requests:
      storage: 20Gi
```

```yaml
# Utilisation dans un StatefulSet (pour PostgreSQL)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: password
        - name: POSTGRES_DB
          value: orders
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi
```

---

## RBAC — Contrôle d'accès

```yaml
# ServiceAccount pour le microservice
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: production
---
# Role : accès lecture seule aux ConfigMaps
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: order-service-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["order-service-secrets"]  # Limité à UN secret
---
# RoleBinding : associer le role au ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-service-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: order-service
  namespace: production
roleRef:
  kind: Role
  apiVersion: rbac.authorization.k8s.io/v1
  name: order-service-role
```

```yaml
# Référencer le ServiceAccount dans le Deployment
spec:
  template:
    spec:
      serviceAccountName: order-service
```

---

## NetworkPolicy — Segmentation réseau

Par défaut, tous les Pods peuvent se parler. En production, appliquez le **principe du moindre privilège** :

```yaml
# networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Autoriser seulement depuis l'ingress-nginx et l'api-gateway
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Autoriser vers PostgreSQL
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  # Autoriser vers Kafka
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kafka
    ports:
    - protocol: TCP
      port: 9092
  # DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

---

## Jobs et CronJobs

```yaml
# cronjob.yaml — Nettoyage des commandes annulées
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-cancelled-orders
  namespace: production
spec:
  schedule: "0 2 * * *"   # Toutes les nuits à 2h
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: ghcr.io/myorg/order-service:1.2.3
            command: ["/order-service", "cleanup", "--older-than=30d"]
            envFrom:
            - secretRef:
                name: order-service-secrets
```

---

## Manifests complets pour la production

<div class="pulse-glow" style="padding: 1rem; border-radius: 0.5rem; margin: 1rem 0; background: #f5f3ff; border-left: 4px solid #8b5cf6;">

**Organisation recommandée des manifests :**

```
k8s/
├── base/
│   ├── namespace.yaml
│   ├── serviceaccount.yaml
│   ├── rbac.yaml
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── pdb.yaml
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replicas-patch.yaml
    └── production/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```

Utilisez **Kustomize** (intégré à kubectl) ou **Helm** pour gérer les différences entre environnements.

</div>

```bash
# Appliquer avec Kustomize
kubectl apply -k k8s/overlays/production/

# Vérifier l'état de tous les objets
kubectl get all,ingress,hpa,pdb -n production
```

---

<span class="rainbow">⚙️ Votre infrastructure Kubernetes est production-ready. Place à l'écosystème CNCF ! ➜</span>
