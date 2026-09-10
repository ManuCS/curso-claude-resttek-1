# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Resttek: a restaurant management platform (admin panel, staff app for kitchen/bar/floor, and a customer ordering app) on top of a Node.js API. Monorepo managed with npm workspaces (`packages/*`): one Express/TypeScript backend and three Angular 21 frontends plus a shared library.

## Commands

```bash
npm install              # installs all workspaces from the root (api, web-admin, web-clientes, web-empleados, web-shared)
npm run seed              # populates SQLite with test data (packages/api/src/scripts/seed.ts, safe to re-run — uses INSERT OR IGNORE)

npm run dev:api           # Express API with hot-reload (tsx watch) -> http://localhost:3000
npm run dev:admin         # Angular admin panel -> http://localhost:4200
npm run dev:empleados     # Angular staff app -> http://localhost:4201
npm run dev:clientes      # Angular customer app -> http://localhost:4202

npm test                  # runs the API test suite (vitest run), from the root
cd packages/api && npm run test:watch   # watch mode
cd packages/api && npx vitest run path/to/file.test.ts   # run a single test file
cd packages/api && npx vitest run -t "test name"          # run tests matching a name
```

Frontends are Angular CLI apps (`ng serve`, `ng build`) run independently; each proxies `/api` to `http://localhost:3000` via `proxy.conf.json`. There is no lint script configured in any package.

Test credentials after seeding: password equals the user's own email (e.g. `admin@resttek.com` / `admin@resttek.com`). See [README.md](./README.md) for the full credentials table and role-to-app mapping.

## Architecture

Full details live in `docs/arquitectura/` (general, API, frontend) and `docs/dominio/` (glossary, data model) — read those before making non-trivial changes. Key points that require cross-file understanding:

### Two coexisting backend styles (`packages/api/src`)

- **Hexagonal + DDD, only in `contexts/employee/`** (covers employees, auth, and clients — see below): `domain/` (entities, value objects, repository interfaces prefixed `I`), `application/` (use cases, one class per action with a single `execute()`), `infrastructure/` (SQLite repos, Bcrypt auth service, HTTP controllers/routes). `contexts/shared/` holds the `Email` value object and the Express middlewares/errorHandler — it's not a business bounded context.
- **Layered, everywhere else** (`restaurant`, `dish`, `ingredient`, `order`): flat `models/`, `repositories/`, `services/`, `controllers/`, `routes/` folders, one file per domain. No entities with behavior — models are plain interfaces; validation lives in services (`services/*.service.ts`) and in `normalizeX()` functions in `models/*.model.ts`.
- Dependency wiring happens inline in each route file (repo → service/use case → controller → router). The exception is `employee`, whose wiring is centralized in `contexts/employee/infrastructure/http/dependencies.ts` because its routes are split across three files.

### Clients are Employees

There is no client entity/table. A client is a row in `employees` with `role = 'cliente'` and `restaurant_id = null`. Login is one endpoint for everyone, and the response key is always `employee`, even after `/auth/register`.

### Orders expand quantities

Creating an order with `quantity: 3` for one dish produces 3 separate `order_items` rows with `quantity: 1` each, so kitchen/bar can track and update the status of each unit independently (`pendiente` → `preparando` → `listo` → `entregado`).

### Error handling has two paths

`AppError` subclasses (`errors/DomainErrors.ts`) map to HTTP codes by class name in the shared `errorHandler` (404 for `*NotFoundError` except `OrderNotFoundError`, 401 for `InvalidCredentialsError`, 400 otherwise). `OrderController` is the one controller that does **not** call `next(error)` — it catches and responds manually, so `OrderNotFoundError` yields 404 from `GET /orders/:id` but 400 from the item-status `PATCH`.

### Path aliases (API)

TypeScript path aliases (`@config/*`, `@shared/*`, `@employee/*`, `@models/*`, `@repositories/*`, `@services/*`, `@controllers/*`, `@routes/*`, `@scripts/*`, `@errors/*`) map to `src/*` and are resolved by `tsx` at runtime and `vite-tsconfig-paths` in tests. Imports use `.js` extensions (ESM, `nodenext`).

### Database

`config/database.ts` wraps `sqlite3` in a promise-based `Database` class, exported as a single pre-built instance `dbConfig`. Tables are created on `initialize()` via `CREATE TABLE IF NOT EXISTS`. DB file is `packages/api/resttek.db`, or `:memory:` when `NODE_ENV=test`.

### Frontends are feature-based but not uniform

- `web-admin` / `web-empleados`: each feature has `models/`, `pages/`, `services/`, `store/`. Data-backed features use a Store pattern — `@Injectable({ providedIn: 'root' })` with private writable signals exposed read-only, services return `Observable`s consumed via `firstValueFrom`. `web-empleados`'s `OrderStore` additionally polls every 30s (no websockets in the API).
- `web-clientes`: centralizes models/services in `core/` instead of per-feature; components call services directly with `.subscribe()` rather than going through a store (the only real store is the local, HTTP-free `CartStore`). Polling for order status is done with raw `setInterval` in the components (`my-orders`: 10s, `order-detail`: 5s).
- Shared code (`auth`, HTTP interceptors, login/register components, the `API_URL` injection token) lives in `@resttek/web-shared`, consumed directly from source (no build step) — restart the frontend dev server after editing it.
- `web-shared/src/lib/styles/base.css` is dead code: each app keeps its own copy of the design system in `src/styles.css` instead of importing it, so shared style changes must be replicated by hand across `web-admin`, `web-clientes`, and `web-empleados`.
- Role display names differ from stored values: the stored role is `manager`, shown as "Gerente" in the UI. In `web-empleados`, role-based menu filtering is navigation-only (computed signals in `ShellComponent`); the API's `authorize()` middleware is the actual authorization boundary.

### Auth

JWT-based (8h expiry, payload has `id`, `role`, `restaurantId`). Backend: `authenticate` middleware verifies the JWT, `authorize(roles)` checks role membership; both respond directly (401/403) without going through `errorHandler`. Frontend: `web-shared`'s `AuthInterceptor` attaches the bearer token, `ErrorInterceptor` clears the session and redirects to `/login` on 401, and `AuthGuard` blocks unauthenticated route access (session presence only, not roles).

### Tests

Vitest, unit tests colocated with source (`*.test.ts`), with hand-written doubles in `mocks/` folders per context/domain. There are no HTTP integration tests — `supertest` is a dev dependency but unused, so routes, middlewares, and `errorHandler` aren't covered by tests.
