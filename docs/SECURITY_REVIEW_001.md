# Security Review 001 — Order & Inventory Design

Date: 2026-10-24
Status: **REVIEW COMPLETE — DESIGN CHANGES REQUESTED; NO IMPLEMENTATION APPROVAL**
Scope: `docs/TECH_SPEC_001.md`, `docs/ADR-001.md`. This is a security-perspective review of design documents, not a live execution of an independent Codex security subagent.

## Findings

| ID | Severity | Reference | Risk and evidence | Required decision/control |
|---|---|---|---|---|
| SEC-01 | High | TECH_SPEC P07 / API | Seeded admin/customer credentials without provisioning/secret policy can lead to exposed admin accounts, particularly if seed runs outside local tests | Seed only in explicit dev/test environments; separate fixture credentials from git; fail closed in production; never expose seeded admin by default |
| SEC-02 | High | TECH_SPEC P07, P08 | JWT role checked but ownership path has no explicit SQL-level filter contract | For reads, include user ownership in DB predicate or admin-authorized path; test BOLA/IDOR on detail and list; no client-controlled user_id trust |
| SEC-03 | High | TECH_SPEC P01-P02 | Raw idempotency key stored and arbitrary client strings permitted within range; query/log/retention abuse and replay ambiguity | Validate strict key format/max length, avoid logging raw keys, scope to user+operation, rate-limit and bound payload, do not reuse key once committed during retention |
| SEC-04 | Medium | TECH_SPEC P07 | Login rate limit mentioned but no operational limits, brute-force safeguards or JWT key lifecycle specified | Choose rate-limit policy, strong password hash parameters, key management/rotation and clock skew; do not put tokens in logs |
| SEC-05 | Medium | TECH_SPEC P08 | 503/409 differentiation and error details can leak internals or drive unsafe retries | Stable public error codes, generic messages; request ID in response; internal diagnostics only in logs; document which failures retry with same key |
| SEC-06 | Medium | TECH_SPEC P05 | Admin `PATCH /products/:id` can update stock/price during checkout; allowed fields/permission boundary undefined | Restrict DTO whitelist, validate stock/price bounds, authorize admin operation, audit changes in later milestone or record at minimum who/when |
| SEC-07 | Medium | TECH_SPEC API | No request-size/item-count/quantity limits; malicious large carts or long lock holds possible | Enforce max line items, quantities, request body size, transaction timeout, bounded retries, login and order rate limiting |
| SEC-08 | Medium | TECH_SPEC P07 | JWT authentication does not specify refresh/logout semantics; lack of explicit scope could accidentally imply revocation | Milestone 1: short-lived access token only, no refresh/logout; document limitations; deny tokens with bad audience/issuer/algorithm/expiry |

## Threat test cases (to implement later)
- SEC-T01 invalid/expired/malformed JWT, wrong issuer/audience/algorithm -> 401.
- SEC-T02 customer attempts POST/PATCH product -> 403.
- SEC-T03 user A tries GET order B by ID; list pagination/search manipulation must not reveal B.
- SEC-T04 oversized body/cart, invalid idempotency key, negative/fractional quantity -> 400 or rate/size error, without DB mutations.
- SEC-T05 repeated login attempts and order flood -> documented throttling, no user enumeration.
- SEC-T06 same key reused for different payload -> 409, no disclosure of prior order details to other users.
- SEC-T07 log inspection: no password/token/hash/raw key in application logs.
- SEC-T08 seeded admin inaccessible in production configuration.

## Security sign-off criteria
SEC-01, SEC-02, SEC-03 must have explicit accepted controls before implementation. Independent security verification of real code requires subsequent diff review and executable tests.

## Notes
No vulnerability is confirmed in running code: no application source or test execution was available. Findings are prospective specification gaps.
