# <span class="rainbow">Conteneurisation avec Docker 🐳</span>

<span class="spotlight">Conteneuriser un microservice Go correctement, c'est obtenir une image petite, sécurisée et reproductible — de quelques mégaoctets, démarrant en millisecondes.</span>

---

## Le Dockerfile multi-stage

La clé pour les microservices Go : le **multi-stage build**. On compile dans une image lourde, on copie le binaire dans une image minimaliste.

```dockerfile
# ── Stage 1 : Build ──────────────────────────────────────────────────────────
FROM golang:1.23-alpine AS builder

# Dépendances système pour CGO (si utilisé)
RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /build

# Copier les fichiers de modules en premier (cache des layers)
COPY go.mod go.sum ./
RUN go mod download

# Copier les sources
COPY . .

# Compiler : binaire statique, sans symboles de debug
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build \
    -ldflags="-w -s -X main.version=$(git describe --tags --always)" \
    -o order-service \
    ./cmd/server

# ── Stage 2 : Image finale ────────────────────────────────────────────────────
FROM gcr.io/distroless/static-debian12

# Copier les certificats TLS (pour les appels HTTPS sortants)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copier le binaire
COPY --from=builder /build/order-service /order-service

# Utilisateur non-root (UID 65532 = nonroot dans distroless)
USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/order-service"]
```

### Pourquoi `distroless` ?

<div class="pulse-glow" style="padding: 1rem; border-radius: 0.5rem; margin: 1rem 0;">

| Image de base | Taille | Shell | Packages | Surface d'attaque |
|---|---|---|---|---|
| `ubuntu:24.04` | ~77MB | ✅ bash | ~400 | Élevée |
| `alpine:3.20` | ~7MB | ✅ sh | ~50 | Faible |
| `distroless/static` | ~2MB | ❌ | 0 | Minimale |
| `scratch` | 0MB | ❌ | 0 | Zéro |

</div>

`distroless` est le meilleur compromis : pas de shell (impossible d'exécuter des commandes en cas de compromission), mais inclut les certificats TLS et les timezone data.

---

## Bonnes pratiques Dockerfile

### 1. Ordre des COPY pour le cache

```dockerfile
# ✅ Correct : les dépendances changent rarement
COPY go.mod go.sum ./
RUN go mod download   # ← cette layer est cachée si go.mod n'a pas changé
COPY . .              # ← seule cette layer invalide le cache

# ❌ Mauvais : tout recompiler à chaque changement de code
COPY . .
RUN go mod download
```

### 2. Variables de build

```dockerfile
# Injecter la version au build
ARG VERSION=dev
ARG GIT_COMMIT=unknown

RUN go build \
    -ldflags="-X main.version=${VERSION} -X main.gitCommit=${GIT_COMMIT}" \
    -o myservice ./cmd/server
```

```bash
docker build \
  --build-arg VERSION=1.2.3 \
  --build-arg GIT_COMMIT=$(git rev-parse --short HEAD) \
  -t myorg/order-service:1.2.3 .
```

### 3. .dockerignore

```
# .dockerignore
.git/
.github/
*.md
*_test.go
testdata/
.env*
docker-compose*.yml
coverage.out
*.prof
```

### 4. Labels OCI

```dockerfile
LABEL org.opencontainers.image.title="order-service"
LABEL org.opencontainers.image.description="Order management microservice"
LABEL org.opencontainers.image.version="${VERSION}"
LABEL org.opencontainers.image.source="https://github.com/myorg/order-service"
```

---

## Construire et tagger l'image

```bash
# Build local
docker build -t order-service:latest .

# Avec version sémantique
VERSION=1.2.3
GIT=$(git rev-parse --short HEAD)
docker build \
  --build-arg VERSION=${VERSION} \
  --build-arg GIT_COMMIT=${GIT} \
  -t ghcr.io/myorg/order-service:${VERSION} \
  -t ghcr.io/myorg/order-service:latest \
  .

# Vérifier la taille
docker images ghcr.io/myorg/order-service
# REPOSITORY                     TAG     SIZE
# ghcr.io/myorg/order-service    1.2.3   8.3MB ✅
```

---

## Variables d'environnement et secrets

Un microservice cloud-native configure tout via l'environnement (12-Factor App) :

```bash
# Développement local
docker run \
  -e DATABASE_URL="postgres://postgres:pass@localhost:5432/orders" \
  -e KAFKA_BROKERS="localhost:9092" \
  -e PORT="8080" \
  -e LOG_LEVEL="debug" \
  -p 8080:8080 \
  order-service:latest
```

> **Jamais de secrets dans l'image** : pas de `.env` copié, pas de credentials hardcodés. En production, utiliser Kubernetes Secrets ou un vault (HashiCorp Vault, AWS Secrets Manager).

---

## Docker Compose pour le développement local

```yaml
# docker-compose.yml
services:
  order-service:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://postgres:devpass@postgres:5432/orders
      KAFKA_BROKERS: kafka:9092
      LOG_LEVEL: debug
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/healthz"]
      interval: 10s
      timeout: 5s
      retries: 3

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: orders
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./internal/infrastructure/postgres/migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk

volumes:
  postgres-data:
```

```bash
# Démarrer l'environnement de dev
docker compose up -d

# Logs en temps réel
docker compose logs -f order-service

# Reconstruire après modification
docker compose up -d --build order-service

# Arrêter et supprimer les volumes
docker compose down -v
```

---

## Scanner la sécurité de l'image

```bash
# Trivy (scanner CNCF)
trivy image ghcr.io/myorg/order-service:1.2.3

# Grype (Anchore)
grype ghcr.io/myorg/order-service:1.2.3

# Docker Scout (intégré à Docker)
docker scout cves ghcr.io/myorg/order-service:1.2.3
```

<div class="pulse-glow" style="padding: 0.5rem 1rem; background: #fef9c3; border-radius: 0.5rem; border-left: 4px solid #eab308;">

**Règle d'or** : zéro CVE critique ou haute dans les images de production. Les images distroless ont rarement des vulnérabilités car elles n'ont pas de gestionnaire de paquets ni de binaires système.

</div>

---

## Publier vers un registry

```bash
# GitHub Container Registry (ghcr.io)
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
docker push ghcr.io/myorg/order-service:1.2.3
docker push ghcr.io/myorg/order-service:latest

# Vérifier le manifest multi-arch (si applicable)
docker manifest inspect ghcr.io/myorg/order-service:1.2.3
```

### Build multi-architecture (amd64 + arm64)

```bash
# Créer un builder multi-arch
docker buildx create --use --name multiarch

# Builder pour les deux architectures
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/myorg/order-service:1.2.3 \
  --push \
  .
```

> Sur Kubernetes, les nœuds Graviton (ARM) sont 20-40% moins chers. Un build multi-arch vous donne la flexibilité de les utiliser sans changer votre image.

---

## Checklist image de production

```
✅ Multi-stage build (seul le binaire dans l'image finale)
✅ Image base distroless ou scratch
✅ CGO_ENABLED=0 (binaire statique)
✅ Utilisateur non-root
✅ .dockerignore configuré
✅ Labels OCI (version, source, revision)
✅ Pas de secrets dans l'image
✅ Scanner de vulnérabilités dans le CI
✅ Tags immutables (jamais :latest en production)
✅ Health checks définis
```

---

<span class="rainbow">🐳 Votre microservice est conteneurisé correctement. Prochaine étape : le déployer sur Kubernetes ! ➜</span>
