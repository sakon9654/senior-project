# Human-controlled multi-agent workflow

## Roles
- **Human Technical Owner:** sets the requirement, approves architecture, rejects unjustified complexity, approves merge.
- **Orchestrator (main Codex session):** coordinates; cannot approve on behalf of human.
- **Architect:** independent design review; no edits.
- **Developer:** code only after design approval.
- **Security / QA / Database reviewers:** independent reviews against original requirements and actual diff.

## Gates
**Gate A — Design approval:** produce brief requirements, assumptions, alternatives, risks, API/schema impact, and acceptance criteria. Stop before implementation.

**Gate B — Implementation:** developer changes feature branch only, tests and records results.

**Gate C — Independent review:** run relevant reviewers; reviewer must not merely summarize developer output. Record confirmed findings and unresolved questions.

**Gate D — Human merge:** do not merge/deploy automatically. Show changed files, failed/passed/not-run checks and risk acceptance.

## Sample Codex prompt
```text
Read AGENTS.md and docs/WORKFLOW.md.
Requirement: Create an order API that prevents duplicate orders and overselling.
First inspect the repository. Ask the architect subagent to propose design options and risks. Do NOT implement yet.
Provide acceptance criteria, proposed API/database changes, and explicit human approval gate.
After my approval, delegate scoped implementation to developer. Independently review the diff with security-reviewer, qa-reviewer and database-reviewer. Report evidence and unresolved risks. Never merge.
```

## Example review report
| Severity | File:line | Evidence | Proposed fix | Status |
|---|---|---|---|---|
| High | src/orders/service.ts:42 (illustrative only) | Two competing requests can decrement one remaining stock unit | Conditional atomic update + transaction | Example |

## Operational notes
- Agent TOML profiles are *instructions*, not an automatic review pipeline or access controls.
- Invocation/configuration of custom subagents depends on the installed Codex version. Inspect Codex documentation and local configuration if they do not load.
- Keep reviewers on read-only tasks; coordinate editing to avoid agent conflicts.
- Add CI checks and branch protections when the application exists. A human should inspect and approve changes.
