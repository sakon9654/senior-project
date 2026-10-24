# Technical Specification 001 — Order & Inventory API

Status: **PROPOSAL — implementation not authorized**
Date: 2026-10-24
Approved baseline: API-only NestJS modular monolith, TypeORM, MySQL InnoDB, JWT seeded users, multi-item orders.

## 1. Scope and invariants

Milestone 1: login seeded users, list active products, admin create/update products, place multi-item orders, list/view own orders, admin order views. No frontend, payments, VAT, coupons, returns, shipments, email, Redis, MongoDB or queue.

I1: stock is nonnegative in every committed state; concurrent checkouts cannot oversell.
I2: one scoped idempotency key + canonical request maps to at most one committed order.
I3: order lines and totals reflect a consistent authoritative price snapshot; never use client-supplied price.
I4: failed transaction leaves no stock/order/idempotency partial commit.
I5: users cannot access others' orders without admin authorization.

## 2. Proposed defaults pending explicit human approval

**P01 — Idempotency key:** mandatory `Idempotency-Key` on `POST /orders`, UUID v4 or opaque string length 16–128 with safe character whitelist. Scoped by (user_id, operation='create-order', key), enforced by unique database constraint. Compute SHA-256 of canonical validated request (aggregate duplicate product IDs, sort by product ID, normalize quantities). Require same logical cart -> same hash independently of line ordering. **Do not hash sensitive token contents.**

**P02 — Idempotency lifecycle:** insert a claim inside the *same* InnoDB transaction as stock + order. Save payload hash, order reference, response status/body or deterministic reconstructable result on success. On conflict from concurrent matching request, wait on unique constraint as MySQL permits; after winning transaction commits, read committed record: identical hash -> replay 201 with same logical response and `Idempotency-Replayed: true`; differing hash -> 409 `IDEMPOTENCY_KEY_REUSED`. If the conflicting transaction rolls back, caller may claim and proceed. Bound statement/lock wait time; on unresolved in-flight/timeout return retriable 409/503 with `Retry-After` as appropriate. No blind second order creation. Store records for **at least 24h** (proposed); after expiration keys may be reused, which must be clearly documented. Do not delete records without a designed cleanup policy; for initial milestone retain records indefinitely, then implement safe pruning later. '24h' is a future minimum contractual guarantee, not an active TTL.

**P03 — Money:** currency THB, stored as integer **satang** (smallest minor unit), not JS floating-point authoritative totals. DB `BIGINT` unsigned or signed-positive with constraints as supported; map to string/BigInt-aware type boundary (do not cast unsafe BIGINT to JS Number). API price/amount values serialized as decimal integer strings representing satang, e.g. `"25900"` means THB 259.00. Require positive unit price, positive quantity, overflow/range checks; exclude VAT, discounts and FX from milestone 1.

**P04 — Price snapshot:** read authoritative product price and active status under transaction and per-product row locks; serialize concurrent admin price edits with checkout row locks. Snapshot each price into `order_items.unit_price_minor`. Total computed as sum of quantity × snapshot price, with checked bounds. Committed order may use price immediately before or after admin edit depending on lock acquisition order, never a mix between check and decrement for the same item.

**P05 — Inventory:** `products.stock_quantity` is current sellable stock for milestone 1. Quantity nonnegative integer with DB CHECK (supported MySQL version) and API validation. Admin stock adjustment is explicit authenticated update within transaction, with audit reason as future enhancement. No direct public stock updates. Checkout sorts unique product IDs ascending and uses a deterministic lock order. For each product, acquire row lock and issue conditional stock decrement `UPDATE products SET stock_quantity = stock_quantity - :qty WHERE id = :id AND active = 1 AND stock_quantity >= :qty`; require exactly 1 affected row; else roll back. Use one transaction encompassing all order lines and idempotency. This proposal explicitly avoids lost updates; do not rely solely on a preceding read.

**P06 — Retry/Isolation:** use InnoDB READ COMMITTED for transaction where configured and tested; enforce row-level locks and atomic conditional update rather than assuming isolation level alone is enough. Retry only the **whole transaction** on recognized deadlock/serialization-type transient errors with bounded attempts (proposed max 3) and randomized backoff. Never retry arbitrary database errors. Disconnect/unknown-commit retries must reuse the same idempotency key. Lock wait timeout handled as retriable error only after guaranteed transaction rollback.

**P07 — Auth and access:** seeded administrator and customer accounts; bcrypt/argon2 hashed passwords, JWT verified with algorithm/issuer/audience/expiry checks as configured, no secrets checked into repo. Roles admin/customer, but `GET /orders/:id` must verify row ownership or admin role; server-side filter for `GET /orders`. JWT tokens are not proof of resource-level authorization.

**P08 — Error/status contract:** success create = 201; replay = 201 with same order id and `Idempotency-Replayed` header. Invalid data=400, unauthenticated=401, insufficient permissions=403, unavailable/not-owned order=404 (avoid enumeration), stock conflict=409 `INSUFFICIENT_STOCK`, key-reuse mismatch=409 `IDEMPOTENCY_KEY_REUSED`, lock contention unresolved=503 `TEMPORARILY_UNAVAILABLE` with Retry-After. Use stable JSON `{code,message,details?,requestId}`. No raw SQL errors to clients. Rate-limit login, not only order creation.

## 3. Example API (PROPOSED, not implemented)

- `POST /auth/login` body `{email,password}`.
- `GET /products?page=1&limit=20`: active catalog.
- `POST /products`, `PATCH /products/:id`: admin.
- `POST /orders` header `Idempotency-Key: <opaque key>` body:
```json
{"items":[{"productId":"101","quantity":2},{"productId":"202","quantity":1}]}
```
- `GET /orders/:id`, `GET /orders?page=1&limit=20`: owner/admin.
- Return price strings in **satang**, not THB decimal. e.g. `{"orderId":"501","totalMinor":"51800","currency":"THB"}`.
- UUID vs numeric database IDs remain to be selected; string IDs at API boundary avoid precision loss.

## 4. MySQL data model (PROPOSED)
- `users(id, email UNIQUE, password_hash, role, created_at)`
- `products(id, sku UNIQUE, name, price_minor BIGINT, stock_quantity INT, active, updated_at)`
- `orders(id, user_id FK, status, total_minor BIGINT, created_at)`
- `order_items(id, order_id FK, product_id FK, quantity INT, unit_price_minor BIGINT)`
- `idempotency_records(id, user_id FK, operation, key, payload_hash, order_id FK nullable in active transaction, response_status, response_json, created_at, UNIQUE(user_id,operation,key))`
- Use DB FK/UNIQUE/CHECK as supported; indexed `orders(user_id,created_at,id)`; evaluate other indexes against queries.

## 5. Concurrency timeline examples

### C1: last product stock=1; 2 buyers
T1 locks product row, conditionally decrements to 0; T2 waits; after T1 commits T2's condition fails; T2 rolls back. Result = 1 order, stock=0.

### C2: same user+key; 2 requests
T1 claims unique key in transaction; T2 conflicts/waits; T1 commits; T2 sees matching committed request hash and returns original order without decrement. Different payload -> 409.

### C3: cart has products 1 and 2; competing reverse order
Normalize duplicates, sort IDs, update/lock 1 then 2 for both transactions; deadlock likelihood reduced, not eliminated. Bounded full retry on recognized errors.

### C4: concurrent admin price update
Admin update takes same product row lock; checkout snapshot and decrement use locked current values, preventing inconsistent per-item check vs update. Outcome depends on lock order.

### C5: connection lost near commit
Client retries using same idempotency key. If prior commit succeeded, replay order; if rolled back, proceed. Never generate a new key on network retry.

## 6. Test plan (not executed)

**Unit:** reject invalid quantities/ids/keys; canonical hash invariant to cart ordering and duplicate line representation; checked monetary math and limits; mapping stable error codes.

**MySQL integration:** same-transaction rollback after injected failure; last-item concurrency (2 and 100 request variants); same-key 100 concurrent requests; key+different body conflict; deadlock/retry bounded; concurrent admin stock/price update; disconnect near commit; FK/UNIQUE/CHECK assertions.

**API E2E:** JWT valid/expired/bad credentials; admin-only product mutation; ownership lookup/list isolation; multiple-line totals/price snapshot; pagination and invalid input; correct 201/400/401/403/404/409/503 contract.

**Operational (later):** dockerized database test environment, migrations, lint/typecheck, secret scan, load p95, API restart idempotency persistence. All concurrency tests must assert database final state, not only HTTP statuses.

## 7. Approval checklist
- [x] Modular monolith + MySQL InnoDB (human)
- [x] API-only milestone 1 (human)
- [x] TypeORM (human)
- [x] JWT seeded users (human)
- [x] Multi-line orders (human)
- [ ] P01–P08 defaults (pending human approval)
- [ ] ID strategy, MySQL/Node/Nest versions and reproducible toolchain selected
- [ ] Explicit permission to implement (not yet granted)
- [ ] Separate permission to merge (not yet granted)

No application source code, migration, CI workflow or deployment changes are authorized by this specification.
