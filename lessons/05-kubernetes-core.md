Kubernetes orchestre vos conteneurs à grande échelle : placement, scalabilité, auto-réparation et réseau. C'est le système d'exploitation du cloud.

---

## Architecture de Kubernetes

![Architecture Kubernetes](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/07_kubernetes_architecture.svg)

**Composants clés :**
- **API Server** : point d'entrée, valide et persiste les objets dans etcd
- **etcd** : base de données distribuée, source de vérité du cluster
- **Scheduler** : décide sur quel nœud placer chaque Pod
- **Controller Manager** : réconcilie l'état souhaité avec l'état réel
- **kubelet** : agent sur chaque nœud, démarre et surveille les Pods
- **kube-proxy** : gère les règles réseau pour les Services

---

## Les objets fondamentaux

### Pod — l'unité de base

Un Pod encapsule un ou plusieurs conteneurs partageant le réseau et les volumes.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-service
  labels:
    app: order-service
    version: v1.2.3
spec:
  containers:
  - name: order-service
    image: ghcr.io/myorg/order-service:1.2.3
    ports:
    - containerPort: 8080
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: order-service-secrets
          key: database-url
    - name: PORT
      value: "8080"
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "200m"
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /readyz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 5
```

> **Attention** : ne déployez jamais un Pod directement en production. Utilisez un Deployment qui gère la résilience.

### Deployment — la gestion des réplicas

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # Max 1 Pod indisponible pendant le rollout
      maxSurge: 1          # Max 1 Pod en plus pendant le rollout
  template:
    metadata:
      labels:
        app: order-service
        version: "1.2.3"
    spec:
      # Distribution anti-affinité : un Pod par nœud
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: order-service
              topologyKey: kubernetes.io/hostname
      containers:
      - name: order-service
        image: ghcr.io/myorg/order-service:1.2.3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http
        envFrom:
        - configMapRef:
            name: order-service-config
        - secretRef:
            name: order-service-secrets
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"] # Drain des connexions
      terminationGracePeriodSeconds: 30
```

```bash
# Commandes essentielles
kubectl apply -f deployment.yaml
kubectl rollout status deployment/order-service
kubectl rollout history deployment/order-service

# Rollback si problème
kubectl rollout undo deployment/order-service
kubectl rollout undo deployment/order-service --to-revision=2
```

---

### Service — l'exposition réseau

Les Services donnent une IP stable et un DNS aux Pods (qui ont des IPs éphémères).

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service
  ports:
  - name: http
    protocol: TCP
    port: 80          # Port du Service (interne cluster)
    targetPort: 8080  # Port du conteneur
  type: ClusterIP     # Accessible seulement dans le cluster
```

**Types de Service :**

| Type | Usage | Accès |
|---|---|---|
| `ClusterIP` | Communication inter-services | Cluster uniquement |
| `NodePort` | Développement/tests | Port sur chaque nœud |
| `LoadBalancer` | Production (cloud) | IP externe via cloud LB |
| `ExternalName` | Alias DNS vers service externe | — |

**DNS interne** : `order-service.production.svc.cluster.local`

---

### ConfigMap — configuration non-sensible

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: production
data:
  PORT: "8080"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  KAFKA_BROKERS: "kafka.kafka.svc.cluster.local:9092"
  KAFKA_TOPIC_ORDERS: "orders.events"
  MAX_CONNECTIONS: "20"
```

```yaml
# Utilisation dans le Deployment
envFrom:
- configMapRef:
    name: order-service-config

# Ou variable par variable
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: order-service-config
      key: LOG_LEVEL
```

---

### Secret — données sensibles

```bash
# Créer un secret depuis la CLI
kubectl create secret generic order-service-secrets \
  --from-literal=database-url="postgres://user:pass@postgres:5432/orders" \
  --from-literal=jwt-secret="$(openssl rand -hex 32)" \
  --namespace=production
```

```yaml
# secret.yaml (valeurs en base64)
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: production
type: Opaque
data:
  database-url: cG9zdGdyZXM6Ly91c2VyOnBhc3NAcG9zdGdyZXM6NTQzMi9vcmRlcnM=
  jwt-secret: YWJjMTIzZGVmNDU2...
```

> ⚠️ Les Secrets Kubernetes sont encodés en base64, **pas chiffrés**. En production, utilisez [External Secrets Operator](https://external-secrets.io) avec HashiCorp Vault ou AWS Secrets Manager pour chiffrement réel.

---

### Namespace — isolation logique

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
```

```bash
# Conventions de nommage
kubectl get pods -n production
kubectl get all -n staging
kubectl config set-context --current --namespace=production
```

---

## Commandes essentielles kubectl

```bash
# Ressources
kubectl get pods,svc,deploy -n production
kubectl describe pod order-service-6d4b9c-xk2lm -n production
kubectl logs order-service-6d4b9c-xk2lm -n production --follow
kubectl exec -it order-service-6d4b9c-xk2lm -n production -- sh

# Débogage
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl top pods -n production
kubectl top nodes

# Dry-run (simuler sans appliquer)
kubectl apply -f deployment.yaml --dry-run=server

# Diff avant apply
kubectl diff -f deployment.yaml
```

---

## Requests & Limits — la ressource clé

**Ne jamais déployer sans `resources`.**

- **requests** : ce que le scheduler garantit au Pod (utilisé pour le placement)
- **limits** : le maximum autorisé (CPU throttling, OOM kill si mémoire dépassée)

```yaml
resources:
  requests:
    cpu: "100m"      # 0.1 vCPU
    memory: "128Mi"
  limits:
    cpu: "500m"      # 0.5 vCPU
    memory: "256Mi"
```

**Bonnes valeurs de départ pour un service Go** :
- CPU request : `50m-100m`, limit : `200m-500m`
- Memory request : `64Mi-128Mi`, limit : `128Mi-256Mi`

---

## PodDisruptionBudget — haute disponibilité

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  minAvailable: 2   # Au moins 2 Pods disponibles pendant la maintenance
  selector:
    matchLabels:
      app: order-service
```

Garantit que lors d'un drain de nœud ou d'une mise à jour de cluster, au moins 2 réplicas restent disponibles.

---

