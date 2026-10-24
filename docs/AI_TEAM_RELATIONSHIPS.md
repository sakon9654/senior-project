# AI Team — relationships and approval flow

## Organization (Mermaid)
```mermaid
flowchart TD
  H["Human — Technical Owner<br/>requirements, design approval, merge approval"]
  O["Codex Orchestrator<br/>delegate, track, aggregate evidence"]
  A["Architect Agent<br/>design / alternatives / risks"]
  G1{"Gate 1: Human approves design?"}
  D["Developer Agent<br/>implementation + tests"]
  S["Security Reviewer<br/>independent evidence"]
  Q["QA Reviewer<br/>acceptance tests"]
  B["Database Reviewer<br/>consistency / transactions"]
  R["Review resolution<br/>triage → fix → re-review"]
  G2{"Gate 2: Human approves merge?"}
  P["Merge (human-authorized)"]
  H --> O --> A --> G1
  G1 -->|no| A
  G1 -->|yes| D
  D --> S
  D --> Q
  D --> B
  S --> R
  Q --> R
  B --> R
  R -->|findings to fix| D
  R -->|evidence complete| G2
  G2 -->|no| R
  G2 -->|yes| P
```

## Communication boundaries
- Human → Orchestrator: original requirements and approvals. Human retains decision power.
- Orchestrator → Architect: request design review based on inspected codebase.
- Architect → Human: options, assumptions, decision record, risks, acceptance criteria.
- Human → Developer (via Orchestrator): explicit approved scope, tests, constraints.
- Developer → Reviewers: code diff and test evidence; provide original requirement independently.
- Reviewers → Orchestrator: independently verified findings; never treat developer self-review as evidence.
- Orchestrator → Human: disagreement matrix, evidence, unverified items, recommendation only.
- Human → GitHub: explicit merge approval; agent approval is not sufficient.

## Escalation
Any architecture change, third-party dependency, unsafe migration, deployment, secret handling, or scope expansion stops for a human decision.

## Agent independence limitations
The model family can share blind spots. Require executable tests, CI, reproducible findings and human review. Profiles alone do not enforce file permissions, execution order, or automatic delegation.

## Practical team size
Small CRUD: architect + developer + QA.
Sensitive authentication: add security reviewer.
Transactional order changes: add database reviewer.
