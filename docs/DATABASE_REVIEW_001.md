# Database Review 001 — Order & Inventory Design

Date: 2026-10-24
Status: **REVIEW COMPLETE — DESIGN CHANGES REQUESTED; NO IMPLEMENTATION APPROVAL**
Scope: `docs/TECH_SPEC_001.md`, `docs/ADR-001.md`. Database-perspective document analysis; not an executed custom Codex database subagent and not a real MySQL test.

## Findings

| ID | Severity | Reference | Risk and evidence | Required clarification/control |
|---|---|---|---|---|
| DB-01 | High | TECH_SPEC P02 | Transaction-local unique idempotency claim may block; caught duplicate errors can poison current transaction/ORM state; unknown commit must not create second order | Perform conflicting committed-record lookup in separate safe transaction/connection after duplicate/deadlock rollback; same key on retry; define timeout/response behavior and replay consistency |
| DB-02 | High | TECH_SPEC P04-P05 | Checkout locks products in order; admin bulk updates may lock in different order, creating deadlocks; TypeORM locking/update semantics not verified | All multi-product writers lock IDs ascending, use QueryRunner on a single connection, verify row lock and `affected` semantics with real MySQL |
| DB-03 | High | TECH_SPEC P03 | BIGINT ↔ JavaScript number conversion may silently lose integer precision; quantity × price may overflow | Store units in BIGINT, parse/format as decimal strings or BigInt, enforce multiplication/sum bounds and output string fields |
| DB-04 | Medium | TECH_SPEC P06 | READ COMMITTED may surprise developers expecting repeatable reads; deadlock vs lock-wait timeout have different transaction outcomes | Explicit isolation/lock semantics, ensure rollback/dispose QueryRunner before full retry; integration tests for deadlocks, lock timeouts and connection drops |
| DB-05 | Medium | TECH_SPEC P04 | Concurrent price edits may lead to mixed-time cart snapshot across different products, even if each row is internally consistent | Decide whether per-line locked snapshot is acceptable or order requires catalog-wide point-in-time pricing; recommend per-line locked snapshot for milestone 1 |
| DB-06 | Medium | TECH_SPEC P02 | At-least-24h replay vs indefinite retention is contradictory unless explained as minimum guarantee; deletion allows later key reuse | Milestone 1: no expiry/cleanup (indefinite retention); document storage cost and revisit for production; do not promise key expiry |
| DB-07 | Medium | TECH_SPEC P05 | `products.stock_quantity` holds inventory but admin change may overwrite concurrent order decrement if read-modify-write | Admin stock adjustment must use transaction and atomic operation or lock; prevent direct unguarded stock assignment; audit later |
| DB-08 | Medium | TECH_SPEC Schema | Constraints/indexes and handling of product deletion not fully specified | InnoDB FK/unique CHECK, positive units; soft deactivate products, no destructive deletion after referenced by order; index orders(user_id, created_at, id) |
| DB-09 | Medium | TECH_SPEC P01 | Canonical payload hash depends on normalization; duplicate lines may change semantics or quantity overflow | Aggregate duplicate product lines before hashing and before stock checks; stable serialization, reject overflow and excess unique lines |
| DB-10 | Low | TECH_SPEC schema | No explicit schema migration reproducibility / deterministic seed | Use versioned migrations and isolated integration DB; avoid TypeORM synchronize in production |

## Recommended transaction pseudoflow
1. Validate/aggregate/cart bounds outside transaction; check authentication.
2. Begin MySQL transaction through one TypeORM QueryRunner; claim scoped idempotency unique key.
3. For each distinct product in ascending ID order, lock/select row and perform conditional decrement; confirm exactly one row affected. Snapshot price while lock held.
4. Insert order+items and response reference in same transaction; commit.
5. If duplicate scoped key, rollback failed transaction, then query committed record; compare hash, replay committed result or conflict. If peer still in flight/commit uncertain, return documented retriable response; never proceed with new key.
6. Retry **whole transaction** only on correctly classified transient deadlocks after confirmed rollback, with same idempotency key and bounded attempts. Never retry partial work.

## Database verification matrix (later)
- DB-T01 last item race across 2 and 100 parallel buyers; assert database rows and stock.
- DB-T02 100 concurrent same-key/same-payload requests, assert 1 order and one stock reduction.
- DB-T03 same key/different hash after commit -> no second order.
- DB-T04 failure injected between stock update and order insert -> full rollback.
- DB-T05 reverse multi-item carts + competing admin writes -> bounded retry, no deadlocked persistent transactions.
- DB-T06 price update during checkout -> documented per-line snapshot, correct totals.
- DB-T07 network failure before/after commit boundary -> retry same key yields zero/one logical committed order, never two.
- DB-T08 BIGINT > Number.MAX_SAFE_INTEGER and max quantity/total cases -> reject or calculate exactly.
- DB-T09 raw database constraints enforce nonnegative stock, uniqueness, FK behavior.
- DB-T10 migrations on fresh test DB, no implicit schema synchronization.

## Decision recommendations
- Accept P01/P02 only after defining committed-response retrieval and concurrent contention behavior.
- Accept P03 with satang as BigInt/string end-to-end.
- Accept P04 with per-line locked snapshot, not a simultaneous whole-cart price guarantee.
- Accept P05 with same lock ordering for admin bulk updates.
- Accept P06 with explicit rollback before retry and MySQL-based tests.

## Notes
No runtime bug asserted. Review is based solely on documentation; test cases are planned, not executed.
