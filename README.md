# Go-Clean-Architecture-Scaffold

Step-by-step AI agent instructions for building and scaling Go microservices. Designed to generate deterministic, production-grade code with built-in Redis caching and observability.

This repository provides a concrete implementation guide and an AI agent workflow to generate consistent Go microservices using Clean Architecture principles.

## Features & Tech Stack

- **Framework**: [Gin](https://gin-gonic.com/) for HTTP routing
- **ORM**: [GORM](https://gorm.io/) with PostgreSQL
- **Caching**: [Redis](https://redis.io/) cache-aside pattern
- **Observability**: OpenTelemetry tracing & Prometheus metrics
- **Documentation**: Swagger via `swaggo/swag`
- **Testing**: Testcontainers-go, k6 load testing, and mocking
- **Dependency Injection**: Manual DI via configuration composition root

## Usage with AI Assistants

This repository includes a pre-defined AI workflow in [`.agent/workflows/CLEAN_ARCH_SKILL.md`](.agent/workflows/CLEAN_ARCH_SKILL.md).
You can use the `/CLEAN_ARCH_SKILL` slash command with your AI coding assistant (like Antigravity) when starting a new project or adding a new domain feature to ensure the assistant follows these strict architectural guidelines.

## Quick Start for New Projects

To rapidly setup an environment that follows the workflow rules:

1. Initialize Go module: `go mod init <module-name>`
2. Create the Clean Architecture directory structure (`cmd`, `internal/config`, `delivery`, `domain`, `infrastructure`, `utils`, `pkg`...)
4. Add framework dependencies: `go get github.com/gin-gonic/gin gorm.io/gorm gorm.io/driver/postgres github.com/redis/go-redis/v9 github.com/sirupsen/logrus`
5. Add OTel dependencies: `go get go.opentelemetry.io/otel go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin`
6. Add Swagger: `go get github.com/swaggo/swag github.com/swaggo/gin-swagger github.com/swaggo/files`
7. Add test dependencies: `go get github.com/stretchr/testify`
8. Implement the core components (`config.go`, `app.go`, and `cmd/app/main.go`)
9. Create `.env` and `docker-compose.yml`

## Project Structure Overview

```text
cmd/
├── app/main.go            # HTTP server entrypoint
├── migrate/main.go        # DB migration entrypoint
└── seeder/main.go         # DB seeder entrypoint

internal/
├── config/                # Viper config (.env) and DI wiring (app.go)
├── delivery/http/         # HTTP Controllers and Route Registration
├── domain/                # Business logic: Entities, Repository & Use Case Interfaces
├── infrastructure/        # Implementations: DB, Caching, Repositories, Seeder
└── utils/mapper/          # Mappers: Entity ↔ Domain ↔ DTO

pkg/dto/                   # Public Request/Response Data Transfer Objects (DTO)
docs/                      # Swagger specifications
test/                      # Unit, Integration, and k6 tests
```

## Dependency Flow

**Strict Rule:** `Delivery → Domain ← Infrastructure`

- **Domain** depends on **nothing** (pure business logic, isolated from DB/HTTP).
- **Infrastructure** implements Domain interfaces.
- **Delivery** calls Use Cases via Domain interfaces.
- **Config** wires dependencies (`app.go` serves as Composition Root).

For full details on naming conventions, testing strategies, cache key configurations, and how to add new domains, refer to the included workflow guide.
