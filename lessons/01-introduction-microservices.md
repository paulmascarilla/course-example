Les microservices transforment la façon dont nous concevons, développons et déployons les applications modernes. Là où le monolithe réunit tout en un seul bloc, l'architecture microservices décompose le système en services indépendants, chacun responsable d'un périmètre métier précis.

---

## Qu'est-ce qu'un microservice ?

Un **microservice** est un service logiciel autonome qui :

- Implémente **une seule responsabilité métier** (paiement, utilisateurs, inventaire…)
- Possède **sa propre base de données** (database per service pattern)
- Communique via des **interfaces bien définies** (API REST, gRPC, événements)
- Est **déployable indépendamment** des autres services
- Peut être **développé par une équipe dédiée**

> **Règle de Conway** : « Toute organisation qui conçoit un système produira un design dont la structure est une copie de la structure de communication de cette organisation. » Les microservices alignent l'architecture sur l'organisation des équipes.

---

## Monolithe vs Microservices

| Critère | Monolithe | Microservices |
|---|---|---|
| Déploiement | Une seule unité | Indépendant par service |
| Scalabilité | Toute l'application | Service par service |
| Technologie | Stack unique | Polyglotte possible |
| Complexité opérationnelle | Faible | Élevée |
| Tolérance aux pannes | Un bug peut tout bloquer | Isolation des pannes |
| Time-to-market | Rapide au début | Rapide à grande échelle |

### Quand choisir les microservices ?

Adoptez les microservices quand :

1. **L'équipe dépasse 8-10 personnes** par domaine fonctionnel
2. **Les cycles de déploiement** sont bloqués par des dépendances inter-équipes
3. **Des modules ont des besoins de scalabilité différents** (ex : le moteur de recherche vs la gestion des commandes)
4. **La résilience est critique** — une panne partielle ne doit pas tout arrêter

> ⚠️ Ne commencez pas avec des microservices ! Partez d'un monolithe bien structuré, puis extrayez les services quand la douleur devient réelle.

---

## Les piliers d'une architecture microservices

### 1. Single Responsibility Principle (SRP)

Chaque service fait **une seule chose, et la fait bien**. Un service `user-service` gère les utilisateurs : création, authentification, profil. Il ne touche pas à la facturation.

### 2. Bounded Context (DDD)

Le **Domain-Driven Design** introduit la notion de *Bounded Context* : chaque service a son propre vocabulaire et modèle de données, même si deux services parlent d'un "utilisateur", leur représentation peut différer.

![Bounded Contexts](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/01_bounded_contexts.svg)

### 3. Design for Failure

Dans un système distribué, **les pannes sont inévitables**. Chaque service doit :

- Implémenter des **circuit breakers** (coupe-circuit)
- Gérer les **timeouts** et **retries avec backoff exponentiel**
- Exposer des **health checks** (`/healthz`, `/readyz`)
- Être stateless (l'état est en base, pas en mémoire)

### 4. Decentralized Data Management

![Shared DB Antipattern](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/02_shared_db_antipattern.svg)

![DB per Service](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/03_db_per_service.svg)

---

## Les patterns essentiels

### API Gateway

Point d'entrée unique pour tous les clients. Il gère :

- **Routage** vers les services appropriés
- **Authentification** et autorisation centralisées
- **Rate limiting** et throttling
- **Aggregation** de réponses multi-services

```bash
Client → API Gateway → user-service
                    → order-service
                    → product-service
```

### Service Discovery

Comment les services se trouvent-ils dans un environnement dynamique (Kubernetes) ?

- **DNS-based** : Kubernetes expose chaque Service via DNS (`user-service.default.svc.cluster.local`)
- **Registry** : Consul, etcd stockent les adresses des instances

### Event-Driven Architecture

Au lieu d'appels synchrones, les services publient des **événements** :

```
order-service publie → OrderCreated → billing-service consomme
                                    → notification-service consomme
                                    → inventory-service consomme
```

Outils : Kafka, NATS, RabbitMQ, Pulsar

---

## Structure d'un projet microservice typique

```
my-service/
├── cmd/
│   └── server/
│       └── main.go          # Point d'entrée
├── internal/
│   ├── domain/              # Logique métier pure
│   │   ├── entity.go
│   │   └── repository.go    # Interface (port)
│   ├── application/         # Use cases / services applicatifs
│   │   └── service.go
│   └── infrastructure/      # Adaptateurs (DB, HTTP, gRPC)
│       ├── postgres/
│       └── http/
├── api/
│   └── openapi.yaml         # Contrat d'API
├── Dockerfile
├── helm/                    # Chart Kubernetes
└── README.md
```

---

## Les 12 facteurs (12-Factor App)

Une application microservice cloud-native suit ces principes :

1. **Codebase** — Un repo, plusieurs déploiements
2. **Dependencies** — Dépendances déclarées explicitement
3. **Config** — Configuration dans l'environnement
4. **Backing services** — DB, cache = ressources attachées
5. **Build, release, run** — Étapes strictement séparées
6. **Processes** — Exécution en processus stateless
7. **Port binding** — Export des services via ports
8. **Concurrency** — Scalabilité par multiplication de processus
9. **Disposability** — Démarrage rapide, arrêt gracieux
10. **Dev/prod parity** — Environnements similaires
11. **Logs** — Traités comme des flux d'événements
12. **Admin processes** — Tâches admin comme des one-offs

---

## Ce que vous allez construire dans ce cours

Au fil des modules, vous allez :

1. **Concevoir** un microservice avec l'architecture hexagonale
2. **Implémenter** la logique métier indépendamment de l'infrastructure
3. **Conteneuriser** avec Docker (Dockerfile multi-stage)
4. **Déployer** sur Kubernetes avec les ressources CNCF
5. **Observer** avec Prometheus, Grafana et Jaeger
6. **Automatiser** avec CI/CD et GitOps (ArgoCD)
