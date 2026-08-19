# HIMSARU ERP

A full-featured ERP system for a small Indian trading/wholesale company — includes dashboard, product catalog, inventory, customers, and a complete purchase module.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — run the API server (port 8080, proxied at `/api`)
- `pnpm --filter @workspace/himsaru-erp run dev` — run the frontend (Vite, proxied at `/`)
- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from the OpenAPI spec
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- Required env: `DATABASE_URL` — Postgres connection string, `SESSION_SECRET` — session signing key

## Stack

- pnpm workspaces, Node.js 24, TypeScript 5.9
- API: Express 5 + express-session (cookie-based auth)
- DB: PostgreSQL + Drizzle ORM
- Validation: Zod (`zod/v4`), `drizzle-zod`
- API codegen: Orval (from OpenAPI spec)
- Build: esbuild (CJS bundle)
- Frontend: React + Vite + Tailwind CSS + shadcn/ui + wouter + TanStack Query

## Where things live

- `lib/db/src/schema.ts` — DB schema (users, suppliers, products, customers, purchases, purchase_returns, sales)
- `lib/api-spec/openapi.yaml` — OpenAPI contract (source of truth for API)
- `lib/api-client-react/src/generated/` — generated hooks and Zod schemas (do not edit)
- `artifacts/api-server/src/routes/` — Express route handlers
- `artifacts/himsaru-erp/src/pages/` — React page components
- `artifacts/himsaru-erp/src/components/layout.tsx` — sidebar layout with auth guard

## Architecture decisions

- Session-based auth (express-session + httpOnly cookie). No JWT.
- Password hashing: SHA256 + `himsaru_salt` (simple, no bcrypt dependency).
- Inventory auto-updated on purchase (increment), sale (decrement), purchase return (decrement). Uses `GREATEST(0, stock - qty)` to prevent negative stock.
- OpenAPI body schemas use entity-shaped names (`SupplierInput`, not `CreateSupplierBody`) to avoid Orval TS2308 collision.
- Express 5: all handlers typed `Promise<void>`, use `res.status().json(); return;` pattern.

## Product

- `/login` — login (admin@himsaru.com / admin123)
- `/dashboard` — stats: this month's sales/purchases, low stock count, recent activity
- `/products` — product catalog CRUD (name, SKU, category, unit, prices, stock levels)
- `/inventory` — read-only stock view, highlights items below minimum stock in amber
- `/customers` — customer master CRUD
- `/purchases/suppliers` — supplier master CRUD
- `/purchases/entry` — create new purchase with dynamic line items; auto-increments inventory
- `/purchases/register` — list all purchases
- `/purchases/returns` — create purchase return; auto-decrements inventory
- `/sales` — sales register (view)

## Seed data

- Admin user: admin@himsaru.com / admin123
- 3 suppliers: Sharma Enterprises, Gupta Trading Co., Himalayan Suppliers
- 8 products (Mustard Oil 1L is a low-stock item: 8 units, min 15)
- 3 customers: Verma General Store, Patel Supermarket, Singh Grocery

## User preferences

_Populate as you build — explicit user instructions worth remembering across sessions._

## Gotchas

- **Production session cookies require `app.set("trust proxy", 1)`** in `app.ts`. Replit's deployment proxy terminates HTTPS and forwards requests to Express over HTTP. Without this, `req.secure` is false, so `express-session` never attaches the `Secure` cookie to responses — login returns 200 but every subsequent request is 401.



- The `QueryClient` in App.tsx disables retries for 401/403 so unauthenticated users redirect to `/login` immediately without 3-retry delay.
- All `setLocation()` redirects must be inside `useEffect` — calling them during render throws React "setState during render" errors.
- NEVER include `setLocation` in `useEffect` dependency arrays — wouter's `setLocation` is not reference-stable across renders (especially after lockfile/dep changes), so including it in deps causes an infinite update loop. Only include the data values that gate the redirect (e.g. `[user]`, `[isLoading, error, user]`).
- `credentials: "include"` is set in `lib/api-client-react/src/custom-fetch.ts` — session cookies are sent automatically on every API call.
- Do NOT run `pnpm dev` at workspace root — use workflow restarts instead.

## Pointers

- See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details
