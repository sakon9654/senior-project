# Architect Review 001 — Order & Inventory

Status: **CONDITIONAL APPROVAL FOR DESIGN DIRECTION ONLY — IMPLEMENTATION NOT APPROVED**
Review scope: `docs/ORDER_INVENTORY_KICKOFF.md`, `AGENTS.md`, `docs/WORKFLOW.md` on branch `setup/codex-ai-team`.
Date: 2026-10-24. No application code exists for runtime evaluation.

## Executive summary
Recommend **Option A: NestJS modular monolith + MySQL** as the first milestone. It is sufficient to explore concurrency, idempotency, authorization and transactional boundaries without distributed-system complexity. Do not add Redis/MongoDB/queues/microservices initially.

The proposal has good high-level requirements and acceptance criteria; implementation must wait for resolved decisions on data semantics, idempotency lifecycle, transaction ordering, auth, API errors and test environment. This is a design review based on documentation, not a runtime verification or executed Codex subagent.

## Alternatives and trade-offs

| Option | Pros | Cons | Decision |
|---|---|---|---|
| A: NestJS modular monolith + MySQL | Atomic transaction for stock/order/idempotency, simple deployment, straightforward debugging | Shared deployment, risks of cross-module coupling | **Recommended** |
| B: Separate Orders + Inventory services | Independent deployment/scaling and explicit service boundaries | Distributed consistency, messaging/compensation/retry, more operations | Defer until a measured or organizational need |

Modular modules: Auth, Catalog/Products, Orders, Inventory. Orders coordinates one transaction; Inventory provides transactional stock operations. Avoid cyclic dependencies and leaking ORM entities into public API.

## Review findings (proposed design risks, not observed bugs)
| ID | Severity | Risk | Design control | Verification |
|---|---|---|---|---|
| AR-01 | High | Oversell from concurrent orders | Atomic conditional decrement with affected-row check in one transaction; do not read then blindly update | Simultaneous last-item orders |
| AR-02 | High | Duplicate orders on retry/timeout | Unique (user_id, key), canonical request hash, persisted committed result, defined conflict semantics | Concurrent same-key retries |
| AR-03 | High | Deadlocks for overlapping multi-line orders | Normalize/aggregate duplicate lines, deterministic product-id update order, bounded retry on specific DB errors | Reverse-order competing orders |
| AR-04 | High | Unauthorized order access | Per-resource ownership check; explicit admin role checks | User A reads/edits User B |
| AR-05 | High | Partial stock decrement on failure | Same MySQL transaction for order, lines, stock, idempotency record | Inject failure before commit |
| AR-06 | Medium | Price changes during checkout | Decide snapshot semantics; select price inside transactional boundary and avoid client-provided price | Admin updates price concurrently |
| AR-07 | Medium | Invalid stock changes outside order workflow | Centralized stock invariants and authorization; guarded updates; nonnegative constraints where supported | Admin update while checkout |
| AR-08 | Medium | Ambiguous HTTP conflict/retry behavior | Stable problem/error codes; document in-progress, duplicate and key reuse behavior | Contract/E2E tests |
| AR-09 | Medium | Monetary precision/overflow | Integer minor units with checked ranges or DECIMAL; never JS floating point for authoritative totals | Large totals, decimal input |
| AR-10 | Medium | Unbounded login abuse and token misuse | Password hashing, short-lived tokens, strict JWT validation, role/resource checks, login throttling | Negative authentication tests |

## Recommended transaction contract
Assume simple immediate-order placement with no payment authorization or reservations.

1. Authenticate and validate lines (positive quantities, bounded line count, aggregate duplicate product IDs, canonical sorted input).
2. Bind idempotency to authenticated user + operation + key; hash canonical relevant payload (explicitly define hash fields).
3. Begin transaction; atomically claim unique idempotency record. Contending same-key requests either wait for commit/rollback or receive documented retriable response. Once committed, identical request returns stored result; differing hash returns 409. Do not equate a transient DB duplicate exception with a safely reusable response without reading committed state.
4. Obtain consistent price and active status in the transactional scope; define whether stock and price are locked together. Sort product IDs, then conditionally decrement each stock row (`active = 1 AND stock_quantity >= qty`), asserting exactly one affected row per product. On any failure roll back all changes.
5. Insert order and items with price snapshots and checked total (never trust client totals). Record committed response/order reference with idempotency claim. Commit transaction.
6. Handle deadlocks and lock timeouts with *bounded* whole-transaction retries and jitter; only if safe for the idempotency design. Emit standardized error codes if exhausted.

**Important:** the above is not a guarantee of identical HTTP response after network disconnect; specify which fields and status are persisted/reconstructed. Do not keep an open transaction while calling external services.

## Suggested API contracts (subject to human approval)
- `POST /auth/login` returns access token; decide registration/seeded demo accounts.
- `GET /products` only active products and server-side pagination.
- `POST /products`, `PATCH /products/:id`: admin role + strict DTO validation.
- `POST /orders`: authenticated, required `Idempotency-Key`, body `{ items: [{ productId, quantity }] }`; on success 201 with order ID / final total / lines. Same key+payload returns documented equivalent result; key+different payload yields 409; insufficient stock yields 409, invalid input 400/422 (choose one), unauthenticated 401, forbidden 403.
- `GET /orders/:id`: owner/admin only; non-owner response must not leak sensitive order information.
- `GET /orders`: filter ownership on server; pagination/authorization.
- Version/namespace approach, error schema and idempotency TTL/retention need decisions.

## Database proposal and invariants
- users: unique email, hashed password, role.
- products: unique SKU, money minor units with safe range, stock_quantity integer >=0, active flag.
- orders: user_id foreign key, status, exact total, timestamp.
- order_items: order_id/product_id FKs, positive qty, unit_price_minor snapshot, line total derived consistently.
- idempotency_records: unique scoped key, payload hash, committed order ID/result, created_at and retention status.
- Foreign keys and unique/check constraints where supported. Index order listing by (user_id, created_at, id). Benchmark index needs on actual queries.
- MySQL InnoDB is assumed; transaction and isolation behavior should be verified with chosen MySQL version and ORM. Do not assume MongoDB participates in MySQL atomic transaction.

## Test plan

### Unit
U01 reject zero/negative/fractional quantities, duplicate lines aggregation, invalid product IDs.
U02 canonical request hashing invariant to item order; distinguish changed quantity/product.
U03 compute total without float error; limits and overflow.
U04 consistent error mapping and authorization policy.

### API / E2E
E01 login good/bad, expired/malformed token -> 401.
E02 customer create/update products -> 403; admin can create/update.
E03 create order with one/multiple items -> 201, snapshots and correct total.
E04 cross-user `GET /orders/:id` blocked; list scoped to owner; admin permitted.
E05 same idempotency key+same body sequentially -> one order; expected replay response.
E06 same idempotency key+different body -> 409; no new order.
E07 nonexistent/inactive products, invalid payload, insufficient stock -> stable error contracts.

### DB integration (real MySQL, not mock)
I01 force failure after decrement before order insert -> all stock and idempotency/order changes rolled back.
I02 2+ concurrent orders for last item -> exactly one success, no negative stock.
I03 50-100 concurrent same-key requests -> one persisted order, no duplicate stock decrement (responses obey documented retry policy).
I04 overlapping products in opposite request order -> no persistent deadlock and no partial orders; verify bounded deadlock handling.
I05 admin product price/stock update concurrently with checkout -> consistent selected semantics.
I06 drop connection/timeout near commit -> retry with same key returns stable committed outcome without duplication.
I07 verify FK, unique and nonnegative check constraints at DB layer.

### Operational checks (later milestones)
O01 static analysis/lint/typecheck, dependency scanning, secret scanning in CI.
O02 basic load test with throughput, p95 latency, error rate and DB pool metrics.
O03 restart API and verify idempotency persists; observe transaction metrics and structured errors.
O04 migrations apply to fresh DB and can be safely reversed or restored.

For concurrency assertions always verify **database final state**, not only HTTP responses. Use isolated seeded databases and deterministic barriers where possible. Never run destructive tests against production.

## Decision gates before implementation
D01 API-only first vs Angular from milestone 1. **Recommend API-only first**, add Angular afterward.
D02 TypeORM vs Prisma vs query builder. **Recommend TypeORM if learner wants NestJS conventions**, but explicitly inspect transaction and update-return semantics with the selected version. Raw SQL for tricky atomic updates may be acceptable.
D03 Auth now vs later. **Recommend minimal auth now** because ownership tests depend on it (seeded users; no full registration flow).
D04 One item vs multiple items. **Recommend multiple items** to exercise ordering/rollback.
D05 Agree modular monolith + MySQL. **Recommend yes**.
D06 Idempotency key scope/retention: choose (user, operation, key), expiry policy, replay response behavior and concurrent pending behavior.
D07 Money representation and currency; **recommend THB minor units**, explicit integer bounds; confirm VAT/discount scope excluded in milestone 1.
D08 Product edits during checkout: define price snapshot/locking behavior, and stock adjustment rules.
D09 CI/testing tooling and MySQL version/ORM version before implementation.

## ADR-001 (proposed)
Context: small order/inventory training system with important atomicity and authorization invariants, unclear future load.
Decision proposed: NestJS modular monolith + MySQL InnoDB, deterministic transactional stock decrement and idempotency in the same database, REST contract, approval gates.
Consequences: fewer moving pieces and strong atomicity; coupling/shared deployment accepted for milestone 1. Reassess distributed components only after learning/testing indicates reason.
**Status: Proposed / requires human sign-off.**

## Validation status
- Document inspection: completed.
- Source-code review: not applicable (no app implementation).
- Test execution: not performed.
- Actual independent custom subagent run: not performed; this document is an architectural review prepared from repository documents.
