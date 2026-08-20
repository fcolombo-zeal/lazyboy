# Reference — Data Model

Companion to [`../LazyBoy-Design.md`](../LazyBoy-Design.md). The SQLite schema (SQLModel), the typed JSON payloads, the RunEvent taxonomy, the state transition table, the REST DTOs, migrations, and retention.

Conventions used throughout:

- All timestamps are **UTC, timezone-aware** (`datetime(timezone.utc)`), stored as ISO-8601 text by SQLite.
- Ids: `Run.id`, `Gate.id`, `Report.id`, `AgentSession.id` are **UUIDv7** strings (`uuid6.uuid7()` → hex-dashed) — monotonic, so `ORDER BY id` is chronological and directory names sort naturally on disk.
- Anything the agent produces that has structure is stored **twice**: as a file on disk (human-readable, diffable) and as a JSON column (queryable). The DB row is authoritative for the API; the file is authoritative for the human.
- Nothing secret is ever stored in SQLite. `CredentialRef` stores *where* a secret lives, never the secret.
- Foreign keys are enabled (`PRAGMA foreign_keys=ON`), journal mode is WAL, `synchronous=NORMAL`.

---

## 1. Enums

```python
# src/lazyboy/db/enums.py
from enum import StrEnum


class RunState(StrEnum):
    """Exactly the states in the master doc's state machine (§4.1)."""
    CREATED               = "created"
    HARVESTING            = "harvesting"
    RESOLVING             = "resolving"
    BLOCKED_NO_REPO       = "blocked_no_repo"
    READY_TO_INVESTIGATE  = "ready_to_investigate"
    INVESTIGATING         = "investigating"
    REPORT_READY          = "report_ready"
    REPORT_POSTED         = "report_posted"
    BRANCH_PREPARED       = "branch_prepared"
    FIXING                = "fixing"
    CHANGES_READY         = "changes_ready"
    CHANGE_POSTED         = "change_posted"
    COMMITTED             = "committed"
    PR_CREATED            = "pr_created"
    DONE                  = "done"
    FAILED                = "failed"
    CANCELLED             = "cancelled"


TERMINAL_STATES  = {RunState.DONE, RunState.FAILED, RunState.CANCELLED}
ACTIVE_STATES    = {RunState.HARVESTING, RunState.RESOLVING,
                    RunState.INVESTIGATING, RunState.FIXING}
RESUMABLE_STATES = ACTIVE_STATES | {RunState.READY_TO_INVESTIGATE,
                                    RunState.REPORT_READY,
                                    RunState.BRANCH_PREPARED,
                                    RunState.CHANGES_READY}


class GateKind(StrEnum):
    START_INVESTIGATION = "start_investigation"   # auto-approved by default (config)
    POST_REPORT         = "post_report"           # ADO comment with investigation.md
    START_FIX           = "start_fix"             # creates bug/<id>-<slug>, arms the fixer
    POST_CHANGE_REPORT  = "post_change_report"    # ADO comment with change-report.md
    COMMIT              = "commit"                # git commit inside the worktree
    CREATE_PR           = "create_pr"             # push + open PR
    MARK_NEEDS_REPO     = "mark_needs_repo"       # tag lazyboy:needs-repo + comment
    TOOL_PERMISSION     = "tool_permission"       # ad-hoc can_use_tool escalation


class RunEventType(StrEnum):
    RUN_STATE_CHANGED      = "run.state_changed"
    AGENT_TEXT             = "agent.text"
    AGENT_THINKING         = "agent.thinking"
    AGENT_TOOL_USE         = "agent.tool_use"
    AGENT_TOOL_RESULT      = "agent.tool_result"
    AGENT_PERMISSION_DENIED= "agent.permission_denied"
    AGENT_FINDING          = "agent.finding"
    STAGE_STARTED          = "stage.started"
    STAGE_COMPLETED        = "stage.completed"
    GATE_OPENED            = "gate.opened"
    GATE_DECIDED           = "gate.decided"
    COST_UPDATED           = "cost.updated"   # name kept; payload is turns/tokens (§3.2)
    ERROR                  = "error"


class ResolvedBy(StrEnum):
    CATALOG     = "catalog"        # repos.yaml exact match
    HEURISTIC   = "heuristic"      # namespace/assembly/role-name similarity
    CODE_SEARCH = "code-search"    # ADO almsearch fallback
    MANUAL      = "manual"         # you attached it in the UI
    PRIOR_RUN   = "prior_run"      # same work item or same problemId resolved before


class ReportKind(StrEnum):
    INVESTIGATION = "investigation"
    CHANGE        = "change"


class GateState(StrEnum):
    PENDING  = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"
    EXPIRED  = "expired"    # run cancelled/failed while gate open


class AgentProfile(StrEnum):
    INVESTIGATOR = "investigator"
    FIXER        = "fixer"


class Stage(StrEnum):
    HARVEST     = "harvest"
    RESOLVE     = "resolve"
    INVESTIGATE = "investigate"
    FIX         = "fix"
    PUBLISH     = "publish"


class CredentialKind(StrEnum):
    ADO_PAT       = "ado_pat"
    ENTRA_SSO     = "entra_sso"      # no material stored; the token cache is the vault
    ANTHROPIC_KEY = "anthropic_api_key"
    CLAUDE_OAUTH  = "claude_oauth"   # `claude login` subscription
```

---

## 2. SQLModel tables

```python
# src/lazyboy/db/models.py
from __future__ import annotations
from datetime import datetime, timezone
from typing import Any, Optional

from sqlalchemy import Column, Index, UniqueConstraint, text
from sqlalchemy.dialects.sqlite import JSON
from sqlmodel import Field, SQLModel

from .enums import (AgentProfile, CredentialKind, GateKind, GateState,
                    ReportKind, ResolvedBy, RunEventType, RunState, Stage)


def utcnow() -> datetime:
    return datetime.now(timezone.utc)
```

### 2.1 `WorkItem` — cache of the ADO item

```python
class WorkItem(SQLModel, table=True):
    __tablename__ = "work_item"

    id:            int      = Field(primary_key=True)          # ADO work item id, natural key
    org:           str
    project:       str
    project_id:    str | None = None
    title:         str
    state:         str                                          # ADO state, verbatim ("Active")
    work_item_type:str                                          # "Bug", "Task", ...
    assigned_to:   str | None = None                            # displayName <upn>
    assigned_to_id:str | None = None                            # identity guid
    tags:          list[str] = Field(default_factory=list, sa_column=Column(JSON))
    priority:      int | None = None
    severity:      str | None = None
    area_path:     str | None = None
    iteration_path:str | None = None
    url:           str                                          # human link, _workitems/edit/{id}
    api_url:       str                                          # _apis/wit/workitems/{id}
    changed_date:  datetime | None = None                       # System.ChangedDate from ADO
    raw:           dict[str, Any] = Field(default_factory=dict, sa_column=Column(JSON))
    last_synced_at:datetime = Field(default_factory=utcnow)

    __table_args__ = (Index("ix_work_item_changed", "changed_date"),)
```

`raw` keeps the whole `$expand=all` response so the harvester can be re-run offline and so new field extractions don't need a re-fetch. It is pruned by the retention job (§8).

### 2.2 `Run` — the unit of work

```python
class Run(SQLModel, table=True):
    __tablename__ = "run"

    id:             str = Field(primary_key=True)               # uuid7
    work_item_id:   int = Field(foreign_key="work_item.id", index=True)
    state:          RunState = Field(default=RunState.CREATED, index=True)
    previous_state: RunState | None = None
    stage:          Stage | None = None                          # current stage, None when idle
    slug:           str                                          # kebab title, used in bug/<id>-<slug>
    fix_branch:     str | None = None                            # bug/<id>-<slug>, same name in every repo
    dir:            str                                          # abs path ~/.lazyboy/runs/<id>
    context_id:     str | None = Field(default=None, foreign_key="bug_context.id")

    # NOTE: branches are per-repo, not per-run. A full-stack bug spans a .NET repo and a
    # JS/TS repo with different default branches and different PR targets, so
    # base_branch / base_sha / pr_target_branch live on RepoCandidate (§2.4).
    # This is the normal case, not an edge case (master doc §8 constraint 3).

    # usage — turns and tokens are the meter; dollars only exist on an API key
    num_turns:      int   = 0                                    # PRIMARY budget signal
    max_turns:      int | None = None                            # copied from config at create
    task_budget_total: int | None = None                         # subagent fan-out ceiling
    input_tokens:   int   = 0
    output_tokens:  int   = 0
    cache_read_tokens:  int = 0
    cache_write_tokens: int = 0
    rate_limit_hits: int  = 0                                    # 429s survived (paused + resumed)
    cost_usd:       float | None = None                          # SECONDARY: null on a subscription

    # outcomes
    pr_id:          int | None = None
    pr_url:         str | None = None
    commit_sha:     str | None = None
    error:          dict[str, Any] | None = Field(default=None, sa_column=Column(JSON))

    last_seq:       int = 0                                      # monotonic RunEvent counter
    created_at:     datetime = Field(default_factory=utcnow)
    updated_at:     datetime = Field(default_factory=utcnow)
    finished_at:    datetime | None = None

    __table_args__ = (Index("ix_run_wi_created", "work_item_id", "created_at"),)
```

`cost_usd` is nullable and stays `None` for the whole life of a run on a Claude
subscription — the SDK reports no meaningful dollar figure and the UI must not invent one.
It is populated only when `usage.mode == "dollars"` (an `ANTHROPIC_API_KEY` is the live
credential). Everything that renders a budget reads `num_turns` / `max_turns` and the
token counters; `cost_usd` is a secondary column, never a required one.

`last_seq` lives on the run, not in a global sequence: `RunEvent.seq` is per-run monotonic, allocated under the same transaction that inserts the event, so SSE `?after=` is exact even across restarts.

### 2.3 `BugContext` — harvest output (JSON blob + typed model)

The table is a thin envelope; the payload is the pydantic model in §3.

```python
class BugContext(SQLModel, table=True):
    __tablename__ = "bug_context"

    id:           str = Field(primary_key=True)                  # uuid7
    run_id:       str = Field(foreign_key="run.id", index=True)
    work_item_id: int = Field(foreign_key="work_item.id", index=True)
    schema_version: int = 1
    payload:      dict[str, Any] = Field(sa_column=Column(JSON)) # BugContextModel.model_dump()
    file_path:    str                                            # runs/<id>/context.json
    harvested_at: datetime = Field(default_factory=utcnow)
    warnings:     list[str] = Field(default_factory=list, sa_column=Column(JSON))
```

Read path is always `BugContextModel.model_validate(row.payload)` — the raw dict never escapes the repository layer.

### 2.4 `RepoCandidate`

```python
class RepoCandidate(SQLModel, table=True):
    __tablename__ = "repo_candidate"

    id:          str = Field(primary_key=True)
    run_id:      str = Field(foreign_key="run.id", index=True)
    repo_name:   str
    repo_id:     str | None = None                     # ADO repository guid
    project:     str
    remote_url:  str | None = None
    default_branch: str | None = None
    confidence:  float                                  # 0.0–1.0
    resolved_by: ResolvedBy
    evidence:    list[dict[str, Any]] = Field(default_factory=list, sa_column=Column(JSON))
    selected:    bool = False                           # included in the worktree set
    worktree_path: str | None = None                    # runs/<id>/worktrees/<repo>

    # per-repo branching — N repos per run is the normal case
    base_branch:      str | None = None                 # chosen at the start_fix gate, per repo
    base_sha:         str | None = None                 # exact commit the worktree was cut from
    pr_target_branch: str | None = None                 # chosen at the create_pr gate, per repo
    checked_out_sha:  str | None = None                 # HEAD of the fix branch in this worktree
    pr_id:            int | None = None                 # one PR per repo
    pr_url:           str | None = None
    commit_sha:       str | None = None
    stack:            str | None = None                 # "dotnet" | "js" — picks the build commands
    restore_ok:       bool | None = None                # restore succeeded in THIS run (§2.10)
    created_at:  datetime = Field(default_factory=utcnow)

    __table_args__ = (UniqueConstraint("run_id", "repo_name", name="uq_repo_candidate_run_repo"),)
```

Because `.NET` back end and `JS/TS` front end live in separate repositories, a full-stack
bug produces **two or more selected `RepoCandidate` rows**, each with its own base branch,
base SHA, PR target and PR. `Run.pr_id` / `Run.pr_url` / `Run.commit_sha` remain as a
convenience pointer to the *primary* repo only (the one holding the defect site); anything
that enumerates outcomes must iterate `RepoCandidate where selected`. The UI groups the
diff review, the files-touched sidebar and the change-report findings by repo for the same
reason.

`evidence[]` entries are `{"kind": "stack_frame"|"assembly"|"role_name"|"catalog"|"code_search"|"symbol", "value": str, "detail": str, "score": float}`. The UI renders them as the "why this repo" list; the resolver's scoring is auditable.

### 2.5 `AgentSession`

```python
class AgentSession(SQLModel, table=True):
    __tablename__ = "agent_session"

    id:            str = Field(primary_key=True)         # uuid7 (LazyBoy's id)
    run_id:        str = Field(foreign_key="run.id", index=True)
    profile:       AgentProfile
    sdk_session_id: str | None = None                    # Claude session id, for resume=/fork_session
    parent_session_id: str | None = None                 # set when forked
    model:         str
    fallback_model: str | None = None
    effort:        str | None = None
    permission_mode: str
    cwd:           str
    options_snapshot: dict[str, Any] = Field(default_factory=dict, sa_column=Column(JSON))
    num_turns:     int = 0
    cost_usd:      float | None = None                   # None on a subscription
    rate_limit_pauses: int = 0                           # 429s waited out and resumed
    interrupted:   bool = False
    started_at:    datetime = Field(default_factory=utcnow)
    ended_at:      datetime | None = None
    end_reason:    str | None = None    # "completed"|"max_turns"|"task_budget"|"interrupt"|"error"
```

`options_snapshot` is the JSON-safe projection of the `ClaudeAgentOptions` actually used (prompts hashed, not inlined: `{"system_prompt_sha256": ..., "allowed_tools": [...], ...}`). It makes "why did this run behave differently" answerable after a prompt edit.

### 2.6 `RunEvent` — append-only

```python
class RunEvent(SQLModel, table=True):
    __tablename__ = "run_event"

    id:        int | None = Field(default=None, primary_key=True)   # rowid
    run_id:    str = Field(foreign_key="run.id", index=True)
    seq:       int                                                   # per-run, monotonic from 1
    type:      RunEventType = Field(index=True)
    stage:     Stage | None = None
    session_id: str | None = None                                    # AgentSession.id
    payload:   dict[str, Any] = Field(default_factory=dict, sa_column=Column(JSON))
    created_at:datetime = Field(default_factory=utcnow)

    __table_args__ = (
        UniqueConstraint("run_id", "seq", name="uq_run_event_seq"),
        Index("ix_run_event_run_seq", "run_id", "seq"),
    )
```

Insert is always `UPDATE run SET last_seq = last_seq + 1 RETURNING last_seq` then insert, in one transaction. No gaps, no duplicates, no cross-run coupling.

### 2.7 `Gate`

```python
class Gate(SQLModel, table=True):
    __tablename__ = "gate"

    id:         str = Field(primary_key=True)
    run_id:     str = Field(foreign_key="run.id", index=True)
    kind:       GateKind
    state:      GateState = Field(default=GateState.PENDING, index=True)
    payload:    dict[str, Any] = Field(default_factory=dict, sa_column=Column(JSON))
    decision:   dict[str, Any] | None = Field(default=None, sa_column=Column(JSON))
    auto:       bool = False                    # satisfied by config policy, not a human click
    opened_at:  datetime = Field(default_factory=utcnow)
    decided_at: datetime | None = None
    decided_by: str | None = None               # user upn, or "auto:<policy>"

    __table_args__ = (
        Index("ix_gate_run_state", "run_id", "state"),
    )
```

At most one `PENDING` gate per run at a time — enforced in `GateKeeper.open()` (raises `GateConflict`), not by a partial index, because SQLite partial-unique support is fine but the error message matters more.

`payload` shapes per kind:

| GateKind | `payload` | `decision` on approve |
|---|---|---|
| `start_investigation` | `{repos: [{name, base_branch}], max_turns, task_budget_total, model, effort}` | `{model?, effort?}` |
| `post_report` | `{report_id, preview_html, target: {work_item_id, project}}` | `{edited_markdown?}` |
| `start_fix` | `{report_id, proposed_fixes: [...], repos: [{name, stack, base_branch_options: [str], suggested_base, restore_status}]}` | `{bases: {repo: base_branch}, fix_index?, model?}` — one base branch **per repo**; the suggestion is pre-filled from the catalog but always confirmed |
| `post_change_report` | `{report_id, preview_html, diffstat}` | `{edited_markdown?}` |
| `commit` | `{by_repo: [{repo, diffstat, files: [{path, added, removed}]}], suggested_message, verification: {compiled, restored, tests_run: false, unverified_reason}}` | `{message, files?: [str]}` |
| `create_pr` | `{repos: [{repo, source_ref, target_branch_options, suggested_target, reviewers, policies}], suggested_title, suggested_description}` | `{per_repo: {repo: {pr_target_branch, title, description, is_draft, reviewers: [guid]}}}` — one PR per repo |
| `mark_needs_repo` | `{evidence_summary, proposed_tag, proposed_comment_html}` | `{comment_html?}` |
| `tool_permission` | `{tool_name, input_data, reason, suggestions}` | `{allow: bool, updated_input?, message?}` |

### 2.8 `Report`

```python
class Report(SQLModel, table=True):
    __tablename__ = "report"

    id:          str = Field(primary_key=True)
    run_id:      str = Field(foreign_key="run.id", index=True)
    kind:        ReportKind
    version:     int = 1                        # bumped when the agent revises after feedback
    markdown_path: str                          # runs/<id>/investigation.md
    markdown:    str                            # duplicated for the API; file is the human artifact
    findings:    dict[str, Any] | None = Field(default=None, sa_column=Column(JSON))
    findings_path: str | None = None            # runs/<id>/findings.json
    patch_path:  str | None = None              # runs/<id>/changes.patch (ChangeReport only)
    session_id:  str | None = Field(default=None, foreign_key="agent_session.id")
    posted_at:   datetime | None = None
    posted_comment_id: int | None = None        # ADO comment id
    created_at:  datetime = Field(default_factory=utcnow)

    __table_args__ = (
        UniqueConstraint("run_id", "kind", "version", name="uq_report_run_kind_version"),
    )
```

`findings` conforms to the `InvestigationFindings` / `ChangeSummary` JSON schemas in [`agent-contracts.md`](agent-contracts.md) §4 — validated on write; a validation failure sets `Run.error` and emits `error` rather than silently storing a malformed blob.

### 2.9 `CredentialRef`

```python
class CredentialRef(SQLModel, table=True):
    __tablename__ = "credential_ref"

    id:          str = Field(primary_key=True)
    kind:        CredentialKind
    service:     str = "lazyboy"                 # keyring service name
    account:     str                             # keyring account/username key
    label:       str                             # "ADO PAT (personal, org zealit)"
    scopes:      list[str] = Field(default_factory=list, sa_column=Column(JSON))
    tenant_id:   str | None = None
    expires_at:  datetime | None = None          # PAT expiry, for the 7-day UI warning
    last_verified_at: datetime | None = None
    last_error:  str | None = None
    active:      bool = True
    created_at:  datetime = Field(default_factory=utcnow)

    __table_args__ = (UniqueConstraint("kind", "account", name="uq_credential_kind_account"),)
```

**No secret material.** `CredentialVault.get(ref)` calls `keyring.get_password(ref.service, ref.account)`. For `ENTRA_SSO` there is nothing to fetch — the row records which tenant/credential chain succeeded so the UI can display it and the health check can re-probe.

### 2.10 `CatalogEntry`

The materialized form of `repos.yaml`, plus learned entries.

```python
class CatalogEntry(SQLModel, table=True):
    __tablename__ = "catalog_entry"

    id:           str = Field(primary_key=True)
    repo_name:    str = Field(index=True)
    repo_id:      str | None = None
    project:      str
    remote_url:   str | None = None
    default_branch: str = "main"
    # match keys — any hit maps to this repo
    assemblies:   list[str] = Field(default_factory=list, sa_column=Column(JSON))
    role_names:   list[str] = Field(default_factory=list, sa_column=Column(JSON))  # cloud_RoleName
    namespaces:   list[str] = Field(default_factory=list, sa_column=Column(JSON))  # prefix match
    path_globs:   list[str] = Field(default_factory=list, sa_column=Column(JSON))
    build_definition_ids: list[int] = Field(default_factory=list, sa_column=Column(JSON))
    owners:       list[str] = Field(default_factory=list, sa_column=Column(JSON))
    language:     str | None = None              # "csharp"|"typescript"|"python"|...
    # build — compile only; there is deliberately no test_cmd, LazyBoy never runs tests
    restore_cmd:  str | None = None              # "dotnet restore" | "npm ci"
    build_cmd:    str | None = None              # "dotnet build -c Debug" | "npm run build"
    typecheck_cmd: str | None = None             # "npx tsc --noEmit"
    lint_cmd:     str | None = None              # "dotnet format --verify-no-changes" | "npm run lint"
    default_pr_target: str | None = None         # per-repo PR target; varies across repos
    # restore health — recorded by `lazyboy catalog scan` and refreshed by the Phase 1 probe.
    # Compile-only verification is worth nothing if restore against the private ADO
    # Artifacts feed fails, so this tells you which repos LazyBoy can actually verify
    # BEFORE you start a run on one.
    restore_status: str = "unknown"              # "ok"|"failed"|"unknown"
    restore_checked_at: datetime | None = None
    restore_error: str | None = None             # verbatim first error line, feed name included
    restore_feed:  str | None = None             # the ADO Artifacts feed the repo restores from
    source:       str = "repos.yaml"             # "repos.yaml"|"learned"|"ado_sync"
    hit_count:    int = 0                        # bumped when a resolution using it is approved
    updated_at:   datetime = Field(default_factory=utcnow)

    __table_args__ = (UniqueConstraint("repo_name", "project", name="uq_catalog_repo_project"),)
```

A repo with `restore_status="failed"` is surfaced in the inbox and in the resolver's repo
picker with a warning: a run on it can still investigate, but the fixer will not be able
to claim `compiled: true` and its Change Report will carry
`unverified_reason: "package restore failed"`. `restore_status` is per repo and per run —
`RepoCandidate.restore_ok` (§2.4) records whether restore actually succeeded *in that run*,
which is the value the `compiled` claim is gated on.

`repos.yaml` is the source of truth for `source="repos.yaml"` rows; the loader does a full delete-and-reinsert of those on startup, leaving `learned` rows alone. `hit_count` feeds resolver tie-breaks and gives you a cheap "promote this learned entry into repos.yaml" prompt.

---

## 3. Typed payloads (pydantic)

### 3.1 `BugContextModel` — the `BugContext.payload`

```python
# src/lazyboy/domain/context.py
from datetime import datetime
from typing import Literal
from pydantic import BaseModel, Field, HttpUrl


class Attachment(BaseModel):
    name: str
    local_path: str                     # runs/<id>/attachments/<name>
    ado_url: str
    content_type: str | None = None
    size_bytes: int | None = None
    inline: bool = False                # was embedded in description HTML
    ocr_text: str | None = None         # optional, only if a text-extraction pass ran


class StackFrame(BaseModel):
    level: int
    assembly: str | None = None
    method: str | None = None
    file_name: str | None = None
    line: int | None = None
    raw: str


class ExceptionRecord(BaseModel):
    item_id: str
    timestamp: datetime
    problem_id: str | None = None
    type: str | None = None
    outer_message: str | None = None
    cloud_role_name: str | None = None
    parsed_stack: list[StackFrame] = Field(default_factory=list)


class TelemetryNode(BaseModel):
    item_id: str
    item_type: Literal["request", "exception", "dependency", "trace",
                       "customEvent", "pageView", "availabilityResult"]
    id: str | None = None
    parent_id: str | None = None
    timestamp: datetime
    name: str | None = None
    message: str | None = None
    target: str | None = None
    result_code: str | None = None
    success: bool | None = None
    duration_ms: float | None = None
    cloud_role_name: str | None = None
    custom_dimensions: dict[str, str] = Field(default_factory=dict)
    children: list["TelemetryNode"] = Field(default_factory=list)


class AppInsightsTransaction(BaseModel):
    resource_id: str
    resource_name: str
    event_id: str | None = None
    operation_id: str | None = None
    operation_name: str | None = None
    timestamp: datetime | None = None
    portal_url: str | None = None
    roles: list[str] = Field(default_factory=list)         # distinct cloud_RoleName
    root: TelemetryNode | None = None
    exceptions: list[ExceptionRecord] = Field(default_factory=list)
    node_count: int = 0
    truncated: bool = False                                 # hit the take-1000 cap
    partial_error: str | None = None                        # LogsQueryPartialResult
    raw_path: str | None = None                             # runs/<id>/transaction.json


class RelatedItem(BaseModel):
    id: int
    rel: str                    # "System.LinkTypes.Duplicate-Forward", "Child", ...
    title: str | None = None
    state: str | None = None
    url: str | None = None


class CandidateSymbol(BaseModel):
    symbol: str                 # "CheckoutService.ApplyDiscount"
    kind: Literal["type", "method", "namespace", "assembly", "file", "route", "sql_object"]
    source: Literal["stack", "description", "repro_steps", "system_info", "attachment"]
    occurrences: int = 1


class BugContextModel(BaseModel):
    schema_version: int = 1
    work_item_id: int
    org: str
    project: str
    title: str
    state: str
    work_item_type: str
    url: str
    description_text: str = ""          # HTML → text, links preserved as markdown
    description_html: str = ""          # rewritten: inline images → local paths
    repro_steps_text: str = ""
    system_info_text: str = ""
    tags: list[str] = Field(default_factory=list)
    attachments: list[Attachment] = Field(default_factory=list)
    hyperlinks: list[str] = Field(default_factory=list)
    related: list[RelatedItem] = Field(default_factory=list)
    artifact_links: list[dict] = Field(default_factory=list)   # commits/PRs/builds from relations
    app_insights_urls: list[str] = Field(default_factory=list)
    transaction: AppInsightsTransaction | None = None
    stack_frames: list[StackFrame] = Field(default_factory=list)   # deduped, flattened, ranked
    candidate_symbols: list[CandidateSymbol] = Field(default_factory=list)
    role_names: list[str] = Field(default_factory=list)
    assemblies: list[str] = Field(default_factory=list)
    environment: str | None = None       # "prod"|"uat"|... inferred from resource/tags
    harvest_warnings: list[str] = Field(default_factory=list)
```

`description_text`/`repro_steps_text` are **untrusted**. They are never concatenated into a system prompt; they're delivered fenced (see [`agent-contracts.md`](agent-contracts.md) §7) or pulled by the agent through `mcp__lazyboy__ado_get_work_item`.

### 3.2 Usage accounting

The meter is **turns and tokens against the rate-limit window**, not money. On a Claude
subscription there is no per-run dollar figure, so `usd` is nullable and never drives a
gauge; the type keeps it only so a later switch to an `ANTHROPIC_API_KEY` is a config
change rather than a schema change.

```python
class UsageSnapshot(BaseModel):
    # primary
    turns: int                                  # cumulative assistant turns this run
    max_turns: int | None = None                # the ceiling; the gauge is turns/max_turns
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int
    cache_write_tokens: int = 0
    task_budget_used: int | None = None         # subagent tasks spawned / total
    task_budget_total: int | None = None
    limit_hit: bool = False                     # a 429 was seen; run paused, will resume
    limit_resets_at: datetime | None = None     # from Retry-After / rate-limit headers
    # secondary — populated only when usage.mode == "dollars" (API key present)
    usd: float | None = None

# Back-compat alias: the type was called CostSnapshot when budgets were dollar-denominated.
CostSnapshot = UsageSnapshot
```

---

## 4. RunEvent taxonomy

Every SSE frame is:

```
event: <RunEventType>
id: <seq>
data: {"run_id": "...", "seq": 12, "type": "agent.tool_use", "stage": "investigate",
       "session_id": "...", "ts": "2026-08-19T09:41:02.113Z", "payload": {...}}
```

The client reconnects with `Last-Event-ID` (or `?after=<seq>`); the server replays `SELECT * FROM run_event WHERE run_id=? AND seq>? ORDER BY seq` then attaches to the live bus. Replay and live are the same shape, so the reducer has one code path.

| Type | When | `payload` |
|---|---|---|
| `run.state_changed` | every state machine transition | `{"from": RunState, "to": RunState, "reason": str, "gate_id": str \| null}` |
| `agent.text` | assistant text block (streamed if `include_partial_messages`) | `{"text": str, "delta": bool, "block_index": int}` |
| `agent.thinking` | extended-thinking block | `{"text": str, "delta": bool, "signature_present": bool}` |
| `agent.tool_use` | tool call issued, **after** `can_use_tool` allowed it | `{"tool_use_id": str, "tool_name": str, "input": dict, "input_truncated": bool, "subagent": str \| null}` |
| `agent.tool_result` | result returned to the model | `{"tool_use_id": str, "tool_name": str, "is_error": bool, "summary": str, "bytes": int, "duration_ms": int, "content_path": str \| null}` |
| `agent.permission_denied` | `can_use_tool` returned `PermissionResultDeny` | `{"tool_name": str, "input": dict, "rule": str, "message": str, "interrupt": bool}` |
| `agent.finding` | `mcp__lazyboy__record_finding` called | `{"finding_id": str, "severity": "info"\|"low"\|"medium"\|"high"\|"critical", "title": str, "detail": str, "file": str \| null, "line": int \| null, "confidence": float, "evidence": [str]}` |
| `stage.started` | orchestrator enters a stage | `{"stage": Stage, "inputs": {...}}` |
| `stage.completed` | stage finished | `{"stage": Stage, "ok": bool, "duration_ms": int, "outputs": {...}}` |
| `gate.opened` | a gate needs you | `{"gate_id": str, "kind": GateKind, "payload": {...}, "auto": bool}` |
| `gate.decided` | approved/rejected/expired | `{"gate_id": str, "kind": GateKind, "state": GateState, "decision": {...}, "decided_by": str}` |
| `cost.updated` | after every SDK `ResultMessage` / usage delta, and on every 429 | `{"turns": int, "input_tokens": int, "output_tokens": int, "cache_read_tokens": int, "usd": float \| null, "limit_hit": bool}`, plus `{"session_id": str}` |
| `error` | anything raised, agent or connector | `{"where": str, "kind": str, "message": str, "retryable": bool, "attempt": int, "traceback_path": str \| null}` |

Payload rules that keep the DB small and the UI fast:

- `agent.tool_use.input` is truncated to **4 KB**; the full input is in `audit.jsonl`, `input_truncated` flags it.
- `agent.tool_result` never stores the content. It stores a ≤512-char `summary` (first line + shape) and, for results >8 KB, `content_path` pointing at `runs/<id>/tool_results/<tool_use_id>.json`.
- `agent.text`/`agent.thinking` deltas are **not** persisted; only the completed block is written to SQLite. Deltas go straight to the bus (`delta: true`) for typing-effect rendering. On replay you get whole blocks — visually different, semantically identical.
- `cost.updated` keeps its name for wire compatibility (the SSE event type and the reducer key are unchanged), but its payload is a usage snapshot: `usd` is `null` on a subscription and the UI renders turns and tokens. `limit_hit: true` means a rate limit was reached — the run is paused, not failed, and a later `cost.updated` with `limit_hit: false` marks the resume.
- Every event that the audit trail needs (`tool_use`, `tool_result`, `permission_denied`, `gate.*`, `run.state_changed`) is written to `audit.jsonl` in full before the truncated copy hits SQLite.

### 4.1 Normalizer mapping (Agent SDK → RunEvent)

| SDK message | Emitted |
|---|---|
| `AssistantMessage` w/ `TextBlock` | `agent.text` |
| `AssistantMessage` w/ `ThinkingBlock` | `agent.thinking` |
| `AssistantMessage` w/ `ToolUseBlock` | `agent.tool_use` |
| `UserMessage` w/ `ToolResultBlock` | `agent.tool_result` |
| `StreamEvent` (partial) | `agent.text`/`agent.thinking` with `delta: true`, bus-only |
| `ResultMessage` | `cost.updated` + `stage.completed` |
| `SystemMessage(subtype="init")` | recorded on `AgentSession` (`sdk_session_id`, model), no event |
| `can_use_tool` → Deny | `agent.permission_denied` |
| SDK exception / `ProcessError` | `error` |

---

## 5. State transition table

Guards are evaluated by `RunStateMachine.can(run, trigger)`; side effects run inside the same DB transaction as the state write, except external I/O which runs after commit and is idempotent (retry-safe).

| From | Trigger (event) | To | Guard | Side effects |
|---|---|---|---|---|
| — | `run.create(work_item_id)` | `created` | work item exists in cache or fetchable | mkdir `runs/<id>`, copy `max_turns` / `task_budget_total` defaults from config |
| `created` | `harvest.start` | `harvesting` | credentials healthy | `stage.started(harvest)` |
| `harvesting` | `harvest.ok` | `resolving` | `BugContext` persisted | write `context.json`, `transaction.json`, attachments; `stage.completed(harvest)` |
| `harvesting` | `harvest.fail` | `failed` | non-retryable, or attempts > 3 | `error`; keep partial context |
| `resolving` | `resolve.ok` | `ready_to_investigate` | ≥1 `RepoCandidate` with `selected=True` | clone/refresh shared clone, create worktrees, `stage.completed(resolve)` |
| `resolving` | `resolve.empty` | `blocked_no_repo` | 0 candidates ≥ `min_confidence` (default 0.35) | open `mark_needs_repo` gate |
| `blocked_no_repo` | `repo.attach(manual)` | `resolving` | repo exists in ADO | insert `RepoCandidate(resolved_by=manual, confidence=1.0)` |
| `blocked_no_repo` | `gate.mark_needs_repo.approved` | `blocked_no_repo` | — | PATCH tags + post comment; stays blocked |
| `ready_to_investigate` | `gate.start_investigation.approved` | `investigating` | worktrees exist, turns remaining | create `AgentSession(investigator)`, connect `ClaudeSDKClient` |
| `investigating` | `investigate.ok` | `report_ready` | `findings.json` validates against `InvestigationFindings` | write `investigation.md`, insert `Report(kind=investigation)`, disconnect client |
| `investigating` | `investigate.needs_more_info` | `report_ready` | `needs_more_info=true` | same, UI flags the open questions banner |
| `investigating` | `run.interrupt` | `ready_to_investigate` | — | `client.interrupt()`, keep `sdk_session_id` for `resume=` |
| `investigating` | `investigate.fail` | `failed` | — | `error`, keep session id for forensic resume |
| `report_ready` | `gate.post_report.approved` | `report_posted` | report exists | render markdown→ADO HTML subset, upload inline images, POST comment, set `Report.posted_*` |
| `report_ready` \| `report_posted` | `gate.start_fix.approved` | `branch_prepared` | ≥1 proposed fix, a base branch chosen **per selected repo** | for each selected `RepoCandidate`: fetch its base, record `base_sha`, `git worktree` checkout `bug/<id>-<slug>` off that repo's `base_branch` |
| `report_ready` | `report.revise` | `investigating` | — | resume session with reviewer feedback, `Report.version += 1` |
| `branch_prepared` | `fix.start` | `fixing` | branch clean, path jail configured | create `AgentSession(fixer)` (`fork_session` from investigator when configured) |
| `fixing` | `fix.ok` | `changes_ready` | worktree dirty, `ChangeSummary` validates | write `change-report.md`, `git diff > changes.patch`, insert `Report(kind=change)` |
| `fixing` | `fix.no_changes` | `report_ready` | worktree clean after run | `error(kind="no_changes")`, back to report for a different approach |
| `fixing` | `run.interrupt` | `branch_prepared` | — | `client.interrupt()`; worktree left as-is |
| `fixing` | `fix.fail` | `failed` | — | `error`; worktree preserved for inspection |
| `changes_ready` | `gate.post_change_report.approved` | `change_posted` | — | POST comment to ADO |
| `changes_ready` \| `change_posted` | `gate.commit.approved` | `committed` | ≥1 worktree dirty, no merge conflicts | per dirty repo: `git add <selected>` + `git commit -m <msg>` (trailer `AB#<id>`), record `RepoCandidate.commit_sha` |
| `committed` | `gate.create_pr.approved` | `pr_created` | a commit and a `pr_target_branch` exist for every selected repo | per repo: `git push -u origin <fix_branch>`, `POST .../pullrequests` into that repo's `pr_target_branch` with `workItemRefs` + `AB#<id>`, record `RepoCandidate.pr_id`/`pr_url`. N repos → N PRs, all linked to the one work item |
| `pr_created` | `run.finish` | `done` | — | set `finished_at`, prune worktree (config), `stage.completed(publish)` |
| `changes_ready` | `fix.revise` | `fixing` | — | resume fixer session with feedback |
| *any non-terminal* | `run.cancel` | `cancelled` | — | interrupt + disconnect client, expire pending gates, keep artifacts |
| `failed` \| `cancelled` | `run.retry` | `previous_state` | `previous_state ∈ RESUMABLE_STATES` | clear `error`, re-enter the stage idempotently |

Illegal transitions raise `InvalidTransition` → HTTP 409 with `{allowed: [triggers]}`. The state machine is a pure function `(state, trigger) -> state | error`; every mutation goes through it, including recovery paths, so there is exactly one place that knows the graph.

---

## 6. REST DTOs

Response models only — request models are the small ones inline. All are pydantic v2, `model_config = ConfigDict(from_attributes=True)`.

```python
# src/lazyboy/api/dto.py

class WorkItemSummary(BaseModel):
    id: int
    title: str
    state: str
    work_item_type: str
    project: str
    tags: list[str]
    priority: int | None
    severity: str | None
    url: str
    changed_date: datetime | None
    last_run: "RunSummary | None" = None      # most recent run for this item


class RunSummary(BaseModel):
    id: str
    work_item_id: int
    work_item_title: str
    state: RunState
    stage: Stage | None
    fix_branch: str | None
    pr_urls: list[str]                        # one per repo; a full-stack fix opens several
    usage: UsageSnapshot                      # turns + tokens; usd is null on a subscription
    pending_gate: "GateOut | None"
    last_seq: int
    created_at: datetime
    updated_at: datetime


class RunDetail(RunSummary):
    context: BugContextModel | None
    repos: list["RepoCandidateOut"]
    sessions: list["AgentSessionOut"]
    reports: list["ReportOut"]
    gates: list["GateOut"]
    error: dict | None


class RepoCandidateOut(BaseModel):
    id: str
    repo_name: str
    project: str
    confidence: float
    resolved_by: ResolvedBy
    evidence: list[dict]
    selected: bool
    worktree_path: str | None
    default_branch: str | None
    base_branch: str | None
    base_sha: str | None
    pr_target_branch: str | None
    pr_url: str | None
    stack: str | None                         # "dotnet" | "js"
    restore_ok: bool | None                   # restore succeeded in this run


class AgentSessionOut(BaseModel):
    id: str
    profile: AgentProfile
    model: str
    num_turns: int
    max_turns: int | None
    cost_usd: float | None                    # null unless usage.mode == "dollars"
    started_at: datetime
    ended_at: datetime | None
    end_reason: str | None


class ReportOut(BaseModel):
    id: str
    kind: ReportKind
    version: int
    markdown: str
    findings: dict | None
    posted_at: datetime | None
    posted_comment_id: int | None
    created_at: datetime


class GateOut(BaseModel):
    id: str
    kind: GateKind
    state: GateState
    payload: dict
    decision: dict | None
    auto: bool
    opened_at: datetime
    decided_at: datetime | None


class GateDecisionIn(BaseModel):
    approve: bool
    decision: dict = Field(default_factory=dict)   # shape per GateKind, §2.7
    note: str | None = None


class RunEventOut(BaseModel):
    seq: int
    type: RunEventType
    stage: Stage | None
    session_id: str | None
    payload: dict
    ts: datetime


class HealthOut(BaseModel):
    ado: "ProbeOut"
    app_insights: "ProbeOut"
    claude: "ProbeOut"
    git: "ProbeOut"
    code_search_available: bool


class ProbeOut(BaseModel):
    ok: bool
    identity: str | None = None
    detail: str | None = None
    credential_kind: CredentialKind | None = None
    expires_at: datetime | None = None
```

Endpoint → DTO map:

| Method | Path | Body | Returns |
|---|---|---|---|
| `GET` | `/api/workitems` | — | `list[WorkItemSummary]` |
| `POST` | `/api/workitems/sync` | — | `{synced: int}` |
| `GET` | `/api/workitems/{id}` | — | `WorkItemSummary` + raw fields |
| `POST` | `/api/runs` | `{work_item_id \| url}` | `RunDetail` (state `created`) |
| `GET` | `/api/runs` | — | `list[RunSummary]` |
| `GET` | `/api/runs/{id}` | — | `RunDetail` |
| `POST` | `/api/runs/{id}/advance` | `{trigger}` | `RunSummary` |
| `POST` | `/api/runs/{id}/interrupt` | — | `RunSummary` |
| `POST` | `/api/runs/{id}/cancel` | — | `RunSummary` |
| `GET` | `/api/runs/{id}/events?after=<seq>` | — | `list[RunEventOut]` |
| `GET` | `/api/runs/{id}/stream?after=<seq>` | — | `text/event-stream` |
| `POST` | `/api/runs/{id}/gates/{gate_id}` | `GateDecisionIn` | `GateOut` |
| `POST` | `/api/runs/{id}/repos` | `{repo_name, project}` | `RepoCandidateOut` |
| `GET` | `/api/runs/{id}/diff` | — | `text/plain` (changes.patch) |
| `GET` | `/api/repos/branches?repo=` | — | `list[str]` |
| `GET` | `/api/health` | — | `HealthOut` |

---

## 7. Migrations

No Alembic. Single-user, single-writer, one process — the ceremony costs more than it buys.

```
src/lazyboy/db/migrations/
├── 0001_initial.sql
├── 0002_add_catalog_hit_count.sql
├── 0003_run_cache_tokens.sql
├── 0004_per_repo_branches.sql        # base_branch/base_sha/pr_target_branch → repo_candidate
└── 0005_catalog_restore_health.sql   # restore_status/checked_at/error/feed on catalog_entry
```

```sql
-- always present, created by the runner before anything else
CREATE TABLE IF NOT EXISTS schema_version (
    version     INTEGER PRIMARY KEY,
    name        TEXT NOT NULL,
    applied_at  TEXT NOT NULL,
    checksum    TEXT NOT NULL          -- sha256 of the script, detects edited history
);
```

Runner, executed at startup before FastAPI serves anything:

```python
def migrate(conn: sqlite3.Connection, scripts_dir: Path) -> None:
    conn.execute("PRAGMA foreign_keys=ON")
    conn.execute("PRAGMA journal_mode=WAL")
    ensure_schema_version_table(conn)
    applied = {v: c for v, c in conn.execute("SELECT version, checksum FROM schema_version")}
    for path in sorted(scripts_dir.glob("[0-9][0-9][0-9][0-9]_*.sql")):
        version = int(path.name[:4])
        sql = path.read_text()
        checksum = hashlib.sha256(sql.encode()).hexdigest()
        if version in applied:
            if applied[version] != checksum:
                raise MigrationDrift(f"{path.name} changed after being applied")
            continue
        backup_db_once(conn)                      # cp lazyboy.db lazyboy.db.bak-<version>
        with conn:                                # one transaction per script
            conn.executescript(sql)
            conn.execute(
                "INSERT INTO schema_version VALUES (?,?,?,?)",
                (version, path.stem, utcnow().isoformat(), checksum),
            )
```

Rules:

- Scripts are **append-only and immutable** once shipped; the checksum check enforces it.
- Every script is written to be safe under `executescript` in one transaction. SQLite's `ALTER TABLE ADD COLUMN` is cheap; for anything else use the 12-step table rebuild (`CREATE new`, `INSERT SELECT`, `DROP old`, `ALTER RENAME`) inside the same script.
- A pre-migration file copy of `lazyboy.db` is taken once per upgrade session. If a migration throws, the process exits with the backup path in the message. There is no down-migration — restore the backup.
- SQLModel `create_all()` is **not** used at runtime. `0001_initial.sql` is generated from the models once, by hand, and then models and SQL evolve together; a startup assertion compares `PRAGMA table_info` against the model metadata and fails loudly on drift in dev (`LAZYBOY_STRICT_SCHEMA=1`).
- `BugContext.payload` and `Report.findings` carry their own `schema_version`. Payload shape changes are handled by an upcaster in Python (`upcast_context(payload) -> BugContextModel`), not by a SQL migration — the blobs are large and rewriting them is pointless when the reader can adapt.

---

## 8. Retention & cleanup

`runs/` is where the disk goes. A cleanup pass runs at startup and then every 6 hours, driven by `config.yaml`:

```yaml
retention:
  keep_runs_days: 30            # runs in a terminal state older than this are swept
  keep_done_runs: 50            # ...but always keep the N most recent done runs
  keep_failed_runs_days: 14
  prune_worktrees_after_done: true
  max_runs_dir_gb: 5            # hard cap; oldest-first eviction beyond it
  keep_events_per_run: 20000    # trim oldest agent.text/thinking beyond this
  vacuum_on_startup_if_free_pages_pct: 25
```

Policy, in order:

1. **Worktrees** — on `done`/`cancelled`, `git worktree remove --force runs/<id>/worktrees/<repo>` and `git worktree prune` in the shared clone. The shared clone under `workspace/<repo>/` is never deleted by retention; it is the expensive artifact and is reused. A separate `workspace.max_age_days: 90` sweeps clones untouched for a quarter.
2. **Tool result spill files** — `runs/<id>/tool_results/` is deleted as soon as the run leaves an active state. Nothing references it after the report is written.
3. **Attachments** — kept while the run is retained; they are also re-downloadable from ADO, so they are the first thing evicted when `max_runs_dir_gb` is exceeded.
4. **Reports and audit** — `investigation.md`, `findings.json`, `change-report.md`, `changes.patch`, `audit.jsonl`, `context.json` are the *keepers*. When a run directory is swept, these are tarred to `runs/_archive/<run-id>.tar.zst` (typically <200 KB) and the directory is removed. The archive is kept for `keep_runs_days * 4`.
5. **Events** — a run over `keep_events_per_run` has its oldest `agent.text`/`agent.thinking`/`agent.tool_result` rows deleted (never `run.state_changed`, `gate.*`, `agent.finding`, `error` — those are the audit spine). A tombstone event `{"type":"error","payload":{"kind":"events_trimmed","removed":N}}` is inserted so the UI can explain the gap.
6. **Database rows** — a swept run keeps its `Run`, `Report`, and `Gate` rows (they're tiny and drive history); its `RunEvent` rows are deleted wholesale and `Run.dir` is repointed at the archive path.
7. **`WorkItem.raw`** — nulled for items with no run in the last 30 days.
8. **VACUUM** — after any sweep that deleted >10k rows, run `PRAGMA incremental_vacuum` (auto_vacuum=INCREMENTAL is set in `0001_initial.sql`).

Manual escape hatches: `lazyboy prune --dry-run`, `lazyboy prune --run <id>`, and a per-run **Pin** flag (`Run.error` is not it — pinning is `CatalogEntry`-style metadata stored as `run.pinned` boolean added in `0002`) which exempts a run from every rule above.

---

## 9. Indexing summary

| Index | Table | Columns | Query it serves |
|---|---|---|---|
| `ix_run_event_run_seq` | `run_event` | `(run_id, seq)` | SSE replay `seq > ?` — the hottest query in the app |
| `ix_run_state` | `run` | `state` | inbox badges, "active runs" poll |
| `ix_run_wi_created` | `run` | `(work_item_id, created_at)` | latest run per work item |
| `ix_gate_run_state` | `gate` | `(run_id, state)` | "is there a pending gate" on every run fetch |
| `ix_work_item_changed` | `work_item` | `changed_date` | inbox ordering |
| `ix_catalog_repo` | `catalog_entry` | `repo_name` | resolver lookups |

Match keys in `CatalogEntry` (`assemblies`, `role_names`, `namespaces`) are JSON arrays — the resolver loads the whole catalog into memory at startup (hundreds of rows at most) and matches there. No JSON1 indexes, no `json_each` joins.
