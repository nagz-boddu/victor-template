# Template App

A production-grade monorepo template demonstrating **API-first development**, **modern Java with Spring Boot 4**, **React 19 with TypeScript**, and **documentation-as-code** practices. Built to serve as a reference architecture for teams that value type safety, strict contracts, security, and software quality.

## Why This Template?

Most "starter" projects give you a basic CRUD app with no guidance on how to scale it. This template goes further:

- **Contract-First API Design** -- The OpenAPI specification is the single source of truth. Backend and frontend code are generated from it, eliminating drift between teams.
- **Security by Default** -- OAuth2/OIDC with the BFF (Backend-for-Frontend) pattern, Keycloak integration, and automated secret scanning in CI.
- **Modular Backend** -- Spring Modulith enforces bounded contexts at the package level, preventing the "big ball of mud" as the codebase grows.
- **Documentation That Ships** -- Arc42 architecture docs, product specs, and operational runbooks live alongside the code in a Docusaurus site, reviewed in PRs like any other artifact.
- **CI/CD That Catches Problems Early** -- GitHub Actions pipelines for linting, testing, security scanning (Semgrep, Gitleaks, SonarQube), and E2E verification.

## Architecture Overview

```
                  ┌──────────────┐
                  │   OpenAPI    │
                  │   Contract   │
                  │  (api/)      │
                  └──────┬───────┘
                         │ generates
              ┌──────────┴──────────┐
              ▼                     ▼
     ┌────────────────┐   ┌────────────────┐
     │    Backend      │   │   Frontend     │
     │  Spring Boot 4  │   │   React 19     │
     │  Java 25        │   │   TypeScript   │
     │  (backend/)     │   │   Vite         │
     └────────┬────────┘   │   (frontend/)  │
              │            └────────────────┘
              ▼
     ┌────────────────┐
     │  PostgreSQL     │
     │  + Keycloak     │
     └────────────────┘
```

## Repository Structure

```
template-app/
├── api/                    # OpenAPI 3.1 specification (the contract)
│   ├── specification/      #   openapi.yaml — single source of truth
│   ├── rules/              #   Custom Spectral linting rules
│   └── package.json        #   Lint, diff, and mock scripts
├── backend/                # Spring Boot 4 application (Java 25)
│   ├── src/main/java/      #   Feature-based package structure
│   ├── src/test/java/      #   Unit, slice, and integration tests
│   └── pom.xml             #   Maven build with code generation
├── frontend/               # React 19 + TypeScript + Vite
│   ├── src/                #   Feature-sliced architecture
│   ├── e2e/                #   Playwright end-to-end tests
│   └── package.json        #   Dev, mock, test, and build scripts
├── website/                # Docusaurus documentation portal
│   └── docs/               #   Arc42 architecture, product specs, runbooks
└── .github/workflows/      # CI/CD pipelines
```

## Tech Stack

| Layer                 | Technology                               | Version |
| --------------------- | ---------------------------------------- | ------- |
| **API Contract**      | OpenAPI                                  | 3.1     |
| **API Linting**       | Spectral                                 | -       |
| **API Mocking**       | Prism                                    | -       |
| **Backend**           | Spring Boot                              | 4.0.1   |
| **Language**          | Java                                     | 25      |
| **Modularity**        | Spring Modulith                          | 2.0.1   |
| **Security**          | Spring Security + OAuth2 + Spring Addons | 9.1.0   |
| **Database**          | PostgreSQL + Hibernate Envers            | -       |
| **ID Strategy**       | TSID (Time-Sorted Unique Identifiers)    | -       |
| **Mapping**           | MapStruct                                | -       |
| **Code Quality**      | Checkstyle, ArchUnit, SpotBugs           | -       |
| **Backend Testing**   | JUnit 5, Testcontainers, RestAssured     | -       |
| **Frontend**          | React                                    | 19      |
| **Build Tool**        | Vite                                     | -       |
| **API Client Gen**    | @hey-api/openapi-ts                      | -       |
| **Frontend Testing**  | Vitest, Playwright, MSW                  | -       |
| **Documentation**     | Docusaurus + Arc42                       | -       |
| **CI/CD**             | GitHub Actions                           | -       |
| **Security Scanning** | Semgrep, Gitleaks, SonarQube Cloud       | -       |

## Prerequisites

- **Java** 25
- **Maven** 3.9+ (or use the included `mvnw` wrapper)
- **Node.js** 25
- **Docker** (for Testcontainers and local infrastructure)

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/VictorHueni/template-app.git
cd template-app
```

### 2. Start the backend

```bash
cd backend
./mvnw spring-boot:run
```

This starts the Spring Boot server on `http://localhost:8080`. Requires a running PostgreSQL instance (see Docker Compose or configure `application-local.properties`).

### 3. Start the frontend

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

### 4. Browse the documentation

```bash
cd website
npm install
npm start
```

Opens the Docusaurus documentation portal at `http://localhost:3000`.

## Module Guide

### `api/` -- The Contract

The OpenAPI 3.1 specification that drives both backend and frontend code generation. Every endpoint, schema, and error response is defined here first.

**Key commands:**
```bash
npm run lint        # Validate against Spectral rules
npm run diff        # Detect breaking changes vs. main branch
npm run mock        # Start Prism mock server on port 4010
```

**Workflow:** Edit `openapi.yaml` -> Lint -> Propagate to backend (`mvn compile`) and frontend (`npm run api:generate`).

See [`api/` README](api/) for full details.

---

### `backend/` -- Spring Boot 4

A modular Spring Boot application structured by business domain (greeting, user, audit) with strict layer separation.

**Highlights:**
- **Spring Modulith** enforces module boundaries at compile time
- **API-First** -- Controllers implement interfaces generated from the OpenAPI spec
- **MapStruct** for type-safe DTO-to-entity mapping
- **TSID** identifiers + human-readable functional IDs (e.g., `GRE-2025-000042`)
- **Hibernate Envers** for entity audit history
- **OAuth2 Resource Server** with Spring Addons for simplified OIDC configuration
- **Testcontainers** with real PostgreSQL and Keycloak for integration tests
- **Contract validation** via `swagger-request-validator` in tests

**Key commands:**
```bash
./mvnw clean compile       # Regenerate API interfaces from the spec
./mvnw verify              # Run all tests (unit + integration + contract)
./mvnw spring-boot:run     # Start the server on localhost:8080
./mvnw checkstyle:check    # Verify coding standards
```

See [`backend/` README](backend/) for full details.

---

### `frontend/` -- React 19 + TypeScript

A feature-sliced React application with auto-generated API client, dual development modes, and comprehensive testing.

**Highlights:**
- **Generated API client** from OpenAPI via `@hey-api/openapi-ts` -- full type safety, no manual fetch calls
- **Mock-first development** with Prism -- work on UI without a running backend
- **MSW (Mock Service Worker)** for unit/integration test mocking
- **Playwright** for E2E tests against both real backend and mock server
- **CSS Modules** for scoped styling

**Key commands:**
```bash
npm run dev                # Dev server (requires backend on :8080)
npm run dev:mock           # Dev server with Prism mock (no backend)
npm run test               # Vitest in watch mode
npm run test:e2e:mock      # Playwright E2E against Prism
npm run api:generate       # Regenerate API client from spec
```

See [`frontend/` README](frontend/) for full details.

---

### `website/` -- Documentation Portal

A Docusaurus-powered documentation site following the **Arc42** template for architecture documentation.

**Documentation structure:**
- **Architecture** (`docs/architecture/`) -- Arc42 sections 01-12: context, building blocks, runtime views, deployment, ADRs, glossary
- **Product** (`docs/product/`) -- PRDs organized by functional domain, with templates for new specs
- **Operations** (`docs/operations/`) -- Runbooks, environment configuration, deployment procedures
- **Developer Guides** (`docs/developer-guides/`) -- Tutorials and how-tos for contributors

See [`website/` README](website/) for full details.

---

### `.github/workflows/` -- CI/CD Pipelines

| Workflow                      | Trigger   | What it does                                                |
| ----------------------------- | --------- | ----------------------------------------------------------- |
| `ci.yml`                      | Push / PR | Orchestrates the full pipeline                              |
| `api-governance.yml`          | Push / PR | Lints OpenAPI spec, checks for breaking changes             |
| `backend-ci.yml`              | Push / PR | Builds, tests (Testcontainers), Checkstyle, JaCoCo coverage |
| `frontend-ci.yml`             | Push / PR | Lint, typecheck, unit tests, build                          |
| `e2e-test.yml`                | Push / PR | Full-stack E2E tests with Playwright                        |
| `code-security-scan.yml`      | Push / PR | Semgrep SAST + Gitleaks secret detection                    |
| `container-security-scan.yml` | Push / PR | Container image vulnerability scanning                      |
| `dast-scan.yml`               | Push / PR | Dynamic Application Security Testing                        |

## Development Workflow

This project enforces an **API-first, test-driven** workflow:

```
1. Write a PRD           → website/docs/product/specs/
2. Design the API        → api/specification/openapi.yaml
3. Lint the contract     → cd api && npm run lint
4. Generate backend code → cd backend && ./mvnw clean compile
5. Write failing tests   → backend/src/test/java/
6. Implement             → backend/src/main/java/
7. Generate frontend SDK → cd frontend && npm run api:generate
8. Build the UI          → frontend/src/features/
9. Run E2E tests         → cd frontend && npm run test:e2e:mock
10. Update documentation → website/docs/
```

## Key Design Decisions

| Decision           | Choice                        | Rationale                                                       |
| ------------------ | ----------------------------- | --------------------------------------------------------------- |
| API-first          | OpenAPI 3.1 + code generation | Eliminates contract drift between backend and frontend          |
| Backend modularity | Spring Modulith               | Enforces bounded contexts without microservice overhead         |
| ID strategy        | TSID + functional IDs         | DB-performant primary keys + human-readable business references |
| Authentication     | OAuth2/OIDC via BFF pattern   | Tokens never exposed to the browser                             |
| Frontend mocking   | Prism + MSW                   | Independent frontend development without backend dependency     |
| Documentation      | Arc42 in Docusaurus           | Living docs reviewed in PRs, no stale wikis                     |
| Diagrams           | Mermaid.js only               | Editable by any contributor, no proprietary tools               |

Architecture Decision Records are maintained in [`website/docs/architecture/09-architecture-decisions/`](website/docs/architecture/09-architecture-decisions/).

## Contributing

1. Read the module-specific guidelines before making changes:
   - [`api/AGENTS.md`](api/AGENTS.md) -- API contract standards
   - [`backend/CODING_GUIDELINES.md`](backend/CODING_GUIDELINES.md) -- Java coding standards
   - [`frontend/AGENTS.md`](frontend/AGENTS.md) -- React/TypeScript patterns
2. Follow the [Development Workflow](#development-workflow) above
3. Ensure CI passes before requesting review
4. Documentation is not optional -- update relevant docs with your changes

## License

This project is licensed under the MIT License. See the [OpenAPI spec](api/specification/openapi.yaml) for license details.
