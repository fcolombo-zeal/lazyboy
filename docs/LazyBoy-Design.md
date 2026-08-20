# LazyBoy — Master Design Document

> An agentic bug-fixing cockpit that runs on your laptop, acts as an extension of *you*, and uses Claude Code (via the Claude Agent SDK) as its reasoning and code-editing engine.

**Owner:** Francesco Colombo (Zeal IT Consultants)
**Status:** Design — v1.1 (all 13 environment constraints answered and folded in)
**Repo:** `fcolombo-zeal/lazyboy`
**Last updated:** 2026-08-19

---

## 1. Product summary

LazyBoy turns an Azure DevOps bug into a reviewed, branch-ready fix with as little human typing as possible — while keeping you in control at every irreversible step.

The loop it automates:

```
ADO inbox → pick a bug → harvest context (fields, attachments, App Insights transaction)
   → resolve which repos are implicated → clone/refresh workspace
   → INVESTIGATE (read-only agent) → Investigation Report  ──► you approve ──► post to ADO
   → FIX (write agent on bug/<id>-<slug>) → Change Report   ──► you approve ──► post to ADO
   → commit → Pull Request into a branch you choose
```

### Design principles

| # | Principle | Consequence |
|---|-----------|-------------|
| P1 | **Local-first, zero-server** | Everything runs on your machine. No LazyBoy backend in the cloud. Your tokens never leave localhost. |
| P2 | **You are the identity** | The tool authenticates *as Francesco*, using SSO where possible and PAT where SSO isn't practical. No service principals, no shared secrets. |
| P3 | **Read before write** | Investigation runs with a hard read-only tool policy. Writing is a separate, separately-approved stage. |
| P4 | **Irreversible = approved** | Posting an ADO comment, pushing a branch, opening a PR — all require an explicit click. Enforced in code (`can_use_tool`), not by prompt. |
| P5 | **Everything is resumable** | Runs, agent sessions, and event streams are persisted. Close the browser, come back, keep going. |
| P6 | **Deterministic shell, agentic brain** | Anything that can be a plain API call or a `git` command *is* one. Claude is used for judgement, not for plumbing. |
| P7 | **Lightweight** | One `uvx lazyboy` command, one process, SQLite, no Docker, no Postgres, no queue broker. |

### Explicit non-goals (v1)

- Multi-user / team deployment, RBAC, shared history.
- CI integration or auto-merge.
- Non-ADO issue trackers (Jira), non-ADO git hosts (GitHub) — the provider layer is pluggable but only ADO ships in v1.
- Automatic root-cause fixes without human review.

---

## 2. Confirmed decisions

| Decision | Choice | Rationale |
|---|---|---|
| Agent engine | **Claude Agent SDK for Python** (`claude-agent-sdk`) | Same language as the backend → one process, one async loop. Full feature parity with the TS SDK: `can_use_tool`, hooks, in-process MCP tools, subagents, session resume. |
| Backend | **Python 3.12 + FastAPI + Uvicorn** | Async-native (matches the SDK), first-class Azure SDKs, trivial to package with `uv`. |
| Frontend | **React 19 + TypeScript + Vite + Tailwind + TanStack Query** | Static SPA, built once, served by FastAPI from `/`. No dev server needed in production. |
| Transport | REST for commands, **SSE** for agent event streams | SSE is one-way, reconnects natively, survives proxies. WebSockets are unnecessary complexity here. |
| Storage | **SQLite** (via SQLModel) + files on disk | Single-user, single-writer. Zero setup. Blobs (reports, diffs, attachments) live on disk, referenced by path. |
| Secrets | **OS keychain** via `keyring` + Azure CLI token cache | Nothing sensitive in SQLite or `.env`. |
| Azure/ADO auth | **Entra SSO (Azure CLI / device code) preferred, PAT fallback** | SSO respects MFA + Conditional Access. PAT is the escape hatch when SSO is blocked. |
| Git host | **Azure DevOps Repos** | Same org as work items; one credential covers both. |
| Repo resolution | **Curated catalog + heuristics**, ADO Code Search as fallback | Deterministic and fast for the 90% case; agentic search only when the catalog misses. |
| Autonomy | **Human-in-the-loop gates** | Four approval gates: start-fix, post-report, commit, create-PR. |
| ADO topology | **One org, one project** | Project-scoped calls throughout. No project disambiguation UI. Org-scope stays behind a config flag for later. |
| Target stacks | **.NET/C# back-end + JS/TS front-end, in separate repos** | Two symbol-extraction paths (CLR `parsedStack` and browser telemetry), two build toolchains, and multi-repo runs are normal for full-stack bugs. |
| Catalog seeding | **`lazyboy catalog scan` is mandatory, not optional** | No existing service catalog exists — the tool must bootstrap its own and learn from your corrections. |
| Verification depth | **Compile only — no test execution** | Tests need infrastructure that isn't available locally. Every Change Report is stamped `unverified`; CI is the real gate. |
| Claude access | **Claude subscription (`claude login`)** | No per-token billing, so budgets are expressed in turns and tokens, not dollars. |

---

## 3. System architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  Browser  http://127.0.0.1:7717                                      │
│  React SPA — Inbox · Bug Workspace · Report Review · Diff Review     │
└───────────────┬──────────────────────────────────┬───────────────────┘
                │ REST (commands)                  │ SSE (event stream)
┌───────────────▼──────────────────────────────────▼───────────────────┐
│  FastAPI (single Uvicorn process, asyncio)                           │
│                                                                       │
│  ┌── API layer ──────────────────────────────────────────────────┐   │
│  │ /api/auth  /api/workitems  /api/runs  /api/repos  /api/events  │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌── Orchestrator ───────────────────────────────────────────────┐   │
│  │ RunManager · RunStateMachine · GateKeeper · EventBus          │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌── Stages ──────────┐ ┌── Agent Runtime ─────────────────────┐     │
│  │ Harvest            │ │ ClaudeSDKClient (one per run)         │     │
│  │ Resolve            │ │ ├─ investigator profile (read-only)   │     │
│  │ Investigate ───────┼─┤ ├─ fixer profile (scoped write)       │     │
│  │ Fix ───────────────┼─┤ ├─ subagents: repo-scout, build-runner│     │
│  │ Publish            │ │ ├─ can_use_tool → GateKeeper          │     │
│  └────────────────────┘ │ └─ hooks → audit log                  │     │
│                          └───────────┬──────────────────────────┘     │
│                                      │ in-process MCP: "lazyboy"      │
│  ┌── Connectors ─────────────────────▼──────────────────────────┐    │
│  │ AdoClient · AppInsightsClient · GitWorkspace · CredentialVault│    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  ┌── Persistence ────────────────────────────────────────────────┐    │
│  │ SQLite (~/.lazyboy/lazyboy.db) · workspace/ · runs/ · repos.yaml│   │
│  └───────────────────────────────────────────────────────────────┘    │
└───────┬───────────────┬───────────────┬───────────────┬───────────────┘
        │               │               │               │
   Azure DevOps    Azure Monitor    OS Keychain     Claude API
   (REST 7.1)      (Log Analytics)  (keyring)       (via Agent SDK)
```

### 3.1 Why one process

The Agent SDK spawns and manages the `claude` CLI as a subprocess itself; you get a duplex JSON stream over stdio. Wrapping that in a second service buys nothing and costs you a serialization hop and a failure mode. FastAPI owns the asyncio loop; each run owns one `ClaudeSDKClient`; the SDK owns its subprocess.

### 3.2 Event flow (the thing that makes the UI feel alive)

```
Agent SDK message ──► EventNormalizer ──► RunEvent (typed, JSON) ──► ┬─► SQLite (append-only)
                                                                      └─► EventBus ─► SSE ─► React
```

Every `RunEvent` has a monotonic `seq`. The SSE endpoint accepts `?after=<seq>`, so a browser refresh or a dropped connection replays from the database and then continues live. This is the single mechanism behind P5 (resumability).

### 3.3 Directory layout on disk

```
~/.lazyboy/
├── lazyboy.db                  # SQLite: work items, runs, events, gates, catalog cache
├── config.yaml                 # org URL, App Insights resources, model prefs, defaults
├── repos.yaml                  # the curated repo catalog (Phase 4)
├── workspace/
│   └── <repo-name>/            # bare-ish clone, reused across runs
├── runs/
│   └── <run-id>/
│       ├── context.json        # normalized BugContext
│       ├── attachments/
│       ├── transaction.json    # App Insights end-to-end trace
│       ├── worktrees/<repo>/   # git worktree pinned to the run's base branch
│       ├── investigation.md    # the report you review
│       ├── findings.json       # structured output (schema-validated)
│       ├── change-report.md
│       ├── changes.patch
│       └── audit.jsonl         # every tool call the agent made
└── logs/
```

Using **git worktrees** off a shared clone per repo means the second run on a 4 GB monorepo costs seconds, not minutes.

### 3.4 Repo layout (the LazyBoy source tree)

```
lazyboy/
├── pyproject.toml              # uv-managed; entrypoint `lazyboy`
├── src/lazyboy/
│   ├── main.py                 # CLI: launch, open browser, serve SPA
│   ├── config.py
│   ├── db/                     # SQLModel models + migrations
│   ├── api/                    # FastAPI routers
│   ├── core/                   # RunManager, state machine, EventBus, GateKeeper
│   ├── connectors/
│   │   ├── credentials.py      # CredentialVault
│   │   ├── ado.py              # AdoClient
│   │   ├── appinsights.py      # AppInsightsClient
│   │   └── git.py              # GitWorkspace
│   ├── stages/                 # harvest, resolve, investigate, fix, publish
│   ├── agent/
│   │   ├── runtime.py          # ClaudeSDKClient lifecycle
│   │   ├── profiles.py         # investigator / fixer ClaudeAgentOptions
│   │   ├── tools.py            # in-process MCP server "lazyboy"
│   │   ├── gates.py            # can_use_tool implementation
│   │   ├── hooks.py            # PreToolUse/PostToolUse audit
│   │   └── prompts/            # system prompts + report templates
│   └── static/                 # built SPA (generated)
└── web/                        # Vite + React source
```

---

## 4. Domain model (summary)

Full schema in [`reference/data-model.md`](reference/data-model.md).

| Entity | Purpose |
|---|---|
| `WorkItem` | Cached ADO work item: id, title, state, type, assigned-to, url, tags, last-synced. |
| `Run` | One end-to-end pass over one work item. Owns the state machine. |
| `BugContext` | Normalized harvest output: description text, repro steps, attachments, App Insights transaction, extracted stack frames, candidate symbols. |
| `RepoCandidate` | A repo the resolver thinks is implicated, with `confidence`, `evidence[]`, `resolved_by` (catalog \| heuristic \| code-search \| manual). |
| `AgentSession` | Claude session id + profile, so a run can be resumed or forked. |
| `RunEvent` | Append-only, monotonic `seq`, typed payload — drives SSE and the audit trail. |
| `Gate` | A pending human decision: kind, payload, state (pending/approved/rejected), decided_at. |
| `Report` | Investigation or Change report: markdown body + structured `findings.json`. |

### 4.1 Run state machine

```
                   ┌──────────────► blocked_no_repo ──(you attach a repo)──┐
                   │                                                        │
created ─► harvesting ─► resolving ─► ready_to_investigate ◄────────────────┘
                                              │
                                    (gate: start investigation — auto by default)
                                              ▼
                                       investigating ─► report_ready
                                              │
                          ┌───────────────────┼───────────────────┐
                          │ (gate: post comment)                  │ (gate: start fix)
                          ▼                                       ▼
                     report_posted                            branch_prepared ─► fixing
                                                                                    │
                                                                                    ▼
                                                                              changes_ready
                                                                                    │
                                              ┌─────────────────────┬───────────────┤
                                              ▼                     ▼               ▼
                                        (gate: post)          (gate: commit)   (gate: PR)
                                        change_posted           committed      pr_created
                                                                                    │
                                                                                    ▼
                                                                                  done
```

Terminal/side states: `failed`, `cancelled`, `blocked_no_repo`.

`blocked_no_repo` is the state that satisfies your "mark the work item as *waiting for one or more repository reference*" requirement: LazyBoy sets the local state, and (behind an approval gate) adds the ADO tag `lazyboy:needs-repo` plus a comment listing what evidence it *did* find and what it needs from you.

---

## 5. Agent design

### 5.1 Two profiles, one SDK

| | **Investigator** | **Fixer** |
|---|---|---|
| Purpose | Understand and explain | Change code |
| `permission_mode` | `default` + deny-all-writes gate | `acceptEdits` inside the worktree only |
| Allowed built-ins | `Read`, `Grep`, `Glob`, `Bash` (allowlisted read-only), `Task`, `WebFetch` | + `Edit`, `Write`, `NotebookEdit`, `Bash` (**build-only** allowlist — no test runners) |
| MCP tools | full `lazyboy` server | full `lazyboy` server |
| `cwd` | `runs/<id>/worktrees/` | `runs/<id>/worktrees/` |
| Model | `claude-opus-*`, `effort: high` | `claude-sonnet-*` default, escalate on demand |
| Output | `investigation.md` + `output_format` JSON schema → `findings.json` | `change-report.md` + `changes.patch` |
| Subagents | `repo-scout`, `log-analyst` | `build-runner`, `reviewer` |

Both are built from `ClaudeAgentOptions` with `system_prompt = {"type": "preset", "preset": "claude_code", "append": <profile prompt>}` so you keep Claude Code's native tool discipline and add domain rules on top.

### 5.2 The `lazyboy` in-process MCP server

Rather than dumping a giant context blob into the prompt, the agent *pulls* what it needs. Created with `create_sdk_mcp_server(name="lazyboy", tools=[...])` and passed via `mcp_servers={"lazyboy": server}` — no subprocess, no network, direct access to the connectors.

| Tool | What it does |
|---|---|
| `ado_get_work_item` | Full fields, relations, comment thread for an id. |
| `ado_list_related` | Linked/child/duplicate work items. |
| `ado_search_code` | ADO Code Search across the org (the catalog-miss fallback). |
| `appi_get_transaction` | The already-harvested end-to-end transaction for this run. |
| `appi_query` | Ad-hoc KQL against the App Insights resource, read-only, row-capped. |
| `catalog_lookup` | assembly / `cloud_RoleName` / namespace → repo. |
| `repo_info` | Branches, recent commits, owners, build definition for a repo. |
| `git_log_search` | `git log -S` pickaxe across the worktrees. |
| `record_finding` | Structured finding emitted incrementally so the UI can render progress. |

Full contracts: [`reference/agent-contracts.md`](reference/agent-contracts.md).

### 5.3 Safety: `can_use_tool` is the real boundary

Prompts are guidance; the permission callback is enforcement. Every tool call routes through `GateKeeper.can_use_tool(tool_name, input_data, context)`:

1. **Phase policy** — investigator denies `Edit`/`Write`/`NotebookEdit` outright with a message telling the agent to report instead.
2. **Path jail** — any file path is resolved and must live under `runs/<id>/worktrees/`. Symlink escapes are caught by resolving to a real path first.
3. **Bash allowlist** — the command is parsed (`shlex` + a small grammar); only allowlisted binaries pass, and a denylist kills `git push`, `git commit`, `az`, `curl`, `npm publish`, `rm -rf /`, and anything with network egress in the fix phase. Everything else is denied with a reason.
4. **Human gate** — genuinely irreversible actions (post comment, push, open PR) are *not* agent tools at all. They're backend endpoints the UI calls after you click. The agent can only *propose* them.

`PreToolUse` / `PostToolUse` hooks write every decision to `runs/<id>/audit.jsonl` and emit a `RunEvent`, so the UI shows a live, greppable trace of what the agent touched.

### 5.4 Budget control

LazyBoy runs on a **Claude subscription**, not an API key, so there is no per-run dollar figure to cap. `max_budget_usd` is therefore **not used**. The real scarce resource is your rate-limit window, and the meaningful controls are:

- `max_turns` ceilings per stage (investigator ~40, fixer ~60) — the primary guard against a runaway loop.
- `task_budget` (`{"total": N}`) to bound subagent fan-out, which is where turns silently multiply.
- `effort` routing per stage: `high` for investigation, `medium` for mechanical fix work.
- A **turn + token meter** in the UI header, with cumulative per-work-item history, so you can see which bugs are expensive before you're rate-limited rather than after.

The runtime detects which credential is present (`claude` CLI session vs `ANTHROPIC_API_KEY`) and switches the meter between turns/tokens and dollars, so moving to an API key later is a config change, not a redesign.

---

## 6. Delivery phases

Each phase is independently shippable and leaves you with something usable. Detailed design docs:

| Phase | Title | Outcome | Doc |
|---|---|---|---|
| **0** | Foundation & Skeleton | `uvx lazyboy` opens a browser to a working (empty) app. | [phases/phase-0-foundation.md](phases/phase-0-foundation.md) |
| **1** | Identity & Credential Vault | Connect ADO + Azure via SSO or PAT; green health checks. | [phases/phase-1-identity.md](phases/phase-1-identity.md) |
| **2** | Work Item Inbox | See everything assigned to you, with state, description, link. | [phases/phase-2-workitem-inbox.md](phases/phase-2-workitem-inbox.md) |
| **3** | Context Harvester | Paste an id or URL → normalized `BugContext` incl. App Insights transaction and attachments. | [phases/phase-3-context-harvester.md](phases/phase-3-context-harvester.md) |
| **4** | Repo Resolution & Workspace | Implicated repos identified, cloned, worktreed — or `blocked_no_repo`. | [phases/phase-4-repo-resolution.md](phases/phase-4-repo-resolution.md) |
| **5** | Investigation Agent | Read-only agent produces the Investigation Report with live streaming. | [phases/phase-5-investigation.md](phases/phase-5-investigation.md) |
| **6** | Review & Publish to ADO | Approve a report → posts a formatted comment with links and images. | [phases/phase-6-review-publish.md](phases/phase-6-review-publish.md) |
| **7** | Fix Engine | `bug/<id>-<slug>` off a branch you choose; scoped edits; Change Report. | [phases/phase-7-fix-engine.md](phases/phase-7-fix-engine.md) |
| **8** | Commit & Pull Request | Approve → commit, push, open PR into a target branch you choose. | [phases/phase-8-commit-pr.md](phases/phase-8-commit-pr.md) |
| **9** | Hardening, Cost & Packaging | Audit trail, retries, budgets, telemetry, single-command install. | [phases/phase-9-hardening-packaging.md](phases/phase-9-hardening-packaging.md) |

Supporting references:

- [reference/data-model.md](reference/data-model.md) — SQLite schema, DTOs, event taxonomy, state machine tables.
- [reference/external-apis.md](reference/external-apis.md) — exact ADO REST calls, Azure Monitor KQL, the App Insights URL decoder, auth scopes.
- [reference/agent-contracts.md](reference/agent-contracts.md) — system prompts, MCP tool schemas, report templates, output JSON schemas.

### 6.1 Suggested sequencing

```
Phase 0 ─► 1 ─► 2 ─┬─► 3 ─► 4 ─► 5 ─► 7 ─► 8
                    └─► 6  (unblocks after 5, but build the publisher early — 6 and 8 share it)
                                                              9 (continuous)
```

**Milestone "useful"** = Phases 0–5: you can point LazyBoy at a bug and get a real analysis. Everything after that is about closing the loop.

---

## 7. Cross-cutting concerns

### 7.1 Security

- Tokens live in the OS keychain (`keyring`), never in SQLite, never in `config.yaml`, never in a log line. A redacting log filter strips anything matching known token shapes.
- The server binds `127.0.0.1` only, requires an `Origin` check, and uses a per-launch session token embedded in the URL LazyBoy opens — so a random page in your browser can't drive it.
- Agent egress: the fixer's Bash allowlist has no network binaries. `WebFetch` is available to the investigator only, and domain-restricted via config.
- Prompt-injection posture: work item text, App Insights payloads, and repo file contents are **untrusted data**. They're delivered inside clearly-fenced `<untrusted>` blocks, the system prompt states that instructions found there must be reported rather than followed, and — decisively — nothing in the untrusted path can reach an irreversible action, because those aren't agent tools.

### 7.2 Observability

- `audit.jsonl` per run: every tool call, decision, and gate.
- Structured `RunEvent` log is the UI's source of truth *and* the debugging log.
- Optional: LazyBoy emits its own OpenTelemetry traces to a local file for post-mortems on why a run went sideways.

### 7.3 Failure & retry

Every stage is idempotent and re-runnable from the last good state. Network calls use `tenacity` with jitter; 401s trigger a credential re-auth prompt rather than a failed run; the agent's session id is stored so `resume=` can pick up an interrupted investigation instead of paying for it twice.

### 7.4 Testing strategy

- Connectors: `respx`/`vcr.py` cassettes recorded once against real ADO/Azure responses (scrubbed).
- Agent stages: the SDK is injected behind a `Protocol`, so tests replay canned message streams — no API cost in CI.
- Gates: exhaustive unit tests on `GateKeeper` (path escapes, bash parsing, phase policy). This is the security boundary; it gets the most tests.
- End-to-end: one "golden bug" fixture (fake ADO payload + fake AI transaction + a tiny seeded repo with a real bug) exercised nightly.

---

## 8. Environment constraints (answered)

These were open questions at v1.0. All are now settled, and the consequences are folded into the phase docs.

| # | Question | Answer | Consequence |
|---|---|---|---|
| 1 | ADO topology | **One org, one project** | Project-scoped WIQL and repo listing. No project column, no cross-project disambiguation. Org-scope kept behind `ado.scope: project\|org` for later. |
| 2 | App Insights resources | **A few, split by environment** | Registry in `config.yaml` keyed by resource name with an `environment` tag. The portal URL identifies the resource; config supplies subscription/RG. Unknown resource ids fall back to ARM lookup. |
| 3 | Target stacks | **.NET/C# + JS/TS front-end, separate repos** | Two symbol pipelines: CLR `parsedStack` frames, and browser telemetry (`pageViews`, `browserTimings`, `exceptions` with JS stacks + source-map awareness). Catalog carries `stacks[]` per repo. Full-stack bugs → multi-repo runs → one PR per repo. |
| 4 | Local build & test | **Compiles, but tests need infra** | **Verification is compile-only.** `dotnet build` / `npm run build` are the fixer's proof. No test execution at all in v1. Every Change Report is stamped `unverified: tests not run locally` and names CI as the gate. |
| 5 | Existing repo catalog | **None — tribal knowledge** | `lazyboy catalog scan` is required Phase 4 scope, not a nice-to-have. It bootstraps from `.csproj`/`AssemblyName`/`package.json`, plus a `cloud_RoleName` discovery query, and learns from every manual correction. |
| 6 | Code Search extension | **Unknown — detect at runtime** | Probed at connect time. Present → primary bootstrap for an empty catalog. Absent → local grep over partial clones, parallelised. Both paths designed. |
| 7 | Repo count | **30–100** | Catalog scan runs as a background job with progress. Cloning everything up front is not viable; partial clones + LRU eviction matter. Resolver precision matters. |
| 8 | PR conventions | **Work item must be linked; branch policies respected** | `workItemRefs` + `AB#<id>` always. Policy configurations API read before PR creation so required reviewers and build validation are shown up front. Not draft by default; no house template. |
| 9 | Base branch | **Varies per repo** | Catalog stores per-repo `default_base_branch` and `default_pr_target`. The prompt pre-selects them but **always asks** — per your requirement. |
| 10 | Private package feed | **Yes — ADO Artifacts** | Phase 1 must configure a NuGet credential provider (and `.npmrc` if the front end uses the feed too) from the ADO token it already holds. **Compile-only verification depends entirely on restore working**, so Phase 1 gains a restore probe health check. |
| 11 | Front-end source maps | **Unknown — verify first** | Phase 3 gains an explicit pre-task: inspect one real browser exception. The resolver is designed to work *without* maps and treats them as an upgrade path. |
| 12 | Work item types | **Everything assigned to you** | No type filter in the WIQL. The inbox shows type badges and sorts Bugs first; non-Bug types are investigable but the fix stage is marked best-effort. |
| 13 | First-run catalog | **Background scan, app usable immediately** | The scan is a resumable background job with progress events. Bugs opened before it completes fall back to manual repo selection — which feeds the catalog anyway. |

### 8.1 What these answers changed

Four answers had real design consequences rather than just filling in config:

1. **No test execution** simplifies Phase 7 considerably — the `test-runner` subagent becomes a `build-runner`, the bash allowlist shrinks to build commands only, and the loop is plan → edit → compile → self-review. It also *weakens the product honestly*: LazyBoy proposes fixes that compile and read correctly, and says so plainly rather than implying they're verified.
2. **No existing catalog + 30–100 repos** makes Phase 4 the critical path. The scan command and the Code Search/grep bootstrap are now first-class deliverables, and the learning loop (every manual correction writes back to `repos.yaml`) is what makes the tool get better with use rather than staying at day-one accuracy.
3. **Subscription, not API key** removes dollar budgets entirely — see §5.4.
4. **Separate front-end and back-end repos** makes multi-repo runs the normal case for full-stack bugs, not an edge case. The run model, diff review, and PR creation all have to handle N repos gracefully from day one.

### 8.2 The one that can sink compile-only verification

Constraint 10 deserves calling out, because it interacts badly with constraint 4. Verification is compile-only, which means `dotnet build` *is* the entire safety net. If restore fails against the ADO Artifacts feed, that net has a hole in it and the fixer degrades to pure static reasoning without anyone noticing.

So the design treats restore as a **first-class, monitored capability**:

- Phase 1 configures the credential provider from the existing ADO token and adds a `restore` probe to `/api/auth/health` — a real `dotnet restore` against one catalog repo, run on demand, with the failure surfaced plainly.
- Phase 7 refuses to claim `compiled: true` unless restore succeeded in the same run. A restore failure sets `unverified_reason: "package restore failed"` and the Change Report says so at the top rather than burying it.
- The catalog records per-repo restore health from the scan, so you know which repos LazyBoy can actually verify before you start a run on one.

### 8.3 Still genuinely open

- **`cloud_RoleName` hygiene.** The scan can only infer role names from telemetry if they're set meaningfully. If several services share a role name, that matcher's weight has to drop. Answerable with one `union requests | summarize dcount(cloud_RoleInstance) by cloud_RoleName` query once credentials are wired — so Phase 4 should run it rather than guess.
- **Front-end source maps** (constraint 11) — resolved by inspection during Phase 3, not by decision now.

---

## 9. Effort estimate

Revised after the §8 answers.

| Phase | Effort | Change vs v1.0 |
|---|---|---|
| 0 Foundation | 0.5–1 day | — |
| 1 Identity | 1–1.5 days (SSO edge cases are where time goes) | — |
| 2 Inbox | 0.5 day | ↓ single project removes disambiguation work |
| 3 Harvester | 2–2.5 days | ↑ a second symbol pipeline for browser/JS telemetry |
| 4 Repo resolution | **3–3.5 days** | ↑↑ catalog scan is now mandatory, plus the grep fallback for 30–100 repos |
| 5 Investigation | 2 days | — |
| 6 Publish | 1–1.5 days | — |
| 7 Fix engine | **1.5 days** | ↓ compile-only verification removes the whole test-harness problem |
| 8 Commit & PR | 1–1.5 days | ↑ branch-policy reads + multi-repo PRs |
| 9 Hardening | 1–2 days for v1 polish | — |

**~14–16 focused days to a complete v1.** Phase 4 is now the long pole and the highest-risk piece; everything else is well-understood work.

**Milestone "useful"** (Phases 0–5) lands around day 9–10. If you want it sooner, the cheapest cut is to ship Phase 4 with catalog-scan + manual repo selection only, defer the scoring resolver to a second pass, and let yourself pick repos by hand for the first dozen bugs — which also generates the catalog corrections the resolver will learn from.
