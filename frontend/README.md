# Frontend Application

React 19 + TypeScript + Vite application with OpenAPI-driven development and BFF authentication.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Development Modes](#development-modes)
- [Authentication](#authentication)
- [Testing](#testing)
- [API Client](#api-client)
- [Project Structure](#project-structure)
- [Scripts Reference](#scripts-reference)
- [FAQ](#faq)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Node.js 25+
- npm 10+

## Getting Started

Install dependencies:

```bash
npm install
```

Generate API client from OpenAPI specification:

```bash
npm run api:generate
```

## Development Modes

### Normal Development (with Backend)

Develop with a running backend server:

```bash
npm run dev
```

**Requirements**: Backend (or Gateway) must be running on `http://localhost:8080`

**How it works**:
- Vite proxies `/api/*` requests to the backend at port 8080
- Vite also proxies auth routes (`/login-options`, `/oauth2`, `/login`, `/logout`) to the Gateway
- Full CRUD operations with real database

**Use when**:
- Developing features that require backend logic
- Testing full-stack integration
- Working with real data and OAuth2 authentication

---

### Mock Development (without Backend)

Develop frontend independently using Prism mock server:

```bash
npm run dev:mock
```

**Requirements**: None! No backend needed.

**How it works**:
- Starts Prism mock server on **port 4010** (serves OpenAPI examples)
- Starts Vite dev server with hot reload
- Sets `VITE_AUTH_MODE=mock` to bypass OAuth2 and use a hardcoded mock user
- Sets `VITE_USE_PRISM=true` to proxy `/api` requests to Prism on port 4010
- Returns realistic data from OpenAPI specification

**Use when**:
- Backend is unavailable or not started
- Developing UI components independently
- Rapid prototyping without backend or Keycloak setup
- Backend developers testing frontend requirements

**Example response** (from OpenAPI examples):
```json
{
  "data": [
    {
      "id": 1001,
      "reference": "GRE-2025-000001",
      "message": "Hello, World!",
      "recipient": "World",
      "createdAt": "2025-01-15T10:30:00Z"
    }
  ],
  "meta": {
    "pageNumber": 0,
    "pageSize": 20,
    "totalElements": 3,
    "totalPages": 1
  }
}
```

---

### Mock Development with Real Auth

```bash
npm run dev:mock-real
```

Like `dev:mock` but with `VITE_AUTH_MODE=real` -- uses Prism for API data while still integrating with real OAuth2 flows via the Gateway. Used by Playwright mock E2E tests.

---

### Standalone Mock Server

Run only the mock API server (useful for backend developers):

```bash
npm run mock-api
```

**Access**: http://localhost:4010/v1/greetings

**Use when**:
- Backend developers need to test API contract compliance
- Testing API client libraries
- Generating sample requests/responses

---

## Authentication

The frontend uses the **BFF (Backend-for-Frontend) pattern** for authentication. Tokens are never stored in the browser.

### How it works

1. The frontend calls `GET /api/v1/me` to check the current session
2. If not authenticated, it redirects to `/login-options` (served by the Gateway)
3. The Gateway handles the OAuth2 Authorization Code Flow with Keycloak
4. After login, the session is maintained via an `HttpOnly` cookie
5. The API client automatically includes `credentials: "include"` for cookie-based auth

### CSRF Protection

The API client reads the `XSRF-TOKEN` cookie and attaches it as the `X-XSRF-TOKEN` header on all state-changing requests (POST, PUT, PATCH, DELETE).

### Session Expiry

On 401 responses, the API client dispatches a `auth:session-expired` custom event on `globalThis`, which the `AuthProvider` listens to for triggering re-authentication.

### Mock Auth Mode

In `dev:mock` mode (`VITE_AUTH_MODE=mock`), all authentication is bypassed with a hardcoded mock user. This allows full UI development without Keycloak or the Gateway.

---

## Testing

### Unit & Integration Tests

```bash
npm run test              # Run tests in watch mode
npm run test:coverage     # Run tests with V8 coverage report
```

**Test Stack**:
- Vitest (test runner)
- React Testing Library
- MSW (Mock Service Worker) for API mocking

**How API mocking works in tests**:
- MSW intercepts network requests at the application level
- Uses same request handlers for all tests (`src/test/mocks/handlers.ts`)
- Stateful in-memory store for CRUD testing with auth guards
- No need to run mock server during tests

---

### End-to-End Tests

#### Against Real Backend

```bash
npm run test:e2e
```

Starts a Vite preview server on port 4173 and runs Playwright against it. API calls can be mocked via `page.route()` in the test files.

#### Against Mock Server (Faster)

```bash
npm run test:e2e:mock
```

Uses `dev:mock-real` (Prism + real auth mode) as the webserver. Prism starts automatically.

#### CI Mode

```bash
npm run test:e2e:ci
```

Same as `test:e2e` but with `--reporter=line` for CI-friendly output.

**E2E test coverage**:
- `e2e/hello.spec.ts` -- Greetings CRUD flow and pagination
- `e2e/auth.spec.ts` -- Authentication states (anonymous vs. authenticated)
- `e2e/bff-integration.spec.ts` -- BFF patterns: cookie auth, CSRF, session expiry, OAuth2 login/logout

---

## API Client

### Code Generation

API client is auto-generated from OpenAPI specification:

```bash
npm run api:generate
```

**Source**: `../api/specification/openapi.yaml`
**Output**: `src/api/generated/`

**Generated files**:
- `sdk.gen.ts` - API functions (listGreetings, createGreeting, etc.)
- `types.gen.ts` - TypeScript interfaces
- `client.gen.ts` - HTTP client configuration

### Client Configuration (`src/api/config.ts`)

The API client is configured with:
- **Base URL** from `VITE_API_URL` env var (defaults to `/api`)
- **Credentials**: `include` for cookie-based BFF authentication
- **CSRF**: Automatically reads `XSRF-TOKEN` cookie and sends `X-XSRF-TOKEN` header
- **Session expiry**: Dispatches `auth:session-expired` event on 401 responses
- **Error handling**: RFC 7807 `ProblemDetail` parsing via `src/api/errors.ts`

### Usage Example

```typescript
import { listGreetings, createGreeting } from './api/generated';

// List greetings with pagination
const response = await listGreetings({
  query: { page: 0, size: 20 }
});

// Create a new greeting
const newGreeting = await createGreeting({
  body: {
    message: "Hello, World!",
    recipient: "World"
  }
});
```

### Type Safety

All requests and responses are fully typed:

```typescript
import type {
  GreetingResponse,
  CreateGreetingRequest,
  GreetingPage
} from './api/generated';
```

---

## Project Structure

```
frontend/
├── src/
│   ├── api/
│   │   ├── generated/           # Auto-generated API client (don't edit!)
│   │   ├── config.ts            # Client init, CSRF, session expiry, re-exports
│   │   ├── config.test.ts       # Client configuration tests
│   │   ├── errors.ts            # RFC 7807 ProblemDetail parsing
│   │   └── errors.test.ts       # Error handling tests
│   ├── features/
│   │   ├── greetings/           # Greeting feature module
│   │   │   ├── components/      #   React components + tests
│   │   │   └── hooks/           #   Custom React hooks + tests
│   │   └── auth/                # Authentication feature module
│   │       ├── AuthProvider.tsx  #   Context provider (session, login, logout)
│   │       ├── hooks.ts         #   useAuth() hook
│   │       ├── types.ts         #   AuthStatus, AuthMode, AuthContextValue
│   │       └── utils.ts         #   resolveAuthMode(), resolveLoginUri()
│   ├── test/
│   │   └── mocks/
│   │       ├── handlers.ts      # MSW request handlers (CRUD + auth guards)
│   │       ├── data.ts          # Mock data factories
│   │       └── server.ts        # MSW server setup
│   ├── App.tsx                  # Root component
│   └── main.tsx                 # Entry: initApiClient + AuthProvider
├── e2e/                         # Playwright E2E tests
│   ├── hello.spec.ts            #   Greetings CRUD flow
│   ├── auth.spec.ts             #   Authentication states
│   └── bff-integration.spec.ts  #   BFF patterns (CSRF, session, OAuth2)
├── playwright.config.ts         # E2E config (Vite preview, port 4173)
├── playwright.config.mock.ts    # E2E config (Prism, port 5173)
├── openapi-ts.config.ts         # API client generator config
├── vite.config.ts               # Vite bundler + proxy config
├── vitest.config.ts             # Vitest test runner config
├── Dockerfile                   # Multi-stage build (Node + nginx)
├── nginx.conf                   # Production nginx configuration
└── package.json                 # Dependencies & scripts
```

---

## Scripts Reference

| Script | Description |
|--------|-------------|
| `npm run dev` | Start development server (requires backend) |
| `npm run dev:mock` | Start development with Prism mock + mock auth (no backend) |
| `npm run dev:mock-real` | Start development with Prism mock + real OAuth2 auth |
| `npm run mock-api` | Start Prism mock server only (port 4010) |
| `npm run build` | Build for production |
| `npm run preview` | Preview production build (port 4173) |
| `npm run test` | Run unit tests in watch mode |
| `npm run test:coverage` | Run tests with V8 coverage |
| `npm run test:e2e` | Run E2E tests (Vite preview) |
| `npm run test:e2e:mock` | Run E2E tests against Prism |
| `npm run test:e2e:ci` | Run E2E tests with CI reporter |
| `npm run api:generate` | Generate API client from OpenAPI spec |
| `npm run lint` | Lint code with ESLint |
| `npm run lint:report` | Lint and output JSON report (for SonarQube) |
| `npm run typecheck` | Type-check with TypeScript |
| `npm run format` | Format code with Prettier |
| `npm run format:check` | Check formatting without writing |

---

## FAQ

### Q: Why do I get a 404 error when calling the API?

**A**: Check which mode you're using:
- **Real backend**: Ensure backend/Gateway is running on port 8080
- **Mock mode**: Use `npm run dev:mock` instead of `npm run dev`

### Q: Mock server returns different data than expected?

**A**: Mock data comes from OpenAPI examples. Update the specification:
- File: `../api/specification/openapi.yaml`
- Add/modify `examples` in response definitions
- Restart mock server

### Q: Can I add custom mock scenarios for tests?

**A**: Yes! Edit test mock handlers:
- File: `src/test/mocks/handlers.ts`
- Add custom logic for specific test scenarios
- Use `mockErrors` utilities for error responses

### Q: How do I switch between backend and mock mode?

**A**:
- Backend: `npm run dev` (default, proxies to backend on :8080)
- Mock: `npm run dev:mock` (sets `VITE_USE_PRISM=true` + `VITE_AUTH_MODE=mock`)
- No restart needed - just kill one and start the other

### Q: What's the difference between MSW (tests) and Prism (dev)?

**A**:
- **MSW**: In-process mocking for unit/integration tests (Node.js)
- **Prism**: Standalone HTTP server for development (separate process on port 4010)
- **Both**: Serve OpenAPI-compliant responses

---

## Troubleshooting

### Port 4010 already in use (Prism)

```bash
# Find process using port 4010
netstat -ano | findstr :4010    # Windows
lsof -i :4010                   # macOS/Linux
```

### API client not generated

```bash
# Ensure OpenAPI spec exists
ls ../api/specification/openapi.yaml

# Regenerate client
npm run api:generate

# Check for errors in openapi-ts.config.ts
```

### Vite proxy not working

1. Check Vite dev server is running (default port: 5173)
2. Verify proxy configuration in `vite.config.ts`
3. In mock mode, ensure Prism is running on port 4010
4. In real mode, ensure backend/Gateway is running on port 8080
5. Check browser DevTools Network tab for proxy errors

---

## Additional Resources

- [Vite Documentation](https://vite.dev/)
- [React Documentation](https://react.dev/)
- [Prism Documentation](https://stoplight.io/open-source/prism)
- [MSW Documentation](https://mswjs.io/)
- [Playwright Documentation](https://playwright.dev/)
- [hey-api/openapi-ts](https://github.com/hey-api/openapi-ts)
