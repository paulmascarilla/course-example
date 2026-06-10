L'architecture hexagonale, inventée par Alistair Cockburn en 2005, place la logique métier au centre de tout. L'infrastructure devient un détail d'implémentation.

---

## Le problème qu'elle résout

Dans une architecture classique en couches, la logique métier **dépend directement de l'infrastructure** :

![Architecture en couches classique](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/04_layered_architecture.svg)

**Problèmes** :
- Impossible de tester la logique métier sans démarrer une base de données
- Changer de base de données implique de modifier le cœur du service
- La logique métier "fuit" vers les contrôleurs HTTP

---

## La solution : l'hexagone

![Architecture Hexagonale](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/05_hexagonal_architecture.svg)

Le domaine ne connaît ni HTTP, ni PostgreSQL, ni Kafka. Il interagit uniquement avec des **ports** (interfaces).

---

## Ports & Adapters : les trois couches

### 1. Le Domaine (le cœur)

Contient les **entités métier** et la **logique pure** — sans aucune dépendance externe.

```go
// domain/order.go
package domain

import (
    "errors"
    "time"
)

type OrderStatus string

const (
    StatusPending   OrderStatus = "pending"
    StatusConfirmed OrderStatus = "confirmed"
    StatusShipped   OrderStatus = "shipped"
)

type Order struct {
    ID         string
    CustomerID string
    Items       []OrderItem
    Status     OrderStatus
    CreatedAt  time.Time
    Total      Money
}

type Money struct {
    Amount   int64  // centimes
    Currency string // "EUR"
}

// Règle métier : on ne peut confirmer qu'une commande en attente
func (o *Order) Confirm() error {
    if o.Status != StatusPending {
        return errors.New("only pending orders can be confirmed")
    }
    o.Status = StatusConfirmed
    return nil
}

// Règle métier : total minimum
func (o *Order) IsValid() error {
    if o.Total.Amount < 100 {
        return errors.New("order total must be at least €1.00")
    }
    return nil
}
```

### 2. Les Ports (interfaces)

Les **ports** définissent comment le domaine communique avec l'extérieur. Ils sont **des interfaces Go**, pas des implémentations.

```go
// domain/ports.go
package domain

// Port entrant (driving port) — comment on utilise le domaine
type OrderService interface {
    CreateOrder(customerID string, items []OrderItem) (*Order, error)
    ConfirmOrder(orderID string) (*Order, error)
    GetOrder(orderID string) (*Order, error)
}

// Port sortant (driven port) — ce dont le domaine a besoin
type OrderRepository interface {
    Save(order *Order) error
    FindByID(id string) (*Order, error)
    FindByCustomer(customerID string) ([]*Order, error)
}

// Port sortant — notification
type EventPublisher interface {
    Publish(event DomainEvent) error
}

type DomainEvent struct {
    Type    string
    Payload interface{}
}
```

### 3. Les Adaptateurs

Les **adaptateurs** implémentent les ports avec de vraies technologies.

```go
// infrastructure/postgres/repository.go
package postgres

import (
    "database/sql"
    "myservice/internal/domain"
)

// PostgresOrderRepository implémente domain.OrderRepository
type PostgresOrderRepository struct {
    db *sql.DB
}

func (r *PostgresOrderRepository) Save(order *domain.Order) error {
    _, err := r.db.Exec(
        `INSERT INTO orders (id, customer_id, status, total_amount, total_currency, created_at)
         VALUES ($1, $2, $3, $4, $5, $6)
         ON CONFLICT (id) DO UPDATE SET status = $3`,
        order.ID, order.CustomerID, order.Status,
        order.Total.Amount, order.Total.Currency, order.CreatedAt,
    )
    return err
}

func (r *PostgresOrderRepository) FindByID(id string) (*domain.Order, error) {
    row := r.db.QueryRow(
        `SELECT id, customer_id, status, total_amount, total_currency, created_at
         FROM orders WHERE id = $1`, id,
    )
    var o domain.Order
    return &o, row.Scan(
        &o.ID, &o.CustomerID, &o.Status,
        &o.Total.Amount, &o.Total.Currency, &o.CreatedAt,
    )
}
```

```go
// infrastructure/http/handler.go
package httphandler

import (
    "encoding/json"
    "net/http"
    "myservice/internal/domain"
)

// HTTPAdapter relie HTTP au domaine via le port entrant
type HTTPAdapter struct {
    orders domain.OrderService
}

func (h *HTTPAdapter) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid body", http.StatusBadRequest)
        return
    }
    order, err := h.orders.CreateOrder(req.CustomerID, req.Items)
    if err != nil {
        http.Error(w, err.Error(), http.StatusUnprocessableEntity)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(order)
}
```

---

## La règle de dépendance

**Règle fondamentale : les dépendances ne pointent que vers l'intérieur.**

![Règle de dépendance](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/06_dependency_rule.svg)

Concrètement en Go : le package `domain` n'importe **aucun** package externe (`database/sql`, `net/http`, etc.). Il ne dépend que de la bibliothèque standard.

---

## Injection de dépendances

Comment connecter les couches sans violer la règle ? Par **l'injection de dépendances** au point de composition (`main.go`) :

```go
// cmd/server/main.go
package main

import (
    "database/sql"
    "net/http"
    "myservice/internal/application"
    "myservice/internal/domain"
    "myservice/internal/infrastructure/postgres"
    httphandler "myservice/internal/infrastructure/http"
    "myservice/internal/infrastructure/kafka"
)

func main() {
    // Infrastructure
    db, _ := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    repo := postgres.NewOrderRepository(db)
    publisher := kafka.NewEventPublisher(os.Getenv("KAFKA_URL"))

    // Application (use cases)
    svc := application.NewOrderService(repo, publisher)

    // Adaptateur HTTP
    handler := httphandler.NewHTTPAdapter(svc)

    // Démarrage
    mux := http.NewServeMux()
    mux.HandleFunc("POST /orders", handler.CreateOrder)
    mux.HandleFunc("GET /orders/{id}", handler.GetOrder)
    http.ListenAndServe(":8080", mux)
}
```

---

## Testabilité : le vrai avantage

Avec l'architecture hexagonale, on peut tester le domaine **sans aucune infrastructure** :

```go
// domain/order_test.go
package domain_test

import (
    "testing"
    "myservice/internal/domain"
)

// Faux repository en mémoire (pas de base de données !)
type inMemoryOrderRepository struct {
    orders map[string]*domain.Order
}

func (r *inMemoryOrderRepository) Save(o *domain.Order) error {
    r.orders[o.ID] = o
    return nil
}

func (r *inMemoryOrderRepository) FindByID(id string) (*domain.Order, error) {
    o, ok := r.orders[id]
    if !ok {
        return nil, domain.ErrNotFound
    }
    return o, nil
}

func TestConfirmOrder(t *testing.T) {
    repo := &inMemoryOrderRepository{orders: make(map[string]*domain.Order)}
    svc := application.NewOrderService(repo, &fakePublisher{})

    order, _ := svc.CreateOrder("customer-1", []domain.OrderItem{
        {ProductID: "prod-1", Quantity: 2, Price: domain.Money{Amount: 1500, Currency: "EUR"}},
    })

    confirmed, err := svc.ConfirmOrder(order.ID)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if confirmed.Status != domain.StatusConfirmed {
        t.Errorf("expected confirmed, got %s", confirmed.Status)
    }
}
```

> **Résultat** : les tests sont **instantanés** (pas de démarrage de DB), **fiables** (pas de données résiduelles) et **déterministes**.

---

## Structure de répertoires recommandée

```
order-service/
├── cmd/
│   └── server/
│       └── main.go              # Composition root
├── internal/
│   ├── domain/
│   │   ├── order.go             # Entités + règles métier
│   │   ├── order_test.go        # Tests purs
│   │   └── ports.go             # Interfaces (ports)
│   ├── application/
│   │   ├── order_service.go     # Use cases
│   │   └── order_service_test.go
│   └── infrastructure/
│       ├── postgres/
│       │   ├── repository.go    # Adaptateur DB
│       │   └── migrations/
│       ├── http/
│       │   ├── handler.go       # Adaptateur HTTP
│       │   └── middleware.go
│       ├── kafka/
│       │   └── publisher.go     # Adaptateur événements
│       └── grpc/
│           └── server.go        # Adaptateur gRPC
├── api/
│   └── openapi.yaml
├── Dockerfile
└── go.mod
```

---

## Comparaison avec d'autres architectures

| Architecture | Testabilité | Flexibilité infra | Complexité |
|---|---|---|---|
| Monolithe en couches | Faible | Faible | Faible |
| Clean Architecture | Élevée | Élevée | Moyenne |
| **Hexagonale** | **Élevée** | **Élevée** | **Moyenne** |
| CQRS + Event Sourcing | Très élevée | Très élevée | Élevée |

L'architecture hexagonale est souvent présentée comme équivalente à la *Clean Architecture* de Robert Martin — les principes sont identiques, la terminologie diffère légèrement.
