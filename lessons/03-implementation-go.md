# <span class="rainbow">Implémenter un Microservice en Go 🦫</span>

<span class="spotlight">Go est le langage de prédilection du cloud-native. Binaires statiques, faible empreinte mémoire, goroutines légères — tout pour les microservices.</span>

---

## Pourquoi Go pour les microservices ?

- **Compilation rapide** : `go build` en secondes
- **Binaire statique** : image Docker `FROM scratch` possible
- **Faible empreinte** : ~10MB de RAM au démarrage vs ~300MB pour la JVM
- **Goroutines** : concurrence légère, idéale pour les I/O
- **Bibliothèque standard riche** : `net/http`, `database/sql`, `encoding/json`

---

## Mise en place du projet

```bash
# Créer le module
mkdir order-service && cd order-service
go mod init github.com/myorg/order-service

# Dépendances essentielles
go get github.com/jackc/pgx/v5              # Driver PostgreSQL
go get github.com/google/uuid               # UUIDs
go get go.uber.org/zap                      # Logging structuré
go get github.com/prometheus/client_golang  # Métriques
```

### Structure initiale

```bash
mkdir -p cmd/server \
         internal/domain \
         internal/application \
         internal/infrastructure/{postgres,http,kafka} \
         api
```

---

## Couche Domaine

### Entités et règles métier

```go
// internal/domain/order.go
package domain

import (
    "errors"
    "time"
)

var (
    ErrNotFound        = errors.New("order not found")
    ErrInvalidStatus   = errors.New("invalid order status transition")
    ErrMinimumAmount   = errors.New("order total below minimum")
)

type OrderStatus string

const (
    StatusDraft     OrderStatus = "draft"
    StatusPending   OrderStatus = "pending"
    StatusConfirmed OrderStatus = "confirmed"
    StatusShipped   OrderStatus = "shipped"
    StatusCancelled OrderStatus = "cancelled"
)

// validTransitions définit les transitions d'état autorisées
var validTransitions = map[OrderStatus][]OrderStatus{
    StatusDraft:     {StatusPending},
    StatusPending:   {StatusConfirmed, StatusCancelled},
    StatusConfirmed: {StatusShipped, StatusCancelled},
}

type Order struct {
    ID         string
    CustomerID string
    Items      []OrderItem
    Status     OrderStatus
    CreatedAt  time.Time
    UpdatedAt  time.Time
    Total      Money
}

type OrderItem struct {
    ProductID string
    Name      string
    Quantity  int
    UnitPrice Money
}

type Money struct {
    Amount   int64  // en centimes
    Currency string
}

func (m Money) String() string {
    return fmt.Sprintf("%.2f %s", float64(m.Amount)/100, m.Currency)
}

// NewOrder crée une commande valide
func NewOrder(id, customerID string, items []OrderItem) (*Order, error) {
    if len(items) == 0 {
        return nil, errors.New("order must have at least one item")
    }
    total := computeTotal(items)
    if total.Amount < 100 { // minimum €1
        return nil, ErrMinimumAmount
    }
    return &Order{
        ID:         id,
        CustomerID: customerID,
        Items:      items,
        Status:     StatusDraft,
        CreatedAt:  time.Now().UTC(),
        UpdatedAt:  time.Now().UTC(),
        Total:      total,
    }, nil
}

func (o *Order) Transition(to OrderStatus) error {
    allowed := validTransitions[o.Status]
    for _, s := range allowed {
        if s == to {
            o.Status = to
            o.UpdatedAt = time.Now().UTC()
            return nil
        }
    }
    return fmt.Errorf("%w: %s → %s", ErrInvalidStatus, o.Status, to)
}

func computeTotal(items []OrderItem) Money {
    var total int64
    for _, item := range items {
        total += item.UnitPrice.Amount * int64(item.Quantity)
    }
    return Money{Amount: total, Currency: "EUR"}
}
```

### Ports (interfaces)

```go
// internal/domain/ports.go
package domain

// OrderRepository — port sortant vers la persistance
type OrderRepository interface {
    Save(ctx context.Context, order *Order) error
    FindByID(ctx context.Context, id string) (*Order, error)
    FindByCustomer(ctx context.Context, customerID string, limit, offset int) ([]*Order, error)
    Delete(ctx context.Context, id string) error
}

// EventPublisher — port sortant vers le bus d'événements
type EventPublisher interface {
    Publish(ctx context.Context, event DomainEvent) error
}

type DomainEvent struct {
    ID        string
    Type      string    // "order.created", "order.confirmed"
    OccuredAt time.Time
    Payload   any
}

// OrderUseCase — port entrant (ce que les adaptateurs appellent)
type OrderUseCase interface {
    CreateOrder(ctx context.Context, cmd CreateOrderCmd) (*Order, error)
    ConfirmOrder(ctx context.Context, orderID string) (*Order, error)
    CancelOrder(ctx context.Context, orderID string, reason string) (*Order, error)
    GetOrder(ctx context.Context, orderID string) (*Order, error)
    ListCustomerOrders(ctx context.Context, customerID string, page, size int) ([]*Order, error)
}

type CreateOrderCmd struct {
    CustomerID string
    Items      []OrderItem
}
```

---

## Couche Application (Use Cases)

```go
// internal/application/order_service.go
package application

import (
    "context"
    "fmt"
    "github.com/google/uuid"
    "github.com/myorg/order-service/internal/domain"
    "go.uber.org/zap"
)

type OrderService struct {
    repo      domain.OrderRepository
    publisher domain.EventPublisher
    log       *zap.Logger
}

func NewOrderService(
    repo domain.OrderRepository,
    publisher domain.EventPublisher,
    log *zap.Logger,
) *OrderService {
    return &OrderService{repo: repo, publisher: publisher, log: log}
}

func (s *OrderService) CreateOrder(ctx context.Context, cmd domain.CreateOrderCmd) (*domain.Order, error) {
    id := uuid.New().String()

    order, err := domain.NewOrder(id, cmd.CustomerID, cmd.Items)
    if err != nil {
        return nil, fmt.Errorf("create order: %w", err)
    }

    if err := order.Transition(domain.StatusPending); err != nil {
        return nil, err
    }

    if err := s.repo.Save(ctx, order); err != nil {
        return nil, fmt.Errorf("save order: %w", err)
    }

    event := domain.DomainEvent{
        ID:        uuid.New().String(),
        Type:      "order.created",
        OccuredAt: order.CreatedAt,
        Payload:   order,
    }
    if err := s.publisher.Publish(ctx, event); err != nil {
        // Non-bloquant : on log mais on ne fail pas la commande
        s.log.Warn("failed to publish order.created event",
            zap.String("orderID", order.ID),
            zap.Error(err),
        )
    }

    s.log.Info("order created",
        zap.String("orderID", order.ID),
        zap.String("customerID", cmd.CustomerID),
    )
    return order, nil
}

func (s *OrderService) ConfirmOrder(ctx context.Context, orderID string) (*domain.Order, error) {
    order, err := s.repo.FindByID(ctx, orderID)
    if err != nil {
        return nil, err
    }

    if err := order.Transition(domain.StatusConfirmed); err != nil {
        return nil, err
    }

    if err := s.repo.Save(ctx, order); err != nil {
        return nil, fmt.Errorf("save confirmed order: %w", err)
    }

    s.publisher.Publish(ctx, domain.DomainEvent{
        ID:      uuid.New().String(),
        Type:    "order.confirmed",
        Payload: order,
    })

    return order, nil
}
```

---

## Couche Infrastructure — Adaptateur PostgreSQL

```go
// internal/infrastructure/postgres/repository.go
package postgres

import (
    "context"
    "encoding/json"
    "fmt"
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/myorg/order-service/internal/domain"
)

type OrderRepository struct {
    pool *pgxpool.Pool
}

func NewOrderRepository(pool *pgxpool.Pool) *OrderRepository {
    return &OrderRepository{pool: pool}
}

func (r *OrderRepository) Save(ctx context.Context, order *domain.Order) error {
    itemsJSON, _ := json.Marshal(order.Items)

    _, err := r.pool.Exec(ctx, `
        INSERT INTO orders (id, customer_id, items, status, total_amount, total_currency, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        ON CONFLICT (id) DO UPDATE SET
            status         = EXCLUDED.status,
            updated_at     = EXCLUDED.updated_at
    `,
        order.ID, order.CustomerID, itemsJSON, order.Status,
        order.Total.Amount, order.Total.Currency,
        order.CreatedAt, order.UpdatedAt,
    )
    if err != nil {
        return fmt.Errorf("postgres save order: %w", err)
    }
    return nil
}

func (r *OrderRepository) FindByID(ctx context.Context, id string) (*domain.Order, error) {
    row := r.pool.QueryRow(ctx, `
        SELECT id, customer_id, items, status, total_amount, total_currency, created_at, updated_at
        FROM orders WHERE id = $1
    `, id)

    var o domain.Order
    var itemsJSON []byte
    err := row.Scan(
        &o.ID, &o.CustomerID, &itemsJSON, &o.Status,
        &o.Total.Amount, &o.Total.Currency,
        &o.CreatedAt, &o.UpdatedAt,
    )
    if err != nil {
        return nil, domain.ErrNotFound
    }
    json.Unmarshal(itemsJSON, &o.Items)
    return &o, nil
}
```

### Migration SQL

```sql
-- internal/infrastructure/postgres/migrations/001_orders.sql
CREATE TABLE IF NOT EXISTS orders (
    id              TEXT PRIMARY KEY,
    customer_id     TEXT NOT NULL,
    items           JSONB NOT NULL DEFAULT '[]',
    status          TEXT NOT NULL DEFAULT 'pending',
    total_amount    BIGINT NOT NULL,
    total_currency  TEXT NOT NULL DEFAULT 'EUR',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
```

---

## Adaptateur HTTP

```go
// internal/infrastructure/http/handler.go
package httphandler

import (
    "encoding/json"
    "errors"
    "net/http"
    "github.com/myorg/order-service/internal/domain"
)

type Handler struct {
    orders domain.OrderUseCase
}

func NewHandler(orders domain.OrderUseCase) *Handler {
    return &Handler{orders: orders}
}

type createOrderRequest struct {
    CustomerID string             `json:"customer_id"`
    Items      []orderItemRequest `json:"items"`
}

type orderItemRequest struct {
    ProductID string `json:"product_id"`
    Name      string `json:"name"`
    Quantity  int    `json:"quantity"`
    UnitPrice int64  `json:"unit_price"` // centimes
}

func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var req createOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    items := make([]domain.OrderItem, len(req.Items))
    for i, it := range req.Items {
        items[i] = domain.OrderItem{
            ProductID: it.ProductID,
            Name:      it.Name,
            Quantity:  it.Quantity,
            UnitPrice: domain.Money{Amount: it.UnitPrice, Currency: "EUR"},
        }
    }

    order, err := h.orders.CreateOrder(r.Context(), domain.CreateOrderCmd{
        CustomerID: req.CustomerID,
        Items:      items,
    })
    if err != nil {
        if errors.Is(err, domain.ErrMinimumAmount) {
            writeError(w, http.StatusUnprocessableEntity, err.Error())
            return
        }
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(order)
}

func writeError(w http.ResponseWriter, code int, msg string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    json.NewEncoder(w).Encode(map[string]string{"error": msg})
}
```

### Health Checks

```go
// internal/infrastructure/http/health.go
package httphandler

import (
    "encoding/json"
    "net/http"
)

type HealthHandler struct {
    db interface{ Ping(context.Context) error }
}

func (h *HealthHandler) Liveness(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}

func (h *HealthHandler) Readiness(w http.ResponseWriter, r *http.Request) {
    if err := h.db.Ping(r.Context()); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(map[string]string{"status": "unavailable", "reason": "database"})
        return
    }
    json.NewEncoder(w).Encode(map[string]string{"status": "ready"})
}
```

---

## Point de composition (main.go)

```go
// cmd/server/main.go
package main

import (
    "context"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
    "go.uber.org/zap"

    "github.com/myorg/order-service/internal/application"
    httphandler "github.com/myorg/order-service/internal/infrastructure/http"
    "github.com/myorg/order-service/internal/infrastructure/kafka"
    "github.com/myorg/order-service/internal/infrastructure/postgres"
)

func main() {
    log, _ := zap.NewProduction()
    defer log.Sync()

    // Infrastructure
    pool, err := pgxpool.New(context.Background(), os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal("cannot connect to database", zap.Error(err))
    }
    defer pool.Close()

    repo      := postgres.NewOrderRepository(pool)
    publisher := kafka.NewPublisher(os.Getenv("KAFKA_BROKERS"))

    // Application
    svc := application.NewOrderService(repo, publisher, log)

    // HTTP
    handler := httphandler.NewHandler(svc)
    health  := &httphandler.HealthHandler{db: pool}

    mux := http.NewServeMux()
    mux.HandleFunc("POST /orders",              handler.CreateOrder)
    mux.HandleFunc("GET /orders/{id}",          handler.GetOrder)
    mux.HandleFunc("POST /orders/{id}/confirm", handler.ConfirmOrder)
    mux.HandleFunc("POST /orders/{id}/cancel",  handler.CancelOrder)
    mux.HandleFunc("GET /healthz",              health.Liveness)
    mux.HandleFunc("GET /readyz",               health.Readiness)

    srv := &http.Server{
        Addr:         ":" + getEnv("PORT", "8080"),
        Handler:      mux,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
    }

    // Graceful shutdown
    go func() {
        log.Info("starting server", zap.String("addr", srv.Addr))
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal("server error", zap.Error(err))
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Info("shutting down gracefully...")
    ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}

func getEnv(key, fallback string) string {
    if v, ok := os.LookupEnv(key); ok {
        return v
    }
    return fallback
}
```

---

## Tests d'intégration

```go
// internal/infrastructure/postgres/repository_test.go
//go:build integration

package postgres_test

import (
    "context"
    "os"
    "testing"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/myorg/order-service/internal/domain"
    "github.com/myorg/order-service/internal/infrastructure/postgres"
)

func TestOrderRepository_SaveAndFind(t *testing.T) {
    pool, _ := pgxpool.New(context.Background(), os.Getenv("TEST_DATABASE_URL"))
    defer pool.Close()

    repo := postgres.NewOrderRepository(pool)

    order, _ := domain.NewOrder("test-1", "cust-1", []domain.OrderItem{
        {ProductID: "p1", Quantity: 1, UnitPrice: domain.Money{Amount: 1500, Currency: "EUR"}},
    })

    if err := repo.Save(context.Background(), order); err != nil {
        t.Fatalf("save failed: %v", err)
    }

    found, err := repo.FindByID(context.Background(), order.ID)
    if err != nil {
        t.Fatalf("find failed: %v", err)
    }
    if found.ID != order.ID {
        t.Errorf("expected %s, got %s", order.ID, found.ID)
    }
}
```

```bash
# Lancer les tests unitaires (sans infra)
go test ./internal/domain/... ./internal/application/...

# Lancer les tests d'intégration
TEST_DATABASE_URL=postgres://localhost/orders_test go test -tags integration ./internal/infrastructure/...
```

---

<div class="pulse-glow" style="padding: 1rem; border-radius: 0.5rem; margin: 1rem 0; background: #f5f3ff; border-left: 4px solid #8b5cf6;">

**Récapitulatif des conventions Go pour microservices :**

- `internal/` : code privé au service, non importable depuis l'extérieur
- `cmd/` : points d'entrée (un sous-dossier par binaire)
- Interfaces petites (1-3 méthodes), définies côté consommateur
- Errors wrappées avec `fmt.Errorf("...: %w", err)` pour la traçabilité
- Context propagé partout pour annulation et timeouts
- Configuration uniquement via variables d'environnement

</div>

<span class="rainbow">🦫 Votre microservice Go est structuré. Prochaine étape : le conteneuriser avec Docker ! ➜</span>
