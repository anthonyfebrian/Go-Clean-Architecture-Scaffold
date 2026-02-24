---
description: Clean Architecture scaffold for Go microservices (Gin + GORM + Redis + OTel)
---

# Clean Architecture — Go Microservice

Use this workflow whenever the user asks to create a new Go microservice or add a new domain/feature to an existing one. Follow the structure, naming conventions, and patterns documented below **exactly**.

---

## 1. Project Structure

```
cmd/
├── app/main.go            # HTTP server entrypoint
├── migrate/main.go        # DB migration entrypoint
└── seeder/main.go         # DB seeder entrypoint

internal/
├── config/
│   ├── config.go          # Viper config struct (env vars via mapstructure tags)
│   └── app.go             # DI wiring: Initialize() builds the full dependency graph
├── delivery/
│   └── http/
│       ├── controller/    # One controller per domain (binds request, calls use case, returns DTO)
│       └── route/         # Route registration (Gin groups)
├── domain/
│   ├── entity/            # Core business models (plain structs, no DB/HTTP tags)
│   ├── repository/        # Repository interfaces (consumed by use cases)
│   └── usecase/
│       ├── <name>_use_case.go               # Use case interface
│       └── implementation/<name>_impl.go    # Use case implementation
├── infrastructure/
│   ├── datasource/
│   │   ├── entity/        # GORM models (DB table mapping with gorm tags)
│   │   ├── persistance/
│   │   │   ├── <name>_datasource.go              # DataSource interface
│   │   │   └── implementation/<name>_impl.go     # GORM implementation
│   │   └── cache/
│   │       ├── <name>_cache.go                   # Cache interface
│   │       ├── key/cache_key.go                  # Deterministic cache key builder
│   │       └── implementation/<name>_impl.go     # Redis implementation
│   ├── db/
│   │   ├── postgres.go    # GORM DB connection setup
│   │   └── redis.go       # Redis client setup
│   ├── repository/        # Repository implementation (cache-aside pattern)
│   └── seeder/            # CSV/data seeders
└── utils/
    └── mapper/            # Mapper functions: Entity ↔ Domain ↔ DTO

pkg/
└── dto/                   # Public request/response DTOs (form/json tags + validation)

docs/                      # Auto-generated Swagger files (swag init)

test/
├── delivery/              # Controller unit tests
├── domain/                # Use case + mapper tests
├── infrastructure/        # Repository + cache key tests
├── integration/           # E2E with testcontainers-go
├── k6/                    # Load tests
└── mocks/                 # Auto-generated mocks (mockery)
```

---

## 2. Dependency Flow (strict, no reverse imports)

```
Delivery → Domain ← Infrastructure
             ↑
           pkg/dto (shared DTOs)
```

- **Domain** depends on nothing (pure business logic).
- **Infrastructure** implements domain interfaces.
- **Delivery** calls use cases via domain interfaces.
- **Config/app.go** is the only place that knows all concrete types (DI composition root).

---

## 3. Adding a New Domain Feature

Follow these steps in order:

### Step 1 — Domain Entity
Create `internal/domain/entity/<name>.go`:
```go
package entity

type Order struct {
    ID          string
    // ... business fields, NO gorm/json tags
}
```

If needed, add a filter struct: `internal/domain/entity/<name>_filter.go`.

### Step 2 — Domain Interfaces
Create repository interface `internal/domain/repository/<name>_repository.go`:
```go
type OrderRepository interface {
    FindAll(ctx context.Context, filter *entity.OrderFilter) (*pagination.Pagination[entity.Order], error)
    FindByID(ctx context.Context, id string) (*entity.Order, error)
    Create(ctx context.Context, order *entity.Order) error
}
```

Create use case interface `internal/domain/usecase/<action>_use_case.go`:
```go
type GetAllOrderUseCase interface {
    Execute(ctx context.Context, filter *entity.OrderFilter) (*pagination.Pagination[entity.Order], error)
}
```

### Step 3 — Infrastructure: Datasource Entity
Create GORM model `internal/infrastructure/datasource/entity/<name>_entity.go`:
```go
type OrderEntity struct {
    ID        uint           `gorm:"primaryKey"`
    Name      string         `gorm:"not null"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

func (OrderEntity) TableName() string { return "orders" }
```

### Step 4 — Infrastructure: DataSource Implementation
Interface at `internal/infrastructure/datasource/persistance/<name>_datasource.go`.
Implementation at `internal/infrastructure/datasource/persistance/implementation/<name>_datasource_impl.go`:
- Use `impl.DB.WithContext(ctx)` for all queries (enables OTel tracing).
- Build dynamic WHERE clauses from filter fields.
- Calculate offset pagination and total pages.

### Step 5 — Infrastructure: Cache
Interface at `internal/infrastructure/datasource/cache/<name>_cache.go`.
Cache key at `internal/infrastructure/datasource/cache/key/cache_key.go` — add a new function.
Implementation at `internal/infrastructure/datasource/cache/implementation/<name>_cache_impl.go`:
- JSON marshal/unmarshal for Redis storage.
- Composite cache key from ALL filter parameters.

### Step 6 — Infrastructure: Repository (Cache-Aside)
Create `internal/infrastructure/repository/<name>_repository_impl.go`:
```go
func (impl *OrderRepositoryImpl) FindAll(ctx context.Context, filter *entity.OrderFilter) (...) {
    // 1. Try cache
    cached, err := impl.Cache.GetAll(ctx, filter)
    if err == nil {
        return cached, nil  // cache hit
    }
    // 2. Query database
    entities, err := impl.DataSource.GetAll(ctx, filter)
    if err != nil {
        return nil, response.InternalError(op, err)
    }
    // 3. Map to domain
    result := mapper.ToOrderPaginatedDomain(entities)
    // 4. Set cache (fire-and-forget on error)
    impl.Cache.SetAll(ctx, filter, result)
    return result, nil
}
```

### Step 7 — Mappers
Create `internal/utils/mapper/<name>_entity_mapper.go` (Datasource Entity ↔ Domain).
Create `internal/utils/mapper/<name>_mapper.go` (Domain → DTO).

Key rules:
- `ToXxxDomainList()` always returns non-nil slice (use `make([]T, len(...))`).
- `ToXxxResponseList()` same pattern — prevents `"data": null` in JSON.

### Step 8 — DTOs
Create `pkg/dto/<name>_request.go` (query params with `form` + `binding` tags).
Create `pkg/dto/<name>_response.go` (JSON response with `json` tags).

Validation uses `go-playground/validator` via Gin's `binding` tag:
```go
type GetAllOrderQuery struct {
    Page  int    `form:"page"  binding:"omitempty,min=1"`
    Limit int    `form:"limit" binding:"omitempty,min=1,max=100"`
    Sort  string `form:"sort"  binding:"omitempty,oneof=asc desc"`
}
```

### Step 9 — Use Case Implementation
Create `internal/domain/usecase/implementation/<action>_use_case_impl.go`:
- Start OTel span: `tracer := otel.Tracer("usecase")`.
- Log operation start/end with duration using logrus.
- Delegate to repository.

### Step 10 — Controller
Create `internal/delivery/http/controller/<name>_controller.go`:
- Bind query/body with `c.ShouldBindQuery()` or `c.ShouldBindJSON()`.
- On bind error: `c.Error(err); return`.
- Apply defaults for optional fields.
- Build domain filter, call use case, map result to DTO, return JSON.
- Add Swagger annotations (`// @Summary`, `// @Param`, etc.).

### Step 11 — Route Registration
Add route group in `internal/delivery/http/route/route.go`:
```go
func (rc *RouteConfig) SetupOrderRoutes() {
    orderRoutes := rc.App.Group("/order")
    {
        orderRoutes.GET("/", rc.OrderController.GetAll)
        orderRoutes.GET("/:id", rc.OrderController.GetByID)
        orderRoutes.POST("/", rc.OrderController.Create)
    }
}
```
Call `rc.SetupOrderRoutes()` from `Setup()`.

### Step 12 — DI Wiring
Wire everything in `internal/config/app.go` → `Initialize()`:
```go
// DataSource → Cache → Repository → UseCase → Controller
orderDataSource := persistance.NewOrderDataSourceImpl(config.DB, config.Log)
orderCache := cache_impl.NewOrderCacheImpl(config.Redis, config.CacheTTLDefault, config.Log)
orderRepo := repository.NewOrderRepositoryImpl(orderDataSource, orderCache, config.Log)
getAllOrderUC := usecase_impl.NewGetAllOrderUseCaseImpl(orderRepo, config.Log)
orderController := controller.NewOrderController(config.Log, getAllOrderUC)
```

---

## 4. Conventions & Rules

| Convention | Rule |
|---|---|
| **Error handling** | Controllers push errors via `c.Error(err)`, never write error responses directly. |
| **Context propagation** | Always pass `ctx` from Gin → UseCase → Repository → DataSource for OTel tracing |
| **Naming** | Interfaces in domain package, implementations in `implementation/` sub-package |
| **Constructors** | `New<Type>Impl(...)` returns the interface type, not the concrete struct |
| **Logging** | Use structured `logrus` fields. Include `operation`, `duration`, `error` |
| **Cache keys** | Must be deterministic and include ALL filter parameters |
| **Pagination** | Defined in `internal/utils/pagination/` for generic pagination |
| **Validation** | Use `binding` tags on DTO structs, never validate in controllers manually |
| **Swagger** | Annotate every controller method; regenerate with `swag init -g cmd/app/main.go -o docs --parseDependency --parseInternal` |

---

## 6. Infrastructure Setup

### docker-compose.yml services (always include):
- **app** — the Go service
- **postgres** — PostgreSQL 18 Alpine
- **redis** — Redis 7 Alpine
- **jaeger** — OpenTelemetry tracing UI
- **prometheus** — metrics scraping
- **grafana** — dashboards (provisioned via `grafana/` directory)

### Environment variables (Config struct fields):
```env
DB_SOURCE=postgres://user:pass@host:5432/dbname?sslmode=disable
SERVER_ADDRESS=0.0.0.0:8080
REDIS_ADDRESS=host:6379
REDIS_PASSWORD=
REDIS_DB=0
CACHE_TTL_DEFAULT=15m
OTEL_EXPORTER_OTLP_ENDPOINT=http://host:4317
OTEL_SERVICE_NAME=service-name
```

---

## 7. Testing Strategy

| Layer | Location | Tools |
|---|---|---|
| Controller | `test/delivery/` | testify + mock use case |
| UseCase | `test/domain/` | testify + mock repository |
| Mapper | `test/domain/` | testify (pure functions) |
| Repository | `test/infrastructure/` | testify + mock datasource + mock cache |
| Cache key | `test/infrastructure/` | testify (pure function) |
| Integration | `test/integration/` | testcontainers-go (real Postgres + Redis) |
| Load | `test/k6/` | k6 (smoke, average, stress, spike) |

Mock generation:
```bash
mockery --dir=internal/domain/repository --name=<Interface> --output=test/mocks --outpkg=mocks
```

---

## 8. Quick Start Checklist for New Project

// turbo-all

1. Initialize Go module: `go mod init <module-name>`
2. Create the directory structure from Section 1
3. Add framework dependencies: `go get github.com/gin-gonic/gin gorm.io/gorm gorm.io/driver/postgres github.com/redis/go-redis/v9 github.com/sirupsen/logrus`
5. Add OTel dependencies: `go get go.opentelemetry.io/otel go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin`
6. Add Swagger: `go get github.com/swaggo/swag github.com/swaggo/gin-swagger github.com/swaggo/files`
7. Add test dependencies: `go get github.com/stretchr/testify`
8. Create `config.go`, `app.go`, and `cmd/app/main.go` following the patterns above
9. Create `.env` file with the environment variables
10. Create `docker-compose.yml` with all infrastructure services
11. Create `Dockerfile` (multi-stage: build + alpine runtime)
12. Follow Section 3 for each domain feature