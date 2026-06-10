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

Le **Domain-Driven Design** introduit la notion de *Bounded Context* : chaque service a son propre vocabulaire et modèle de données. Même si deux services parlent d'un "utilisateur", leur représentation diffère — le `user-service` y voit un profil avec mot de passe, le `billing-service` une entité avec IBAN.

Dans le schéma ci-dessous, chaque service possède ses propres entités métier sans partager de modèle commun :

![Bounded Contexts](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/01_bounded_contexts.svg)

### 3. Design for Failure

Dans un système distribué, **les pannes sont inévitables**. Chaque service doit :

- Implémenter des **circuit breakers** (coupe-circuit)
- Gérer les **timeouts** et **retries avec backoff exponentiel**
- Exposer des **health checks** (`/healthz`, `/readyz`)
- Être stateless (l'état est en base, pas en mémoire)

### 4. Decentralized Data Management

L'anti-pattern le plus courant est la **base de données partagée** : tous les services lisent et écrivent dans le même schéma. Résultat — une migration dans un service peut casser les autres, et il est impossible de déployer indépendamment.

![Shared DB Antipattern](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/02_shared_db_antipattern.svg)

La solution est de donner à chaque service **sa propre base de données**. Les services ne se parlent plus via la DB — uniquement via leurs APIs. En bonus, chaque service peut choisir la technologie la mieux adaptée à son usage (relationnel, document, cache…) :

![DB per Service](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/03_db_per_service.svg)

---

## Les patterns essentiels

### API Gateway

Dans une architecture microservices, les clients ne s'adressent pas directement aux services internes. Un **API Gateway** sert de point d'entrée unique : il reçoit toutes les requêtes et les route vers le bon service selon le chemin URL. Il prend aussi en charge l'authentification, le rate limiting et l'agrégation de réponses.

![API Gateway](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/12_api_gateway.svg)

### Service Discovery

Comment les services se trouvent-ils dans un environnement dynamique où les pods démarrent et s'arrêtent en permanence ?

- **DNS-based** : Kubernetes expose chaque Service via DNS (`user-service.default.svc.cluster.local`)
- **Registry** : Consul, etcd stockent les adresses des instances actives

### Event-Driven Architecture

Dans les architectures synchrones, un appel bloquant entre `order-service` et `billing-service` crée un couplage fort : si `billing-service` est lent, tout le pipeline est ralenti. La solution est d'adopter la communication **asynchrone par événements** : un service publie un événement dans un bus de messages, et les consommateurs s'y abonnent indépendamment.

![Event-Driven Architecture](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/13_event_driven.svg)

Un seul événement `OrderCreated` déclenche ainsi la facturation, la notification et la mise à jour du stock — sans que `order-service` ne connaisse ses consommateurs. Outils courants : Kafka, NATS, RabbitMQ, Pulsar.

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

## Ce que vous allez construire dans ce cours

Au fil des modules, vous allez :

1. **Concevoir** un microservice avec l'architecture hexagonale
2. **Implémenter** la logique métier indépendamment de l'infrastructure
3. **Conteneuriser** avec Docker (Dockerfile multi-stage)
4. **Déployer** sur Kubernetes avec les ressources CNCF
5. **Observer** avec Prometheus, Grafana et Jaeger
6. **Automatiser** avec CI/CD et GitOps (ArgoCD)
