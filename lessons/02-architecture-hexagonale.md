L'architecture hexagonale, inventée par Alistair Cockburn en 2005, place la logique métier au centre de tout. L'infrastructure devient un détail d'implémentation.

---

## Le problème qu'elle résout

Dans une architecture classique en couches, chaque couche dépend directement de celle du dessous — la logique métier finit par importer `database/sql`, les contrôleurs HTTP manipulent des entités de base de données, et tester quoi que ce soit nécessite de démarrer une vraie DB.

Le schéma suivant illustre ce couplage : les dépendances descendent vers l'infrastructure sans barrière d'abstraction :

![Architecture en couches classique](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/04_layered_architecture.svg)

**Problèmes** :
- Impossible de tester la logique métier sans démarrer une base de données
- Changer de base de données implique de modifier le cœur du service
- La logique métier "fuit" vers les contrôleurs HTTP

---

## La solution : l'hexagone

L'architecture hexagonale inverse ce rapport : c'est l'infrastructure qui dépend du domaine, jamais l'inverse. Le domaine ne connaît ni HTTP, ni PostgreSQL, ni Kafka — il interagit uniquement avec des **ports** (interfaces Go).

![Architecture Hexagonale](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/05_hexagonal_architecture.svg)

Les adaptateurs en haut (HTTP, CLI, gRPC) pilotent le domaine via des ports entrants. Les adaptateurs en bas (PostgreSQL, Redis, Kafka) sont pilotés par le domaine via des ports sortants. Le domaine lui-même ignore tout de ces technologies.

---

## Ports & Adapters : les trois couches

### 1. Le Domaine (le cœur)

Contient les **entités métier** et la **logique pure** — sans aucune dépendance externe.

```go
// domain/order.go
type Order struct {
    ID string; CustomerID string; Status OrderStatus; Total Money
}

func (o *Order) Confirm() error {
    if o.Status != StatusPending {
        return errors.New("only pending orders can be confirmed")
    }
    o.Status = StatusConfirmed; return nil
}
```

### 2. Les Ports (interfaces)

Les **ports** définissent comment le domaine communique avec l'extérieur.

```go
// domain/ports.go
type OrderService interface {
    CreateOrder(customerID string, items []OrderItem) (*Order, error)
    ConfirmOrder(orderID string) (*Order, error)
}

type OrderRepository interface {
    Save(order *Order) error
    FindByID(id string) (*Order, error)
}
```

### 3. Les Adaptateurs

Les **adaptateurs** implémentent les ports avec de vraies technologies.

```go
// infrastructure/postgres/repository.go
type PostgresOrderRepository struct{ db *sql.DB }

func (r *PostgresOrderRepository) Save(order *domain.Order) error {
    _, err := r.db.Exec(
        `INSERT INTO orders (id, customer_id, status, total_amount)
         VALUES ($1, $2, $3, $4) ON CONFLICT (id) DO UPDATE SET status=$3`,
        order.ID, order.CustomerID, order.Status, order.Total.Amount,
    )
    return err
}
```

---

## La règle de dépendance

La règle fondamentale est simple : **les dépendances ne pointent que vers l'intérieur**. L'infrastructure connaît l'application, l'application connaît le domaine — mais le domaine ne connaît personne.

![Règle de dépendance](https://raw.githubusercontent.com/paulmascarilla/course-example/main/diagrams/06_dependency_rule.svg)

Concrètement en Go : le package `domain` n'importe **aucun** package externe (`database/sql`, `net/http`, etc.). Si vous voyez un import d'infrastructure dans `domain/`, la règle est violée.

---

## Injection de dépendances

Comment connecter les couches sans violer la règle ? Par **l'injection de dépendances** dans `main.go` — le seul endroit où tout le système est assemblé :

```go
// cmd/server/main.go
func main() {
    db, _  := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    repo   := postgres.NewOrderRepository(db)
    svc    := application.NewOrderService(repo)
    handler := httphandler.NewHTTPAdapter(svc)

    mux := http.NewServeMux()
    mux.HandleFunc("POST /orders",      handler.CreateOrder)
    mux.HandleFunc("GET  /orders/{id}", handler.GetOrder)
    http.ListenAndServe(":8080", mux)
}
```

---

## Testabilité : le vrai avantage

Avec l'architecture hexagonale, on peut tester le domaine **sans aucune infrastructure** en substituant les adaptateurs réels par des implémentations en mémoire :

```go
// domain/order_test.go
type inMemoryRepo struct{ orders map[string]*domain.Order }

func (r *inMemoryRepo) Save(o *domain.Order) error {
    r.orders[o.ID] = o; return nil
}

func TestConfirmOrder(t *testing.T) {
    svc := application.NewOrderService(&inMemoryRepo{orders: map[string]*domain.Order{}})
    order, _ := svc.CreateOrder("customer-1", someItems)
    confirmed, err := svc.ConfirmOrder(order.ID)
    assert.NoError(t, err)
    assert.Equal(t, domain.StatusConfirmed, confirmed.Status)
}
```

> **Résultat** : tests **instantanés** (pas de DB), **fiables**, **déterministes**.

---

## Structure de répertoires

```
order-service/
├── cmd/server/main.go          # Composition root
├── internal/
│   ├── domain/
│   │   ├── order.go            # Entités + règles métier
│   │   ├── order_test.go       # Tests purs
│   │   └── ports.go            # Interfaces (ports)
│   ├── application/
│   │   └── order_service.go    # Use cases
│   └── infrastructure/
│       ├── postgres/repository.go
│       └── http/handler.go
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
