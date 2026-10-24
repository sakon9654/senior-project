# Learning roadmap: implementation-focused Senior -> Technical Owner

## Project scenario
Build an **Order & Inventory** application incrementally:
Angular frontend, NestJS API, MySQL transactional data. Add Redis/queues/MongoDB only to solve observed needs.

## Milestones
1. **Requirements and architecture:** inventory, orders, roles; draw module/data flow; write ADR-001.
2. **Database invariants:** prevent overselling and duplicate orders with concurrency tests; compare locking strategies.
3. **API auth:** JWT, permissions, refresh/logout trade-offs; test object-level authorization.
4. **Scale:** run API instances behind a load balancer; benchmark bottlenecks before adding cache.
5. **Async jobs:** PDF invoice task, retries, deduplication, failure recovery.
6. **Operations:** health checks, logging, CI, integration tests, incident simulation.

## Exercise for every milestone
Write original requirement and acceptance tests first. Ask architect to propose options, choose and justify a design, have developer implement, have independent reviewers try to falsify it, then log lessons learned.

## Decision record template
- Context/problem
- Constraints and invariants
- Options considered
- Decision and why
- Consequences and risks
- Validation plan / rollback path

## Definition of Done
All acceptance criteria demonstrated; relevant tests recorded with actual results; security and data integrity reviewed; human signs off. No claim of production readiness without validation.
