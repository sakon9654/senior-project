# Independent-perspective Review Summary 001

Date: 2026-10-24
Status: **PENDING HUMAN DESIGN APPROVAL**. No code, migrations, merges or production operations authorized.

## Scope and method
Security and Database reviewer *perspectives* were separately evaluated against the original approved baseline and TECH_SPEC_001. Outputs: SECURITY_REVIEW_001 and DATABASE_REVIEW_001. **These documents do not prove that independently spawned Codex subagents executed.** No automated security scanning or database tests have been run.

## Key shared conclusions
1. Maintain API-only NestJS modular monolith + TypeORM + MySQL InnoDB + JWT + multi-item orders, as approved.
2. Idempotency claim and committed-result replay need precise contention and unknown-commit behavior; never repeat checkout with a new key on network timeout.
3. Persist stock/order/idempotency together; use deterministic product lock order for all multi-item writers and guarantee rollback before retry.
4. Seeded users and JWT require safe environment restrictions and per-resource authorization.
5. Safely represent satang and totals across MySQL BIGINT / JS BigInt / serialized decimal-string interfaces.
6. Treat per-line price snapshot as the proposed Milestone 1 consistency level; no whole-cart point-in-time guarantee.
7. Retain idempotency records indefinitely for Milestone 1; re-evaluate retention policy only with a future design.
8. Independently verify executable test outcomes once implementation is explicitly approved.

## Proposed amendments for human acceptance (not silently approved)
- **A01 Security:** allow seeded credentials *only* in test/dev; no default admin seeding in production; JWT access-token-only for milestone 1.
- **A02 Authorization:** check ownership in database queries, return 404 for non-owned details, apply list filters server-side.
- **A03 Resource safety:** bounded cart lines/quantities, request sizes and transaction lock times; structured errors and request IDs.
- **A04 Idempotency:** canonical normalized cart; uniqueness by (user, operation, key); DB-persisted committed response/reference; post-conflict lookup after rollback; never auto-generate replacement key after timeout.
- **A05 Financial correctness:** currency THB, integer satang with BigInt/string boundaries, reject overflow and invalid amounts.
- **A06 Concurrency:** product lock order ascending, conditional decrement, admin updates using compatible locking; retry only whole confirmed-rolled-back transactions.
- **A07 Price:** per-line locked snapshot; consistent within product, not necessarily a simultaneous whole-cart snapshot.
- **A08 Retention:** initial idempotency records have no automatic expiry; retention scaling is later ADR.
- **A09 Testing:** require real MySQL integration and final-state assertions plus security negative cases before human code approval.

## Approval remains separated
- Architecture baseline: approved.
- P01–P08 details and A01–A09 amendments: recommendations, not approved.
- Implementation: not approved.
- Merge: not approved.
