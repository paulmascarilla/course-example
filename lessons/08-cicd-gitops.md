GitOps est le principe qui fait de Git la source de vérité unique pour l'infrastructure et les applications. Tout changement passe par une PR. Le cluster se réconcilie automatiquement.

---

## Le pipeline CI/CD pour microservices

![Pipeline CI/CD](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/09_cicd_pipeline.svg)

### Principe GitOps

![GitOps Workflow](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/10_gitops_workflow.svg)

---

## GitHub Actions — Pipeline CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── Test ────────────────────────────────────────────────────────────────────
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: orders_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5

    steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-go@v5
      with:
        go-version: '1.23'
        cache: true

    - name: Run unit tests
      run: go test ./internal/domain/... ./internal/application/... -v -race

    - name: Run integration tests
      env:
        TEST_DATABASE_URL: postgres://postgres:testpass@localhost:5432/orders_test
      run: go test -tags integration ./internal/infrastructure/... -v

    - name: Check coverage
      run: |
        go test ./... -coverprofile=coverage.out
        go tool cover -func=coverage.out | grep total | awk '{print $3}' | \
          awk '{if ($1+0 < 80) {print "Coverage below 80%: " $1; exit 1}}'

  # ── Lint ────────────────────────────────────────────────────────────────────
  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: golangci/golangci-lint-action@v6
      with:
        version: latest
        args: --timeout=5m

  # ── Build & Push ─────────────────────────────────────────────────────────────
  build:
    needs: [test, lint]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
    - uses: actions/checkout@v4

    - uses: docker/setup-buildx-action@v3

    - name: Log in to registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha,prefix=sha-,format=short
          type=raw,value=latest,enable={{is_default_branch}}

    - name: Build and push
      id: build
      uses: docker/build-push-action@v6
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        build-args: |
          VERSION=${{ steps.meta.outputs.version }}
          GIT_COMMIT=${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  # ── Security Scan ────────────────────────────────────────────────────────────
  scan:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
        format: sarif
        output: trivy-results.sarif
        severity: CRITICAL,HIGH
        exit-code: '1'   # Fail si CVE critique trouvée

    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: trivy-results.sarif

  # ── Update Config Repo ───────────────────────────────────────────────────────
  deploy-staging:
    needs: [build, scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Checkout config repo
      uses: actions/checkout@v4
      with:
        repository: myorg/k8s-config
        token: ${{ secrets.CONFIG_REPO_TOKEN }}
        path: config

    - name: Update image tag in staging
      run: |
        cd config
        # Mise à jour du values.yaml avec yq
        yq e '.image.tag = "${{ needs.build.outputs.image-tag }}"' \
          -i apps/order-service/overlays/staging/values.yaml
        
        git config user.name "github-actions[bot]"
        git config user.email "github-actions[bot]@users.noreply.github.com"
        git add .
        git commit -m "chore(order-service): bump image to ${{ needs.build.outputs.image-tag }}"
        git push
```

---

## ArgoCD — Déploiement GitOps

### Installer ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Accéder à l'UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Mot de passe admin initial
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### Application ArgoCD

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-staging
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: microservices

  source:
    repoURL: https://github.com/myorg/k8s-config
    targetRevision: main
    path: apps/order-service/overlays/staging

  destination:
    server: https://kubernetes.default.svc
    namespace: staging

  syncPolicy:
    automated:
      prune: true      # Supprimer les ressources orphelines
      selfHeal: true   # Re-appliquer si quelqu'un modifie manuellement
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - ApplyOutOfSyncOnly=true   # Appliquer seulement ce qui a changé
    retry:
      limit: 5
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 5m

  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas   # Ignorer si HPA a changé les replicas
```

### AppProject — isolation multi-équipes

```yaml
# project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: microservices
  namespace: argocd
spec:
  description: "Microservices backend team"

  sourceRepos:
  - https://github.com/myorg/k8s-config
  - https://charts.helm.sh/stable

  destinations:
  - namespace: staging
    server: https://kubernetes.default.svc
  - namespace: production
    server: https://kubernetes.default.svc

  clusterResourceWhitelist:
  - group: ""
    kind: Namespace

  namespaceResourceBlacklist:
  - group: ""
    kind: ResourceQuota   # Seul l'admin peut modifier les quotas

  roles:
  - name: developer
    policies:
    - p, proj:microservices:developer, applications, get, microservices/*, allow
    - p, proj:microservices:developer, applications, sync, microservices/staging-*, allow
    groups:
    - myorg:developers
```

---

## Kustomize — Gestion des environnements

```yaml
# apps/order-service/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
- configmap.yaml
- hpa.yaml
- pdb.yaml
```

```yaml
# apps/order-service/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
bases:
- ../../base
images:
- name: ghcr.io/myorg/order-service
  newTag: sha-abc1234   # ← mis à jour par GitHub Actions
patchesStrategicMerge:
- replicas-patch.yaml
- resources-patch.yaml
```

```yaml
# apps/order-service/overlays/staging/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 1   # Staging : 1 seul réplica
```

```yaml
# apps/order-service/overlays/production/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3   # Production : 3 réplicas
```

---

## Stratégies de déploiement

### Blue/Green avec ArgoCD Rollouts

```bash
kubectl apply -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml -n argo-rollouts
```

```yaml
# rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 4
  selector:
    matchLabels:
      app: order-service
  template:
    # ... même spec que Deployment
  strategy:
    blueGreen:
      activeService: order-service-active      # Service live
      previewService: order-service-preview    # Service preview (blue)
      autoPromotionEnabled: false              # Promotion manuelle
      scaleDownDelaySeconds: 60               # Attendre avant de tuer l'ancienne version

    # OU : Canary progressif
    canary:
      steps:
      - setWeight: 10    # 10% du trafic vers la nouvelle version
      - pause: {}        # Pause manuelle
      - setWeight: 50
      - pause:
          duration: 30m  # Attendre 30min
      - setWeight: 100
      canaryService: order-service-canary
      stableService: order-service-stable
      trafficRouting:
        nginx:
          stableIngress: order-service-stable-ingress
```

```bash
# Visualiser le déploiement canary
kubectl argo rollouts get rollout order-service -n production --watch

# Promouvoir manuellement
kubectl argo rollouts promote order-service -n production

# Annuler
kubectl argo rollouts abort order-service -n production
```

---

## Notifications et alertes CD

```yaml
# argocd-notifications-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
      send: [slack-notification]
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [slack-alert]
  template.slack-notification: |
    message: |
      ✅ *{{ .app.metadata.name }}* déployé en *{{ .app.spec.destination.namespace }}*
      Version: `{{ .app.status.sync.revision }}`
  template.slack-alert: |
    message: |
      🚨 *{{ .app.metadata.name }}* dégradé en *{{ .app.spec.destination.namespace }}*
```

---

## Récapitulatif du pipeline complet

![Pipeline complet E2E](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/11_full_pipeline_e2e.svg)

---



Vous maîtrisez maintenant :
- La conception en architecture hexagonale (Ports & Adapters)
- L'implémentation d'un microservice Go production-grade
- La conteneurisation optimale avec Docker multi-stage
- Le déploiement sur Kubernetes avec toutes les ressources avancées
- L'écosystème CNCF : Helm, Prometheus, Jaeger, cert-manager
- Le CI/CD moderne avec GitHub Actions et ArgoCD (GitOps)

Allez plus loin : explorez le Service Mesh (Istio/Linkerd) et l'Event Sourcing avec CQRS !
