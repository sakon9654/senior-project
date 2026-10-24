# Design proposal 001 — Order & Inventory (NOT APPROVED)

Status: **Proposal only — no application code implemented**.
Owner: Human Technical Owner.
Goal: practice Codex multi-agent technical planning, implementation review, and data consistency in a realistic small application.

## Problem
Manage a product catalog and stock, create orders without overselling, prevent accidental repeated creation, and inspect order history.

## Functional requirements (proposed)
1. Admin can create/update products (SKU, name, price, stock quantity, active status).
2. Authenticated customer can browse active products and place an order with one or more line items.
3. Order creation snapshots product prices and reduces available stock atomically.
4. Repeated requests using the same idempotency key and identical payload return the original order; same key with changed payload is rejected.
5. User can view their own orders; admin can inspect all orders.

## Non-functional requirements
- No overselling, even under simultaneous requests.
- Authorization checked per resource, never just a signed-in role.
- Validate inputs; monetary values stored as DECIMAL or integer minor units, not floating-point.
- Errors mapped to stable HTTP statuses.
- Tests reproduce duplicate and concurrent orders.
- Keep development runnable locally using Docker Compose when implementation is approved.

## Initial design options
### A (recommended for learning): NestJS modular monolith + MySQL
One API process with modules Auth, Products, Inventory, Orders. MySQL handles transactionally consistent order + stock state. Easy to reason about and test locally; limited independent scaling.

### B: split Orders and Inventory services
Separate deployable services with messaging and distributed consistency considerations. More operational complexity, cross-service atomicity unavailable without compensation; not justified by current load/requirements.

**Suggested decision:** A. No Redis, MongoDB, queue, or microservices in first milestone. Add only when justified by a measured problem.

## Preliminary API contract (subject to approval)
- `POST /auth/login` — authenticate
- `GET /products` — active products
- `POST /products` — admin only
- `PATCH /products/:id` — admin only
- `POST /orders` — authenticated; header `Idempotency-Key`
- `GET /orders/:id` — owner or admin
- `GET /orders` — authenticated, filter by ownership/role

## Proposed data model
- users (id, email unique, password_hash, role, created_at)
- products (id, sku unique, name, price_minor, stock_quantity, active, updated_at)
- orders (id, user_id, status, total_minor, created_at)
- order_items (id, order_id, product_id, qty, unit_price_minor)
- idempotency_records (id, user_id, key, payload_hash, order_id, created_at; unique(user_id, key))

Use foreign keys and explicit unique constraints. Final table types/indexes depend on the implementation libraries and MySQL version.

## Transaction sketch
1. Validate input and authenticated user.
2. Begin transaction; reserve idempotency key uniquely.
3. Conditional update for each product: `stock_quantity >= requested_qty`, atomic decrement; verify affected rows.
4. Create order and order_items with snapshot prices.
5. Persist idempotency result; commit.
6. On idempotency collision, compare payload hash and return prior result or conflict, with safe concurrent behavior.
Exact lock ordering, isolation, deadlock retries and failure cases must be finalized before code.

## Acceptance criteria
- Two concurrent orders for the last single item cannot both succeed.
- A retry of the same key+payload returns one order only.
- Same key with different payload returns HTTP 409.
- User A cannot read User B's order.
- Failed order creation does not partially decrement stock.
- Prices and totals use exact units with explicit rounding policy.

## Decisions needing your approval
1. Angular frontend in milestone 1, or API-only first?
2. ORM: TypeORM vs Prisma vs raw SQL/query builder?
3. Auth: local JWT for learning, or defer to milestone 2?
4. First milestone scope: one product per order vs multiple line items?
5. Confirm modular monolith + MySQL as baseline.

## Immediate next action
Have Architect analyze this proposal and the repository; produce an ADR and test plan for human approval. No implementation until approved.
