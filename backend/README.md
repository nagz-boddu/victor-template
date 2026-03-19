# Backend

Spring Boot 4 application with Spring Modulith, OAuth2 Resource Server, and a strict API-first workflow.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Architecture](#architecture)
- [Spring Profiles](#spring-profiles)
- [Maven Profiles](#maven-profiles)
- [Testing](#testing)
- [Code Quality](#code-quality)
- [API-First Workflow](#api-first-workflow)
- [Docker](#docker)

## Prerequisites

- **Java** 25
- **Maven** 3.9+ (or use the included `./mvnw` wrapper)
- **Docker** (required for integration tests via Testcontainers)
- **PostgreSQL** for local development (see Docker command below)
- **Keycloak** for OAuth2/OIDC (available via the root `docker-compose.yml`)

### Local PostgreSQL

```bash
docker run --name spring-postgres -p 5432:5432 \
  -e POSTGRES_DB=spring_db \
  -e POSTGRES_USER=spring_user \
  -e POSTGRES_PASSWORD=secure_password \
  -d postgres:18.1-alpine
```

## Quick Start

```bash
# Regenerate API interfaces from the OpenAPI spec
./mvnw clean compile

# Run all tests (unit + integration + contract)
./mvnw verify

# Start the server (defaults to 'local' profile)
./mvnw spring-boot:run
```

The API server starts on `http://localhost:8081/api` with the `local` profile. Actuator/management endpoints are exposed on a separate port (see [Spring Profiles](#spring-profiles)).

## Project Structure

```
backend/
├── src/main/java/com/example/demo/
│   ├── DemoApplication.java              # Entry point
│   ├── api/                              # OpenAPI-generated interfaces (DO NOT EDIT)
│   ├── greeting/                         # Greeting domain module
│   │   ├── controller/                   #   Implements GreetingsApi
│   │   ├── service/                      #   Business logic + event publishing
│   │   ├── repository/                   #   Spring Data JPA + RevisionRepository
│   │   ├── mapper/                       #   MapStruct DTO-to-entity mapping
│   │   ├── model/                        #   @Audited JPA entity
│   │   ├── event/                        #   GreetingCreatedEvent (domain event)
│   │   ├── exception/                    #   Domain-specific exceptions
│   │   └── dto/                          #   Audit history DTOs
│   ├── user/                             # User domain module
│   │   ├── controller/                   #   Implements UserApi (GET /v1/me)
│   │   ├── service/                      #   UserDetailsService
│   │   ├── repository/                   #   User persistence
│   │   ├── model/                        #   UserDetailsImpl
│   │   └── config/                       #   SystemUserLoader
│   ├── audit/                            # Audit module (event consumer)
│   │   ├── BusinessActivityLog.java      #   Immutable audit log entity
│   │   └── internal/                     #   @ApplicationModuleListener
│   └── common/                           # Shared kernel
│       ├── audit/                        #   Hibernate Envers revision entity
│       ├── config/                       #   Security, JPA auditing, web config
│       ├── domain/                       #   AbstractBaseEntity (TSID PK, audit fields)
│       ├── exception/                    #   GlobalExceptionHandler (RFC 7807)
│       └── repository/                   #   FunctionalIdGenerator
├── src/main/resources/
│   ├── application.properties            # Base config
│   ├── application-local.properties      # Local dev (default)
│   ├── application-dev.properties        # Shared dev environment
│   ├── application-staging.properties    # Staging
│   ├── application-prod.properties       # Production
│   └── db/migration/                     # Flyway SQL migrations
├── src/test/java/com/example/demo/
│   ├── testsupport/                      # Test infrastructure
│   │   ├── AbstractIntegrationTest.java  #   Base class for ITs
│   │   ├── AbstractControllerIT.java     #   Base for HTTP-level ITs
│   │   ├── TestcontainersConfiguration/  #   Singleton PostgreSQL container
│   │   ├── SchemaIsolationExtension/     #   Per-test schema isolation
│   │   └── TestJwtUtils.java             #   JWT minting for tests
│   ├── architecture/                     # ArchUnit + Modulith tests
│   ├── contract/                         # OpenAPI contract validator
│   └── greeting/                         # Domain-specific tests
├── checkstyle.xml                        # Checkstyle ruleset
├── checkstyle-suppressions.xml           # Checkstyle suppressions
├── spotbugs-exclude.xml                  # SpotBugs exclusions
├── Dockerfile                            # Multi-stage build (builder, ci, production)
└── pom.xml
```

## Architecture

The application follows **Screaming Architecture** -- organized by business domain, then by technical layer within each domain. Module boundaries are enforced at compile time by **Spring Modulith**.

### Modules

| Module | Package | Purpose |
|--------|---------|---------|
| `greeting` | `com.example.demo.greeting` | Core domain. Full CRUD with domain events. |
| `user` | `com.example.demo.user` | User management, JWT principal resolution. |
| `audit` | `com.example.demo.audit` | Business activity logging. Consumes events via Modulith outbox. |
| `common` | `com.example.demo.common` | Open shared kernel -- base entities, security, exceptions. |
| `api` | `com.example.demo.api.v1` | OpenAPI-generated interfaces (read-only). |

### Data Flow

```
Request -> Controller (receives DTO) -> Mapper (DTO to Entity)
  -> Service (business logic on Entity) -> Repository
  -> Mapper (Entity to DTO) -> Controller (returns DTO) -> Response
```

### Key Patterns

- **TSID** surrogate primary keys (DB-performant, loosely chronological) + human-readable functional IDs (e.g., `GRE-2025-000042`)
- **Hibernate Envers** for full entity audit history
- **Spring Modulith transactional event outbox** (JPA-backed, DELETE completion mode)
- **RFC 7807 ProblemDetail** for all error responses with `traceId` correlation
- **MapStruct** for compile-time DTO-to-entity mapping

## Spring Profiles

| Profile | Use Case | Server Port | Management Port | Database |
|---------|----------|-------------|-----------------|----------|
| `local` (default) | Local development | 8081 | 8082 | PostgreSQL localhost:5432 |
| `dev` | Shared dev environment | 8080 | 8081 | PostgreSQL via env vars |
| `staging` | QA / pre-production | 8080 | 8081 | PostgreSQL via env vars |
| `prod` | Production | 8080 | 8081 | PostgreSQL via env vars |

All profiles use context path `/api`. Management endpoints use base path `/management` (not the default `/actuator`).

## Maven Profiles

| Profile | Command | Purpose |
|---------|---------|---------|
| *(default)* | `./mvnw verify` | Unit tests + integration tests |
| `quality-check` | `./mvnw verify -Pquality-check` | Static analysis only (Checkstyle, PMD, SpotBugs) |
| `integration-tests` | `./mvnw verify -Pintegration-tests` | Skip unit tests, run only ITs |
| `parallel-its` | `./mvnw verify -Pparallel-its` | Integration tests with 4-way parallelism |
| `ci-fast` | `./mvnw verify -Pci-fast` | Skip all quality checks and ITs |
| `full-build` | `./mvnw verify -Pfull-build` | Full verification including coverage |
| `coverage-enforce` | `./mvnw verify -Pcoverage-enforce` | Enforce 80% instruction / 70% branch coverage |

## Testing

### Test Pyramid

| Level | Convention | Tools | Context |
|-------|-----------|-------|---------|
| **Unit** | `*Test.java` | JUnit 5, Mockito | No Spring context |
| **Slice** | `*Test.java` | `@WebMvcTest`, `@DataJpaTest` | Partial context |
| **Integration** | `*IT.java` | `@SpringBootTest`, RestAssured | Full context + Testcontainers |
| **Architecture** | `*IT.java` / `*Test.java` | ArchUnit, Spring Modulith | Compile-time verification |
| **Contract** | via ITs | swagger-request-validator | Validates requests/responses against OpenAPI spec |

### Test Infrastructure

- **Testcontainers**: Singleton PostgreSQL 16 container shared across all ITs
- **Schema isolation**: Each IT runs in its own PostgreSQL schema (parallel-safe)
- **JWT minting**: `TestJwtUtils` creates HS256 tokens for auth testing
- **Keycloak Testcontainer**: Available for full OIDC flow integration tests
- **Container reuse**: Add `testcontainers.reuse.enable=true` to `~/.testcontainers.properties` for faster local runs

### Running Tests

```bash
./mvnw test                         # Unit tests only
./mvnw verify                       # Unit + integration tests
./mvnw verify -Pintegration-tests   # Integration tests only
./mvnw verify -Pparallel-its        # Parallel integration tests
```

## Code Quality

| Tool | Purpose | Command |
|------|---------|---------|
| **Checkstyle** | Code style and naming conventions | `./mvnw checkstyle:check` |
| **PMD** | Static code analysis | `./mvnw pmd:check` |
| **SpotBugs** | Bug detection (includes FindSecBugs) | `./mvnw spotbugs:check` |
| **ArchUnit** | Architectural constraints (layering, naming) | Runs as part of `./mvnw verify` |
| **JaCoCo** | Code coverage reporting | `./mvnw verify -Pfull-build` |
| **SonarQube** | Cloud code analysis | `./mvnw sonar:sonar` |
| **CycloneDX** | SBOM generation | Generated during `./mvnw package` at `target/sbom/backend-sbom.json` |

Run all quality checks at once:

```bash
./mvnw verify -Pquality-check
```

## API-First Workflow

Controllers implement interfaces generated from `api/specification/openapi.yaml` by the `openapi-generator-maven-plugin`. **Never manually create controller interfaces.**

```bash
# After editing openapi.yaml, regenerate Java interfaces:
./mvnw clean compile

# The generated interfaces appear in:
# target/generated-sources/openapi/com/example/demo/api/v1/
```

## Docker

The Dockerfile supports three build stages:

```bash
# Full build from source (production stage)
docker build -t demo-backend:latest -f backend/Dockerfile .

# CI stage (requires pre-built JAR in backend/target/)
docker build --target ci -t demo-backend:ci -f backend/Dockerfile .
```

**Note:** The Docker build context must be the **repository root** (not `backend/`), as the Dockerfile references sibling directories.

The image uses `gcr.io/distroless/java25-debian13` as the runtime base and runs as a non-root user.
