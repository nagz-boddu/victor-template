# Template App

A production-grade monorepo template demonstrating **API-first development**, **modern Java with Spring Boot 4**, **React 19 with TypeScript**, and **documentation-as-code** practices. Built to serve as a reference architecture for teams that value type safety, strict contracts, security, and software quality.

## Why This Template?

Most "starter" projects give you a basic CRUD app with no guidance on how to scale it. This template goes further:

- **Contract-First API Design** -- The OpenAPI specification is the single source of truth. Backend and frontend code are generated from it, eliminating drift between teams.
- **Security by Default** -- OAuth2/OIDC with the BFF (Backend-for-Frontend) pattern via Spring Cloud Gateway, Keycloak as the identity provider, and automated secret scanning (Gitleaks, Semgrep) in CI.
- **Modular Backend** -- Spring Modulith enforces bounded contexts at the package level with a transactional event outbox, preventing the "big ball of mud" as the codebase grows.
- **Documentation That Ships** -- Arc42 architecture docs, product specs, ADRs, and operational runbooks live alongside the code in a Docusaurus site, reviewed in PRs like any other artifact.
- **CI/CD That Catches Problems Early** -- GitHub Actions pipelines with 8 specialized workflows covering linting, testing, SAST, DAST, container scanning, SonarQube analysis, and E2E verification.

## Architecture Overview

```
                        ┌──────────────────┐
                        │   OpenAPI 3.1    │
                        │    Contract      │
                        │    (api/)        │
                        └────────┬─────────┘
                                 │ generates code
                  ┌──────────────┼──────────────┐
                  ▼              │              ▼
         ┌──────────────┐       │     ┌──────────────┐
         │   Backend    │       │     │   Frontend   │
         │ Spring Boot 4│       │     │  React 19    │
         │   Java 25    │       │     │  TypeScript  │
         │  (backend/)  │       │     │  (frontend/) │
         └──────┬───────┘       │     └──────┬───────┘
                │               │            │
                │        ┌──────┴───────┐    │
                │        │   Gateway    │    │
                │        │ Spring Cloud │◄───┘ BFF proxies
                │        │  (gateway/)  │      API calls
                │        └──────┬───────┘
                │               │
         ┌──────┴───────┐  ┌───┴────────┐
         │ PostgreSQL   │  │  Keycloak   │
         │   18.x       │  │  26.x      │
         └──────────────┘  └────────────┘
```

**Request flow:** Browser -> Frontend (Nginx) -> Gateway (BFF, handles OAuth2 sessions) -> Backend (stateless Resource Server, validates JWT) -> PostgreSQL

## Repository Structure

```
template-app/
├── api/                    # OpenAPI 3.1 specification (the contract)
│   ├── specification/      #   openapi.yaml — single source of truth
│   ├── rules/              #   Custom Spectral linting rules (style + examples)
│   └── package.json        #   Lint, diff, validate, and mock scripts
├── backend/                # Spring Boot 4.0.1 application (Java 25)
│   ├── src/main/java/      #   Feature-based package structure (Screaming Architecture)
│   ├── src/test/java/      #   Unit, slice, integration, and architecture tests
│   └── pom.xml             #   Maven build with OpenAPI code generation
├── frontend/               # React 19 + TypeScript + Vite
│   ├── src/                #   Feature-sliced architecture
│   ├── e2e/                #   Playwright end-to-end tests
│   └── package.json        #   Dev, mock, test, and build scripts
├── gateway/                # Spring Cloud Gateway (BFF for OAuth2)
│   └── pom.xml             #   OAuth2 Client + session management
├── website/                # Docusaurus documentation portal
│   └── docs/               #   Arc42 architecture, product specs, runbooks
├── keycloak/               # Keycloak realm configuration
│   └── import/             #   Auto-imported realm JSON
├── docker-compose.yml      # Full local stack (6 services)
├── .env.template           # Environment variable reference
└── .github/workflows/      # 8 CI/CD pipeline definitions
```

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| **API Contract** | OpenAPI | 3.1 |
| **API Governance** | Spectral, openapi-diff | - |
| **API Mocking** | Prism | - |
| **Backend** | Spring Boot | 4.0.1 |
| **Language** | Java (preview features enabled) | 25 |
| **Modularity** | Spring Modulith (JPA event outbox) | 2.0.1 |
| **Security** | Spring Security + OAuth2 Resource Server | - |
| **OIDC Simplification** | Spring Addons | 9.1.0 |
| **Gateway / BFF** | Spring Cloud Gateway | - |
| **Identity Provider** | Keycloak | 26.x |
| **Database** | PostgreSQL | 18.x |
| **Migrations** | Flyway | - |
| **Audit Trail** | Hibernate Envers | - |
| **ID Strategy** | TSID + human-readable functional IDs | - |
| **DTO Mapping** | MapStruct | 1.6.3 |
| **Code Quality** | Checkstyle, PMD, SpotBugs (FindSecBugs), ArchUnit | - |
| **Backend Testing** | JUnit 5, Testcontainers, RestAssured, swagger-request-validator | - |
| **Frontend** | React | 19 |
| **Frontend Build** | Vite + SWC | - |
| **API Client Gen** | @hey-api/openapi-ts | - |
| **Frontend Testing** | Vitest, Playwright, MSW | - |
| **Documentation** | Docusaurus + Arc42 + Mermaid.js + Scalar (API Reference) | - |
| **CI/CD** | GitHub Actions (reusable workflows) | - |
| **SAST** | Semgrep (OWASP Top 10) | - |
| **Secrets Detection** | Gitleaks | - |
| **Container Scanning** | Trivy | - |
| **DAST** | OWASP ZAP | - |
| **Code Analysis** | SonarQube Cloud | - |
| **SBOM** | CycloneDX | - |
| **Observability** | Micrometer + OpenTelemetry | - |

## Prerequisites

- **Java** 25
- **Maven** 3.9+ (or use the included `mvnw` wrapper)
- **Node.js** 25
- **Docker** (for Testcontainers, Keycloak, and the local stack)

## Quick Start

### Option A: Full Stack with Docker Compose

The fastest way to run everything locally:

```bash
cp .env.template .env          # Configure environment variables
docker compose up -d           # Starts PostgreSQL, Keycloak, Backend, Gateway, Frontend, Docs
```

| Service | URL |
|---------|-----|
| Frontend | `http://localhost:${FE_PORT}` |
| Gateway (BFF) | `http://localhost:${GW_PORT}` |
| Backend API | `http://localhost:8080/api` |
| Keycloak | `http://localhost:${KC_PORT}` |
| Documentation | `http://localhost:3001` |

### Option B: Individual Services

#### Start the backend

```bash
cd backend
./mvnw spring-boot:run
```

Requires a running PostgreSQL instance. Configure connection in `application-local.properties`.

#### Start the frontend

**With backend running:**
```bash
cd frontend
npm install
npm run dev
```

**Without backend (mock mode):**
```bash
cd frontend
npm install
npm run dev:mock
```

Mock mode starts a Prism server that serves responses from the OpenAPI spec examples -- no backend or database needed.

#### Browse the documentation

```bash
cd website
npm install
npm start
```

Opens the Docusaurus documentation portal at `http://localhost:3000`.

## Module Guide

### `api/` -- The Contract

The OpenAPI 3.1 specification that drives both backend and frontend code generation. Defines endpoints for **Greetings** (full CRUD) and **User** (current session), with RFC 7807 error responses, pagination, and security schemes.

Custom Spectral rules enforce:
- `kebab-case` paths, `camelCase` properties and operationIds
- Mandatory response examples (powers the Prism mock server)
- `ProblemDetail` schema for all error responses
- `additionalProperties` explicitly set on all schemas

**Key commands:**
```bash
npm run lint        # Validate against Spectral governance rules
npm run diff        # Detect breaking changes vs. main branch
npm run mock        # Start Prism mock server on port 4010
npm run validate    # Structural validation via openapi-generator-cli
```

**Workflow:** Edit `openapi.yaml` -> Lint -> Propagate to backend (`mvn compile`) and frontend (`npm run api:generate`).

---

### `backend/` -- Spring Boot 4

A modular Spring Boot application structured by business domain using Screaming Architecture. Each domain module (`greeting`, `user`, `audit`) owns its controller, service, repository, mapper, and model layers.

**Highlights:**
- **Spring Modulith** enforces module boundaries and provides a transactional event outbox (JPA-backed)
- **API-First** -- Controllers implement interfaces generated from the OpenAPI spec by `openapi-generator-maven-plugin`
- **MapStruct** for compile-time, type-safe DTO-to-entity mapping
- **TSID** surrogate keys (DB-performant, loosely chronological) + human-readable functional IDs (e.g., `GRE-2025-000042`)
- **Hibernate Envers** for full entity audit history with custom revision metadata
- **OAuth2 Resource Server** with Spring Addons for simplified OIDC/Keycloak integration
- **Testcontainers** with real PostgreSQL and Keycloak, schema-per-test isolation for parallel execution
- **Contract validation** via `swagger-request-validator` ensures every test request/response matches the OpenAPI spec
- **ArchUnit** tests enforce layering rules and naming conventions
- **Observability** via Micrometer + OpenTelemetry bridge, structured logging, dedicated management port

**Key commands:**
```bash
./mvnw clean compile          # Regenerate API interfaces from the spec
./mvnw verify                 # Run all tests (unit + integration + contract)
./mvnw spring-boot:run        # Start the server on localhost:8080
./mvnw checkstyle:check       # Verify coding standards
./mvnw verify -Pquality-check # Run static analysis only (Checkstyle, PMD, SpotBugs)
```

---

### `gateway/` -- Spring Cloud Gateway (BFF)

The Backend-for-Frontend gateway handles OAuth2 login flows and session management, so tokens are never exposed to the browser. The frontend talks to the gateway, which proxies authenticated requests to the backend with JWT tokens.

- OAuth2 Client with Keycloak
- CSRF protection (cookie-based `XSRF-TOKEN`)
- Session-based authentication for the browser

---

### `frontend/` -- React 19 + TypeScript

A feature-sliced React application with auto-generated API client, dual development modes, and comprehensive testing at every level.

**Highlights:**
- **Generated API client** from OpenAPI via `@hey-api/openapi-ts` -- full type safety, zero manual fetch calls
- **BFF-aware auth** -- cookie-based sessions via the Gateway, automatic CSRF token handling, session expiry detection
- **Mock-first development** with Prism -- work on UI independently, no backend required
- **MSW (Mock Service Worker)** for network-level test mocking with an in-memory CRUD store
- **Playwright** for E2E tests against both real backend and Prism mock server
- **Feature-sliced architecture** -- `greetings` and `auth` modules with co-located components, hooks, and tests

**Key commands:**
```bash
npm run dev                # Dev server (requires backend on :8080)
npm run dev:mock           # Dev server with Prism mock (no backend needed)
npm run test               # Vitest in watch mode
npm run test:coverage      # Vitest with V8 coverage
npm run test:e2e:mock      # Playwright E2E against Prism
npm run api:generate       # Regenerate API client from spec
npm run lint               # ESLint
npm run typecheck          # TypeScript strict checking
```

See [`frontend/` README](frontend/) for full details including FAQ and troubleshooting.

---

### `website/` -- Documentation Portal

A Docusaurus-powered documentation site with interactive API reference (Scalar), Mermaid.js diagrams, and Kroki/PlantUML support.

**Documentation structure:**

| Section | Path | Purpose |
|---------|------|---------|
| Architecture | `docs/architecture/` | Arc42 sections 01-12: goals, constraints, context, building blocks, runtime views, deployment, ADRs, quality, risks, glossary |
| Product | `docs/product/` | PRDs organized by functional domain, with templates |
| Operations | `docs/operations/` | Runbooks, environment config, deployment procedures |
| Developer Guides | `docs/developer-guides/` | Tutorials, cheatsheets, and how-tos |
| API Reference | `/api-reference` route | Interactive API docs powered by Scalar |

**Key commands:**
```bash
npm start           # Local dev server on port 3000
npm run build       # Build static site (broken links = build failure)
npm run typecheck   # Validate TypeScript config
```

---

### `.github/workflows/` -- CI/CD Pipelines

The CI system uses a **monorepo-aware orchestrator** (`ci.yml`) that detects which modules changed and triggers only the relevant pipelines. Pipelines are chained with strict dependency ordering:

```
Changes Detection
    ├── API Governance ──────────────────────┐
    ├── Code Security (Semgrep + Gitleaks) ──┤
    ├── Frontend CI ─────────────────────────┤
    ├── Docs Build ──────────────────────────┤
    │                                        ▼
    │                              Backend CI (compile, test,
    │                              static analysis, SonarQube)
    │                                        │
    │                              Container Security (Trivy)
    │                                        │
    │                              DAST (OWASP ZAP)
    │                                        │
    └────────────────────────────► E2E Tests (Playwright)
```

| Workflow | What it does |
|----------|-------------|
| **`ci.yml`** | Orchestrator -- detects changes via `paths-filter`, conditionally triggers sub-workflows, cancels stale runs |
| **`api-governance.yml`** | Spectral linting, breaking change detection (blocks unless major version bumped), generated code drift detection, SonarQube scan of YAML/JSON |
| **`backend-ci.yml`** | 3-phase pipeline: compile + generate artifacts -> parallel quality gates (unit tests, static analysis, Trivy, SonarQube) -> integration tests with Testcontainers. Generates SBOM (CycloneDX) |
| **`frontend-ci.yml`** | Parallel jobs: ESLint + Prettier, TypeScript checking, Vitest with coverage, OSV vulnerability scan + license audit, SonarQube scan |
| **`code-security-scan.yml`** | Semgrep SAST (OWASP Top 10 rules) + Gitleaks secret detection. Results uploaded to GitHub Security tab |
| **`container-security-scan.yml`** | Trivy scans of backend Docker image and PostgreSQL image for CRITICAL/HIGH CVEs |
| **`dast-scan.yml`** | OWASP ZAP baseline scan against running backend. Spins up PostgreSQL + backend, then attacks the API |
| **`e2e-test.yml`** | Full-stack Playwright tests: PostgreSQL + Backend + Frontend build, runs E2E suite |

## Development Workflow

This project enforces an **API-first, test-driven** workflow:

```
 1. Write a PRD            website/docs/product/specs/
 2. Design the API         api/specification/openapi.yaml
 3. Lint the contract      cd api && npm run lint
 4. Generate backend code  cd backend && ./mvnw clean compile
 5. Write failing tests    backend/src/test/java/  (TDD)
 6. Implement              backend/src/main/java/
 7. Generate frontend SDK  cd frontend && npm run api:generate
 8. Build the UI           frontend/src/features/
 9. Run E2E tests          cd frontend && npm run test:e2e:mock
10. Update documentation   website/docs/
```

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| API-first | OpenAPI 3.1 + code generation | Eliminates contract drift between backend and frontend |
| Backend modularity | Spring Modulith + event outbox | Enforces bounded contexts without microservice overhead; reliable async events |
| ID strategy | TSID + functional IDs | DB-performant primary keys + human-readable business references (e.g., `GRE-2025-000042`) |
| Authentication | OAuth2/OIDC + BFF Gateway | Tokens stay server-side; browser uses only session cookies |
| Error model | RFC 7807 ProblemDetail | Standard error format with `traceId` correlation across all layers |
| Frontend mocking | Prism + MSW | Independent frontend development; contract-verified mock responses |
| Documentation | Arc42 in Docusaurus | Living docs reviewed in PRs; broken links fail the build |
| Diagrams | Mermaid.js + Kroki | Editable by any contributor; no proprietary tools |
| Testing | Testcontainers + schema isolation | Real database in tests; parallel-safe via per-test PostgreSQL schemas |

Full Architecture Decision Records: [`website/docs/architecture/09-architecture-decisions/`](website/docs/architecture/09-architecture-decisions/)

## Contributing

1. Read the module-specific guidelines before making changes:
   - [`api/AGENTS.md`](api/AGENTS.md) -- API contract standards and governance rules
   - [`backend/AGENTS.md`](backend/AGENTS.md) -- Spring Boot architecture, coding standards, and testing strategy
   - [`frontend/AGENTS.md`](frontend/AGENTS.md) -- React/TypeScript patterns and testing guidelines
   - [`website/AGENTS.md`](website/AGENTS.md) -- Documentation standards and Arc42 structure
2. Follow the [Development Workflow](#development-workflow) above
3. Ensure CI passes before requesting review
4. Documentation is not optional -- update relevant docs with your changes

## License

This project is licensed under the MIT License. See the [OpenAPI spec](api/specification/openapi.yaml) for license details.
