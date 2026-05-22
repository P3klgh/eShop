# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Build
```bash
dotnet build eShop.Web.slnf        # web projects only (faster, recommended)
dotnet build eShop.slnx            # full solution
```

### Run
```bash
dotnet run --project src/eShop.AppHost/eShop.AppHost.csproj
```
After starting, the Aspire dashboard URL (with login token) is printed to the console. Docker Desktop must be running — Aspire manages PostgreSQL, Redis, and RabbitMQ containers automatically.

### Test
```bash
dotnet test eShop.Web.slnf         # unit + functional tests
dotnet test eShop.Web.slnf --filter "FullyQualifiedName~SomeTest"  # single test
```

### E2E (Playwright)
Node modules must be installed once from the repo root:
```bash
npm ci
```
Then run tests — Playwright auto-starts the AppHost locally (via `webServer` in `playwright.config.ts`):
```bash
npx playwright test
```
Set `ESHOP_USE_HTTP_ENDPOINTS=1` to disable HTTPS (required in CI; `playwright.config.ts` sets this automatically in CI via the workflow). The base URL is `http://localhost:5045`. In CI, `reuseExistingServer` is disabled so Playwright always starts a fresh instance.

## Architecture

This is a microservices e-commerce reference application orchestrated with **.NET Aspire 13.1** targeting **.NET 10**.

### Services (src/)
| Project | Role | Storage |
|---------|------|---------|
| `eShop.AppHost` | Aspire host — wires all services together | — |
| `eShop.ServiceDefaults` | Shared library: OpenTelemetry, health checks, service discovery | — |
| `WebApp` | Blazor Server frontend | — |
| `Catalog.API` | Product catalog REST API | PostgreSQL + pgvector |
| `Basket.API` | Shopping basket **gRPC** API | Redis |
| `Ordering.API` | Order management REST API | PostgreSQL |
| `Identity.API` | Duende IdentityServer (OAuth 2.0/OIDC) | PostgreSQL |
| `Ordering.Domain` / `Ordering.Infrastructure` | Domain model + EF Core for Ordering | — |
| `OrderProcessor` / `PaymentProcessor` | Background workers consuming RabbitMQ events | PostgreSQL |
| `Webhooks.API` + `WebhookClient` | Outbound webhook system | PostgreSQL |
| `EventBus` / `EventBusRabbitMQ` | Event bus abstraction + RabbitMQ implementation | — |
| `IntegrationEventLogEF` | Outbox pattern for integration events | — |

### Key Architectural Patterns

**Event-Driven:** Services communicate via RabbitMQ integration events (publish/subscribe). The outbox pattern (`IntegrationEventLogEF`) ensures at-least-once delivery alongside EF Core transactions.

**gRPC for Basket:** `Basket.API` is a gRPC service with a `.proto` file. The client-side generated code lives in `src/ClientApp/Services/Basket/Protos/`. Use `protoc` / `grpc_tools_dotnet_protoc` for regeneration.

**Database per Service:** Each service has its own PostgreSQL database. EF Core migrations are run at startup via `MigrateDbContextExtensions` (linked source file shared across projects).

**Aspire Orchestration:** No `docker-compose.yml` — infrastructure containers and service wiring are defined entirely in `src/eShop.AppHost/Program.cs`. This is the single source of truth for how services discover each other.

**Service Defaults:** All services reference `eShop.ServiceDefaults` which configures OpenTelemetry, HTTP resilience, service discovery, OpenAPI docs, and health checks consistently.

### Testing Strategy
- **Unit tests** (`Basket.UnitTests`, `Ordering.UnitTests`): MSTest, fast, no containers.
- **Functional/integration tests** (`Catalog.FunctionalTests`, `Ordering.FunctionalTests`): Spin up real containers via Aspire test host; slower but test actual DB behavior.
- **E2E** (`e2e/`): Playwright TypeScript tests that exercise the full running application. Login credentials: `bob` / `Pass123$`.

### Central Package Management
All NuGet versions are pinned in `Directory.Packages.props`. Do not add `Version` attributes to individual `.csproj` `<PackageReference>` elements — add the version entry to `Directory.Packages.props` instead.

### Shared Source Files
Some source files (e.g., `ActivityExtensions.cs`, `MigrateDbContextExtensions.cs`) are shared across projects using `<Compile Include="..." Link="..." />` in `.csproj` files rather than a shared library — this is intentional.

### AI / OpenAI Integration
The Catalog service supports optional AI-powered search via OpenAI embeddings (pgvector). In `AppHost/Program.cs`, set `useOpenAI = true` or `useOllama = true` to enable. Connection strings go in user secrets or `appsettings.json`.

## Gotchas

**`TreatWarningsAsErrors` is globally enabled** (`Directory.Build.props`). Compiler warnings are build errors — fix them or they'll block CI.

**EF migrations run automatically at startup.** All four services (`Catalog.API`, `Identity.API`, `Ordering.API`, `Webhooks.API`) call `MigrateDbContextExtensions` on startup. Never run `dotnet ef database update` manually.

**`OrderProcessor` waits for `Ordering.API` to be healthy before starting** (wired in AppHost). This is intentional — `Ordering.API` owns the EF migrations for `orderingdb`. If `OrderProcessor` is stuck in a restart loop, check that `Ordering.API` passed its health check first.
