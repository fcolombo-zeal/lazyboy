# LazyBoy — Design Documentation

Start here: **[LazyBoy-Design.md](LazyBoy-Design.md)** — the master design document. Everything else hangs off it.

## Phases

| # | Doc | What you get at the end of it |
|---|---|---|
| 0 | [Foundation & Skeleton](phases/phase-0-foundation.md) | `uvx lazyboy` opens a working (empty) local app |
| 1 | [Identity & Credential Vault](phases/phase-1-identity.md) | ADO + Azure connected via SSO or PAT, health checks green |
| 2 | [Work Item Inbox](phases/phase-2-workitem-inbox.md) | Everything assigned to you, with state and links |
| 3 | [Context Harvester](phases/phase-3-context-harvester.md) | A bug id → normalized context incl. the App Insights transaction |
| 4 | [Repo Resolution & Workspace](phases/phase-4-repo-resolution.md) | Implicated repos found, cloned, worktreed — or `blocked_no_repo` |
| 5 | [Investigation Agent](phases/phase-5-investigation.md) | A streaming, read-only investigation and a real report |
| 6 | [Review & Publish to ADO](phases/phase-6-review-publish.md) | One click posts a formatted comment with links and images |
| 7 | [Fix Engine](phases/phase-7-fix-engine.md) | `bug/<id>-<slug>` with reviewed, scoped changes |
| 8 | [Commit & Pull Request](phases/phase-8-commit-pr.md) | Commit, push, PR into a target branch you choose |
| 9 | [Hardening, Cost & Packaging](phases/phase-9-hardening-packaging.md) | Audit trail, budgets, one-command install |

## Reference

| Doc | Contents |
|---|---|
| [data-model.md](reference/data-model.md) | SQLite/SQLModel schema, enums, RunEvent taxonomy, state transition table, REST DTOs, migrations |
| [external-apis.md](reference/external-apis.md) | Exact ADO REST calls, auth scopes, the App Insights portal-URL decoder, the KQL |
| [agent-contracts.md](reference/agent-contracts.md) | `ClaudeAgentOptions` profiles, full system prompts, subagents, MCP tool schemas, output schemas, report templates |

## Reading order

- **Deciding whether to build it:** master doc §§1–5, then §8 (open questions).
- **Starting to build:** master doc → `data-model.md` → Phase 0 → Phase 1.
- **Understanding the agent:** master doc §5 → `agent-contracts.md` → Phase 5 → Phase 7.
- **Understanding the integrations:** `external-apis.md` → Phase 2 → Phase 3.
