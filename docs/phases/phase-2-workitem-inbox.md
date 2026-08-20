# Phase 2 — Work Item Inbox

Part of [LazyBoy Master Design](../LazyBoy-Design.md).

> Depends on Phase 1 (`CredentialVault` yields a working ADO auth header). Unblocks Phase 3 — the Inbox's "Investigate" button is the only production entry point into `POST /api/runs`.

---

## Goal

Turn "open ADO, filter, squint, copy the id" into a single screen that is *already* filtered to you, already sorted by what changed most recently, and one click away from starting a run. The Inbox is also where LazyBoy's `AdoClient` is born — every later phase (comments, attachments, repos, PRs) reuses it, so the connector design carries more weight here than the UI does.

Two entry paths, both must work:

1. **Assigned to me** — WIQL `@Me`, **project-scoped**, cached in SQLite, incrementally synced.
2. **Manual** — paste an id or *any* ADO work-item URL shape into a box and go, even for an item that belongs to someone else or is filtered out of the default query.

**Scope, settled ([master §8](../LazyBoy-Design.md#8-environment-constraints-answered)):** one org, one project. Every call is project-scoped, there is no project column and no cross-project disambiguation. Org-scope survives only as the config flag `ado.scope: project|org` (§2.6) so a second project later is a config change, not a rewrite.

**Work item types, settled (master §8 constraint 12):** **everything assigned to you**, whatever its type. The WIQL carries **no `System.WorkItemType` clause** — a Task that is really a bug, a User Story with a defect hiding in it, and a Support Request are all things you are expected to work on, and a type filter would hide exactly the items you would otherwise have to go to ADO for. The inbox instead *ranks* rather than *excludes*: every row shows a **type badge**, the default sort puts **Bugs first**, and non-Bug types are investigable on equal terms but carry a **best-effort marker on the fix stage** (§5.1) — the fix engine is tuned for "something is broken, here is the stack", and a User Story asking for a new feature is not that.

## Definition of Done

- [ ] `GET /api/workitems` returns cached items in < 50 ms warm, with filters for `state`, `type`, `tag`, `q`, `assigned`, and cursor pagination.
- [ ] `POST /api/workitems/sync` performs an incremental sync and returns `{fetched, updated, unchanged, duration_ms}`.
- [ ] A cold sync of 200 items completes in ≤ 3 s over a home connection; warm incremental sync of 0 changed items in ≤ 400 ms (one WIQL round-trip, no batch call).
- [ ] Every cached item carries `has_appinsights_link`, `has_attachments`, `brief` (plain-text ≤ 280 chars), `state_category`, `priority`, `severity`.
- [ ] Inbox renders a sortable, filterable table; keyboard `j/k/Enter//` work; empty/loading/error/stale states all have designed treatments.
- [ ] The assigned-to-me WIQL contains **no type clause**; every assigned type appears. Each row shows a type badge, the default sort is Bugs-first-then-changed-date, and non-Bug rows carry the best-effort fix marker.
- [ ] Pasting `https://dev.azure.com/zeal/Portal/_workitems/edit/12345`, `.../_boards/board/t/Team/Stories/?workitem=12345`, `https://zeal.visualstudio.com/…`, or plain `12345` all resolve to the same item.
- [ ] A URL naming a project other than the configured one is rejected with a clear message rather than silently queried — one org, one project is an invariant, not a default.
- [ ] "Investigate" creates a `Run` in `created` and navigates to the Bug Workspace (which is a Phase-3 shell at this point).
- [ ] 429 from ADO is retried per `Retry-After` and never surfaces as a failed sync; a hard failure surfaces as a non-blocking banner over stale cache.

---

## Design

### 2.1 `AdoClient` — the shared connector

One `httpx.AsyncClient` for the whole process (P7: one process, one pool). HTTP/2 on, 30 s timeout, 20 keep-alive connections. `AdoClient` is *not* a per-request object; it is constructed once at app startup for the configured org+project and stored on `app.state.ado`.

Design commitments:

| Concern | Decision |
|---|---|
| Auth | **Reuses Phase 1's `AdoAuth`** (`httpx.Auth` subclass) rather than a header dict — it is injected at construction. Token refresh, the single silent 401 retry, and `ReauthRequired` are Phase 1's concern; `AdoClient` never sees a raw token and never re-implements refresh. |
| Scoping | **Everything is project-scoped.** The client holds the single configured `project` and injects it into every route — WIQL, `workitemsbatch`, comments, attachments alike (comments and attachments 404 without it anyway). No per-call `project` argument, no `project_by_id` learning, no merge step. `ado.scope: org` flips the two scopable routes back to org scope and is unused in v1. |
| Retry | `tenacity` with `Retry-After` honoured exactly, then exponential backoff + full jitter, cap 5 attempts, only on `429/500/502/503/504` and `httpx.TransportError`. `401` is *not* retried — it raises `AdoAuthError`, which the API layer turns into a 401 with `reauth_required: true`. |
| Pagination | ADO uses two schemes: `continuationToken` in the `x-ms-continuationtoken` response header (repos, refs, comments) and `$top`/`$skip` (some older routes). `_paginate()` handles the header form and yields items; batch endpoints are chunked at 200. |
| Conditional GET | Work item GETs support `If-None-Match` against the item's ETag. LazyBoy stores `etag` per work item and sends it on detail refresh; a `304` costs one round trip and zero parsing. WIQL and `workitemsbatch` are `POST` and do **not** support it — hence the `System.ChangedDate` watermark strategy below. |
| Concurrency | An `asyncio.Semaphore(8)` prevents a 200-item hydration fan-out from tripping TSTU throttling. |
| Redaction | The client's logger runs the global redacting filter; `Authorization` is never logged, and URLs are logged with query strings stripped of `token`/`sig`. |

### 2.2 Fetch path: WIQL → batch

```
WIQL (project scope, ids only, ORDER BY ChangedDate DESC)
   → [12345, 12346, …]                    ~1 round trip, cheap, no field cost
   → chunk(200) → POST workitemsbatch     ~1 round trip per 200 items
   → normalize → upsert into SQLite
```

Why not `GET /wit/workitems?ids=…&$expand=all`? Because `$expand=all` on 200 items pulls relations and `_links` for every one — hundreds of KB we discard. The Inbox only needs the summary field set; **relations and description HTML are a Phase-3 concern**, fetched per-item on demand at run creation. The one exception: `System.Description` *is* pulled in the batch, because the `brief` and the `has_appinsights_link` flag are inbox-visible and computing them lazily makes the list flicker.

Batch field set (exact):

```
System.Id, System.Title, System.State, System.WorkItemType, System.AssignedTo,
System.Tags, System.ChangedDate, System.CreatedDate, System.TeamProject,
System.Description, System.AreaPath, System.IterationPath,
Microsoft.VSTS.Common.Priority, Microsoft.VSTS.Common.Severity
```

`Microsoft.VSTS.TCM.ReproSteps` is deliberately *not* in the batch — it is frequently 50–200 KB of HTML per bug and only Phase 3 needs it. The `has_appinsights_link` flag computed from `System.Description` alone can therefore be a false negative; the flag is recomputed (and the row updated) at harvest time. The UI treats it as a hint, not a promise.

**State categories.** ADO state names are process-template-dependent (`New/Active/Resolved/Closed` for Agile, `Proposed/Active/Resolved/Closed` for CMMI, `New/Committed/Done` for Scrum, plus arbitrary custom states). Hardcoding colours by name is wrong. LazyBoy fetches the per-project, per-type state metadata once per day and caches the *category*:

```http
GET https://dev.azure.com/{org}/{project}/_apis/wit/workitemtypes/{type}/states?api-version=7.1
→ [{"name":"Active","color":"007acc","category":"InProgress"}, …]
```

Categories are a closed set: `Proposed`, `InProgress`, `Resolved`, `Completed`, `Removed`. That is what drives pill colour. If the metadata call fails, fall back to a name-heuristic map and mark `state_category = "Unknown"` (grey pill).

### 2.3 Caching & sync strategy

SQLite is the source of truth for the UI; ADO is the source of truth for reality. The gap is closed by a watermark.

**No type clause, ever.** Neither the full nor the incremental WIQL filters `[System.WorkItemType]`. The only predicates are `@Me`, the excluded-states list, and the watermark. Type is a *display and ordering* concern handled client-side from the cached `System.WorkItemType` field; it is a filter chip the user can apply, never a filter the sync applies on their behalf. A regression test asserts the generated WIQL string contains no `WorkItemType`.

**Incremental sync.** Persist `sync_state(scope_key, watermark_utc, last_full_sync, last_error)` — one row, because there is one org and one project. The incremental WIQL is the assigned-to-me query plus a watermark clause, POSTed to the project-scoped `wiql` route (no `System.TeamProject` clause is needed — the route already constrains it):

```sql
SELECT [System.Id] FROM WorkItems
WHERE [System.AssignedTo] = @Me
  AND [System.State] NOT IN ('Closed','Removed','Done')
  AND [System.ChangedDate] >= '2026-08-19T08:31:00.0000000Z'
ORDER BY [System.ChangedDate] DESC
```

WIQL datetime literals are quoted ISO-8601; ADO compares in UTC. **Subtract a 2-minute skew margin** from the stored watermark before substituting it — ADO's `ChangedDate` is set by the server at commit time and index visibility lags marginally; a hard `>=` on the exact previous max occasionally drops an item. Re-fetching a handful of unchanged items is free (they hash-compare equal and count as `unchanged`).

The watermark is advanced to `max(System.ChangedDate)` of the *returned* rows, not to `now()` — this is safe under clock skew in either direction.

**Deletions and un-assignments.** An item that leaves the query (reassigned, closed) will never appear in an incremental result, so it would linger forever. Two mitigations:
- Every incremental sync also runs the *id-only* full WIQL (no watermark). Ids-only is one cheap call regardless of result size; diffing the returned id set against the cached id set gives departures for free. Departed items are soft-marked `in_inbox = 0` rather than deleted, so run history keeps its foreign key.
- A full hydration (batch call) is forced when `now - last_full_sync > 24 h`, or on explicit "Force full refresh".

**Background refresh.** A single `asyncio` task started at app startup:

```
interval = config.inbox.refresh_seconds (default 300)
backoff on error: 300 → 600 → 1200 → 1800 (cap), reset on success
skipped entirely when: no credential, or the browser has had no SSE client for > 30 min (idle laptop = no reason to burn tokens)
```

The task publishes an `inbox.synced` event on the EventBus; the SPA listens on the existing SSE channel and calls `queryClient.invalidateQueries({queryKey:['workitems']})`. No polling from the browser.

**Per-item conditional refresh.** When the Bug Workspace opens an item, `get_work_item(id, etag=cached)` sends `If-None-Match`. A `304` short-circuits to cache. ADO returns strong ETags on work item GETs; if the header is absent for a given org (older on-prem), the code degrades silently to unconditional GET.

**Cache invalidation table**

| Trigger | Action |
|---|---|
| Background timer (5 min) | incremental sync |
| Manual "Refresh" button | incremental sync, `force_full=false` |
| Shift-click / "Force full" | full WIQL + full batch hydration, resets watermark |
| Run created on item | conditional refresh of that item only |
| LazyBoy posts a comment/tag (Phase 6) | conditional refresh of that item only |
| App startup | incremental sync if `now - watermark > refresh_seconds`, else serve cache immediately |

### 2.4 HTML → brief description

`System.Description` is ADO rich-text HTML: `<div>`, `<br>`, `<img src="…/_apis/wit/attachments/{guid}">`, `<a href>`, tables, `&nbsp;` soup, and occasionally pasted Word markup with `<o:p>` and inline styles.

Rules for the inbox `brief`:

1. Parse with `selectolax` (`HTMLParser`) — 5–10× faster than BeautifulSoup and tolerant of the malformed fragments ADO stores. `lxml` is the fallback if selectolax is unavailable on the platform wheel.
2. Strip `<script>`, `<style>`, `<head>`, comments.
3. Convert block boundaries (`div, p, br, li, tr, h1-h6`) to `\n`; `li` gets a `- ` prefix.
4. **Preserve links as text**: `<a href="X">Y</a>` → `Y (X)` when `Y != X`, else just `X`. This is what makes the App Insights URL survive into the plain-text brief, which matters because the agent reads the plain text.
5. `<img>` → `[image: filename]`, filename taken from the `fileName` query param or the `alt` attribute.
6. Unescape entities, collapse runs of whitespace, collapse ≥3 newlines to 2, strip.
7. Truncate to 280 chars **on a word boundary**, append `…`. Store the full plain text as `description_text` too — Phase 3 reuses it and re-deriving it is wasteful.

The full-fidelity sanitised HTML (allowlist: `p,br,div,span,b,strong,i,em,u,code,pre,ul,ol,li,a,img,table,thead,tbody,tr,td,th,h1..h6,blockquote`; allowed attrs `href,src,alt,title,colspan,rowspan`; `href` schemes limited to `http/https/mailto`) is kept for the Bug Workspace's rendered view — computed in Phase 3, not here.

### 2.5 URL parsing

Work-item URLs in the wild take at least eight shapes. One regex ladder, ordered by specificity, returning `(org, project|None, id)`:

| Shape | Example |
|---|---|
| Edit route | `https://dev.azure.com/{org}/{project}/_workitems/edit/12345` |
| Edit route, no project | `https://dev.azure.com/{org}/_workitems/edit/12345` |
| Query param | `…/_boards/board/t/TeamName/Stories/?workitem=12345` |
| Query param alt | `…/_backlogs/backlog/Team/Stories/?workitem=12345&_a=…` |
| Sprint/taskboard | `…/_sprints/taskboard/Team/Proj/Iteration?workitem=12345` |
| Legacy VSTS host | `https://{org}.visualstudio.com/{project}/_workitems/edit/12345` |
| On-prem TFS | `https://tfs.corp/tfs/{collection}/{project}/_workitems/edit/12345` |
| API url | `https://dev.azure.com/{org}/_apis/wit/workItems/12345` |
| Short link | `https://dev.azure.com/{org}/_workitems/edit/12345?src=…` or a `aka.ms`/`bit.ly` wrapper |
| Bare | `12345`, `#12345`, `AB#12345`, `Bug 12345` |

Short links that are not ADO hosts (`aka.ms`, `bit.ly`, `t.co`, `lnkd.in`) are resolved with a single `HEAD` following redirects, max 3 hops, 5 s timeout, and only if the final host matches an *allowlisted* ADO host pattern — otherwise rejected. This is deliberate: a paste box that follows arbitrary URLs is an SSRF hole in a tool that holds your tokens.

The parser still returns `(org, project|None, id)` — the shapes in the wild are unchanged and the fixture set is worth keeping. What changed is what happens next: `org`/`project` parsed out of a URL are **validated against the configured pair**, not used to route. A mismatch is a 422 (`"That work item is in project Billing; LazyBoy is configured for Portal."`), and a bare id resolves against the single configured project with no disambiguation dialog and no parallel fan-out.

### 2.6 Scope configuration

`config.yaml`:

```yaml
ado:
  org: zeal
  url: https://dev.azure.com/zeal
  project: Portal
  auth: sso               # sso | pat
  scope: project          # project | org — v1 is always `project`
inbox:
  refresh_seconds: 300
  page_size: 50
  default_states_excluded: [Closed, Removed, Done, Cut]
```

One `AdoClient`, one sync, one result set. Work item primary key in SQLite is the bare ADO `id` — unique within a project, and there is only one. `System.TeamProject` is still carried through the batch field set and stored (it costs nothing and keeps the DTO stable if `scope` ever flips), but it is not a filter facet, not a column, and not part of any key.

`scope: org` is the forward door: it drops the project segment from the WIQL and `workitemsbatch` routes and re-enables the project facet. It is designed, not built — nothing in v1 exercises it, and no other code branches on it.

---

## Code

### 3.1 `connectors/ado.py` (excerpt)

```python
from __future__ import annotations

import asyncio
import base64
import logging
from collections.abc import AsyncIterator
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta
from typing import Any

import httpx
from tenacity import (
    AsyncRetrying, retry_if_exception, stop_after_attempt,
    wait_exponential_jitter, RetryCallState,
)

log = logging.getLogger("lazyboy.ado")

API = "7.1"
API_COMMENTS = "7.1-preview.4"
BATCH_MAX = 200


class AdoError(RuntimeError):
    def __init__(self, status: int, message: str, url: str = "") -> None:
        super().__init__(f"ADO {status}: {message}")
        self.status, self.message, self.url = status, message, url


class AdoAuthError(AdoError):
    """401/403 — surfaces to the UI as 'reconnect', never retried."""


# AdoAuth comes from Phase 1 (phases/phase-1-identity.md §7) — it owns header
# acquisition (PAT basic vs Entra bearer), the single silent 401 retry, and
# raising ReauthRequired. Phase 2 does not redefine it and does not handle 401
# retry itself; it only translates the resulting exception at the API boundary.
from lazyboy.connectors.credentials import AdoAuth, ReauthRequired


def _retryable(exc: BaseException) -> bool:
    if isinstance(exc, httpx.TransportError):
        return True
    return isinstance(exc, AdoError) and exc.status in (429, 500, 502, 503, 504)


@dataclass(slots=True)
class WorkItemSummary:
    id: int
    project: str            # constant in v1; kept so `ado.scope: org` stays a config change
    title: str
    state: str
    state_category: str
    type: str                # verbatim ADO type; no type filter is applied anywhere
    type_category: str       # "bug" | "other" — Bugs-first sort key
    fix_support: str         # "full" | "best_effort"
    assigned_to: str | None
    assigned_to_email: str | None
    tags: list[str]
    priority: int | None
    severity: str | None
    changed_date: datetime
    created_date: datetime
    area_path: str | None
    iteration_path: str | None
    description_html: str | None
    url: str
    etag: str | None = None


class AdoClient:
    """One instance for the configured org+project, created at startup, shared by all stages."""

    def __init__(
        self,
        org: str,
        project: str,
        base_url: str,
        auth: AdoAuth,                 # constructed by Phase 1's AuthContext
        *,
        scope: str = "project",        # "project" | "org"; v1 is always "project"
        max_concurrency: int = 8,
        timeout: float = 30.0,
    ) -> None:
        self.org, self.project, self.scope = org, project, scope
        self.base = base_url.rstrip("/")
        self._sem = asyncio.Semaphore(max_concurrency)
        self._http = httpx.AsyncClient(
            base_url=self.base,
            auth=auth,
            http2=True,
            timeout=httpx.Timeout(timeout, connect=10.0),
            limits=httpx.Limits(max_keepalive_connections=20, max_connections=40),
            headers={"Accept": "application/json", "User-Agent": "LazyBoy/0.1"},
            follow_redirects=False,   # an ADO redirect means auth trouble, not a move
        )

    async def aclose(self) -> None:
        await self._http.aclose()

    # ---------------------------------------------------------------- plumbing

    def _url(self, path: str, *, org_scope: bool = False) -> str:
        """Project-scoped by default. `org_scope=True` exists only for ado.scope: org."""
        seg = "" if (org_scope or self.scope == "org") else f"/{self.project}"
        return f"{seg}/_apis/{path.lstrip('/')}"

    async def _request(
        self,
        method: str,
        path: str,
        *,
        api_version: str = API,
        params: dict[str, Any] | None = None,
        json_body: Any = None,
        content: bytes | None = None,
        headers: dict[str, str] | None = None,
        expect_304: bool = False,
    ) -> httpx.Response:
        params = {**(params or {}), "api-version": api_version}
        url = self._url(path)

        async for attempt in AsyncRetrying(
            retry=retry_if_exception(_retryable),
            stop=stop_after_attempt(5),
            wait=self._wait,
            reraise=True,
        ):
            with attempt:
                async with self._sem:
                    resp = await self._http.request(
                        method, url, params=params, json=json_body,
                        content=content, headers=headers,
                    )
                if resp.status_code == 304 and expect_304:
                    return resp
                if resp.status_code in (401, 403):
                    # AdoAuth already did its one silent refresh-and-retry and
                    # re-raised as ReauthRequired if that failed; reaching here
                    # means genuinely insufficient permission, not a stale token.
                    raise AdoAuthError(resp.status_code, _msg(resp), url)
                if resp.status_code >= 400:
                    err = AdoError(resp.status_code, _msg(resp), url)
                    err.retry_after = _retry_after(resp)   # consumed by _wait
                    raise err
                return resp
        raise AssertionError("unreachable")

    @staticmethod
    def _wait(state: RetryCallState) -> float:
        exc = state.outcome.exception() if state.outcome else None
        after = getattr(exc, "retry_after", None)
        if after is not None:
            # ADO's Retry-After is authoritative; +250 ms so we don't race the window edge.
            return min(float(after) + 0.25, 60.0)
        return wait_exponential_jitter(initial=0.5, max=20.0, jitter=1.0)(state)

    async def _paginate(
        self, path: str, *, params: dict | None = None
    ) -> AsyncIterator[dict]:
        """Continuation-token pagination (repos, refs, comments)."""
        token: str | None = None
        while True:
            p = dict(params or {})
            if token:
                p["continuationToken"] = token
            resp = await self._request("GET", path, params=p)
            body = resp.json()
            for item in body.get("value", body.get("comments", [])):
                yield item
            token = resp.headers.get("x-ms-continuationtoken")
            if not token:
                return

    # ------------------------------------------------------------- work items

    async def wiql_ids(
        self,
        *,
        assigned_to_me: bool = True,
        excluded_states: list[str] | None = None,
        changed_since: datetime | None = None,
        top: int = 500,
    ) -> list[int]:
        """Project-scoped WIQL returning ids only. One round trip, no field cost.

        No [System.TeamProject] clause: the route segment already scopes it.
        """
        clauses: list[str] = []
        if assigned_to_me:
            clauses.append("[System.AssignedTo] = @Me")
        if excluded_states:
            joined = ",".join(f"'{s}'" for s in excluded_states)
            clauses.append(f"[System.State] NOT IN ({joined})")
        if changed_since:
            # 2-minute skew margin: ADO index visibility lags server commit time.
            watermark = (changed_since - timedelta(minutes=2)).astimezone(UTC)
            stamp = watermark.strftime("%Y-%m-%dT%H:%M:%S.0000000Z")
            clauses.append(f"[System.ChangedDate] >= '{stamp}'")

        query = (
            "SELECT [System.Id] FROM WorkItems"
            + (" WHERE " + " AND ".join(clauses) if clauses else "")
            + " ORDER BY [System.ChangedDate] DESC"
        )
        resp = await self._request(
            "POST", "wit/wiql", json_body={"query": query}, params={"$top": top}
        )
        return [w["id"] for w in resp.json().get("workItems", [])]

    BATCH_FIELDS = [
        "System.Id", "System.Title", "System.State", "System.WorkItemType",
        "System.AssignedTo", "System.Tags", "System.ChangedDate",
        "System.CreatedDate", "System.TeamProject", "System.Description",
        "System.AreaPath", "System.IterationPath",
        "Microsoft.VSTS.Common.Priority", "Microsoft.VSTS.Common.Severity",
    ]

    async def work_items_batch(self, ids: list[int]) -> list[dict]:
        """Hydrate ids in chunks of 200, concurrently, order-preserving."""
        chunks = [ids[i:i + BATCH_MAX] for i in range(0, len(ids), BATCH_MAX)]

        async def one(chunk: list[int]) -> list[dict]:
            resp = await self._request(
                "POST", "wit/workitemsbatch",
                json_body={
                    "ids": chunk,
                    "fields": self.BATCH_FIELDS,
                    "errorPolicy": "omit",   # a deleted id must not 404 the whole batch
                },
            )
            return resp.json().get("value", [])

        results = await asyncio.gather(*(one(c) for c in chunks))
        return [wi for part in results for wi in part]

    async def get_work_item(
        self, wid: int, *, expand: str = "all", etag: str | None = None
    ) -> tuple[dict | None, str | None]:
        """Conditional full fetch. Returns (item|None, etag); None item == 304."""
        headers = {"If-None-Match": etag} if etag else None
        resp = await self._request(
            "GET", f"wit/workitems/{wid}",
            params={"$expand": expand}, headers=headers, expect_304=True,
        )
        if resp.status_code == 304:
            return None, etag
        return resp.json(), resp.headers.get("ETag")

    async def work_item_type_states(self, wit_type: str) -> list[dict]:
        resp = await self._request(
            "GET", f"wit/workitemtypes/{wit_type}/states"
        )
        return resp.json().get("value", [])


def _msg(resp: httpx.Response) -> str:
    try:
        return resp.json().get("message") or resp.text[:400]
    except Exception:
        return resp.text[:400]


def _retry_after(resp: httpx.Response) -> float | None:
    for h in ("Retry-After", "X-RateLimit-Reset-After", "x-ms-retry-after-ms"):
        v = resp.headers.get(h)
        if v:
            try:
                return float(v) / 1000.0 if h.endswith("-ms") else float(v)
            except ValueError:
                continue
    return None
```

### 3.2 HTML → brief

```python
import html as htmllib
import re
from selectolax.parser import HTMLParser

_BLOCK = {"div", "p", "br", "li", "tr", "h1", "h2", "h3", "h4", "h5", "h6", "blockquote"}
_WS = re.compile(r"[ \t ]+")
_NL = re.compile(r"\n{3,}")

AI_PORTAL = re.compile(
    r"https?://portal\.azure\.com/[^\s\"'<>)\]]*"
    r"(?:AppInsightsExtension|DetailsV2Blade|applicationinsights)[^\s\"'<>)\]]*",
    re.I,
)


def html_to_text(raw: str | None) -> str:
    """ADO rich text -> plain text, preserving link targets and image names."""
    if not raw:
        return ""
    tree = HTMLParser(raw)
    for tag in ("script", "style", "head"):
        for node in tree.css(tag):
            node.decompose()

    out: list[str] = []
    for node in tree.root.iter(include_text=True) if tree.root else []:
        tag = node.tag
        if tag == "-text":
            out.append(node.text(deep=False) or "")
        elif tag == "a":
            href = (node.attributes.get("href") or "").strip()
            label = (node.text(deep=True) or "").strip()
            out.append(f"{label} ({href})" if href and label and label != href else (href or label))
            node.decompose()
        elif tag == "img":
            src = node.attributes.get("src") or ""
            name = node.attributes.get("alt") or _filename_from(src) or "image"
            out.append(f"[image: {name}]")
        elif tag == "li":
            out.append("\n- ")
        elif tag in _BLOCK:
            out.append("\n")

    text = htmllib.unescape("".join(out))
    text = _WS.sub(" ", text).replace(" \n", "\n")
    return _NL.sub("\n\n", text).strip()


def brief(text: str, limit: int = 280) -> str:
    text = " ".join(text.split())
    if len(text) <= limit:
        return text
    cut = text[:limit].rsplit(" ", 1)[0]
    return f"{cut}…"


def _filename_from(src: str) -> str | None:
    m = re.search(r"[?&]fileName=([^&]+)", src)
    return m.group(1) if m else None
```

### 3.3 URL / id parsing

```python
import re
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class WorkItemRef:
    org: str | None
    project: str | None
    id: int

_HOST_DEVAZURE = r"dev\.azure\.com/(?P<org>[^/?#]+)"
_HOST_VSTS = r"(?P<org>[^./]+)\.visualstudio\.com"
_HOST = rf"(?:{_HOST_DEVAZURE}|{_HOST_VSTS})"

PATTERNS: list[re.Pattern[str]] = [
    # .../{project}/_workitems/edit/123  |  .../_workitems/edit/123
    re.compile(rf"https?://{_HOST}(?:/(?P<project>(?!_)[^/?#]+))?/_workitems/edit/(?P<id>\d+)", re.I),
    # any board/backlog/sprint/query route carrying ?workitem=123
    re.compile(rf"https?://{_HOST}(?:/(?P<project>(?!_)[^/?#]+))?/_(?:boards|backlogs|sprints|queries|dashboards)/[^?#]*[?&]workitem=(?P<id>\d+)", re.I),
    # REST url
    re.compile(rf"https?://{_HOST}(?:/(?P<project>(?!_)[^/?#]+))?/_apis/wit/workitems/(?P<id>\d+)", re.I),
    # on-prem TFS collection
    re.compile(r"https?://[^/]+/tfs/(?P<org>[^/]+)/(?P<project>[^/?#]+)/_workitems/edit/(?P<id>\d+)", re.I),
    # generic fallback: any ADO-ish host with a trailing /123
    re.compile(rf"https?://{_HOST}/.*?/(?P<id>\d+)(?:[?#]|$)", re.I),
]

BARE = re.compile(r"^\s*(?:AB#|#|bug\s+|work\s*item\s+)?(?P<id>\d{1,9})\s*$", re.I)

SHORTENERS = {"aka.ms", "bit.ly", "t.co", "lnkd.in", "tinyurl.com", "go.microsoft.com"}


def parse_work_item_ref(s: str) -> WorkItemRef | None:
    s = s.strip()
    if (m := BARE.match(s)):
        return WorkItemRef(None, None, int(m.group("id")))
    for pat in PATTERNS:
        if (m := pat.search(s)):
            g = m.groupdict()
            project = g.get("project")
            if project and project.startswith("_"):
                project = None
            return WorkItemRef(g.get("org"), project, int(g["id"]))
    return None


async def resolve_short_link(url: str, http: httpx.AsyncClient) -> str | None:
    """Follow at most 3 hops; refuse anything that doesn't land on an ADO host.

    A paste box that follows arbitrary redirects is an SSRF hole in a process
    that holds your tokens. This is why the allowlist is on the *destination*.
    """
    from urllib.parse import urlparse
    if urlparse(url).hostname not in SHORTENERS:
        return url
    current = url
    for _ in range(3):
        r = await http.head(current, follow_redirects=False, timeout=5.0)
        loc = r.headers.get("location")
        if not loc:
            break
        current = loc
        host = (urlparse(current).hostname or "").lower()
        if host == "dev.azure.com" or host.endswith(".visualstudio.com"):
            return current
    return None
```

---

## API

All routes under `/api`, all responses `application/json`, all mutating routes require the per-launch session token (Phase 0).

### `GET /api/workitems`

| Param | Type | Notes |
|---|---|---|
| `state` | `string[]` | repeatable; exact ADO state names |
| `category` | `string[]` | `Proposed\|InProgress\|Resolved\|Completed\|Removed` |
| `type` | `string[]` | `Bug`, `Task`, … |
| `tag` | `string[]` | AND semantics |
| `q` | `string` | matches title, id, and `description_text` (SQLite FTS5) |
| `assigned` | `me\|any` | default `me` |
| `has_appinsights` | `bool` | |
| `sort` | `smart\|changed\|created\|priority\|severity\|id\|title` | default `smart` = type rank (Bug 0, Defect/Incident 0, everything else 1) then `changed` desc. Every other value sorts on that field alone. |
| `dir` | `asc\|desc` | default `desc` |
| `cursor` | `string` | opaque `(sort_value, id)` keyset cursor |
| `limit` | `int` | default 50, max 200 |

```jsonc
{
  "items": [{
    "id": 12345, "project": "Portal",
    "title": "NullReferenceException on checkout when cart is empty",
    "type": "Bug",
    "type_category": "bug",          // "bug" | "other" — drives the badge and the Bugs-first sort
    "fix_support": "full",           // "full" for bug-like types, "best_effort" for everything else
    "state": "Active", "state_category": "InProgress", "state_color": "007acc",
    "assigned_to": {"display_name": "Francesco Colombo", "unique_name": "francesco.colombo@…", "image_url": "…"},
    "tags": ["prod", "checkout"],
    "priority": 1, "severity": "2 - High",
    "brief": "User reports a 500 on POST /api/checkout. App Insights: https://portal.azure.com/#blade/… ",
    "has_appinsights_link": true,
    "has_attachments": true,
    "attachment_count": 3,
    "changed_date": "2026-08-19T08:31:02Z",
    "created_date": "2026-08-13T19:20:11Z",
    "url": "https://dev.azure.com/zeal/Portal/_workitems/edit/12345",
    "last_run": {"id": "r_01J…", "state": "report_ready", "created_at": "2026-08-18T10:02:00Z"}
  }],
  "next_cursor": "eyJjIjoiMjAyNi0wOC0xOVQwODozMTowMloiLCJpIjoxMjM0NX0",
  "sync": {"last_synced_at": "2026-08-19T08:35:00Z", "stale": false, "last_error": null}
}
```

`last_run` is a left join onto `runs` — it powers the "already investigated" affordance and stops you re-running the same bug by accident.

### `POST /api/workitems/sync`

```jsonc
// request
{"force_full": false}
// response 200
{"fetched": 12, "updated": 3, "unchanged": 9, "departed": 1,
 "watermark": "2026-08-19T08:31:02Z", "duration_ms": 430}
```

Concurrent syncs are collapsed: the in-flight sync is held in a single future; a second caller awaits it rather than issuing a second WIQL.

### `POST /api/workitems/resolve`

```jsonc
{"input": "https://dev.azure.com/zeal/Portal/_boards/board/t/Team/Stories/?workitem=12345"}
→ 200 {"item": { …WorkItemDTO… }, "created": true}
→ 404 {"error": "not_found", "detail": "No work item 12345 in zeal/Portal."}
→ 422 {"error": "unparseable", "detail": "Could not find a work item id in that text."}
→ 422 {"error": "out_of_scope", "detail": "That URL points at zeal/Billing; LazyBoy is configured for zeal/Portal."}
```

Resolves, fetches (project-scoped conditional GET), upserts into the cache with `in_inbox = 0`, and returns the DTO. This is the manual entry path. There is no `ambiguous` outcome — one project means one candidate.

### `GET /api/workitems/{id}`

Conditional refresh + full DTO including sanitised description HTML and relation summary. `?refresh=true` forces an ADO round trip.

### `GET /api/workitems/facets`

Distinct `state`, `type`, `tag` values with counts, computed from the cache in one grouped query — feeds the filter chips without a second network hop.

### `POST /api/runs`

```jsonc
{"work_item_id": 12345, "mode": "investigate"}
→ 201 {"id": "r_01J8…", "state": "created", "work_item_id": 12345}
```

Phase 2 only creates the row and returns it; Phase 3 attaches the harvest stage. Idempotency: if a run in a non-terminal state already exists for that work item id, return `409` with that run's id plus `{"resumable": true}` and let the UI offer "Open existing" / "Start new".

---

## UI

Route `/` → `InboxPage`. Layout: sticky toolbar, virtualised list, right-hand detail drawer on selection.

### 5.1 Layout

Table on ≥ 1024 px, stacked cards below. Columns:

| Col | Width | Content |
|---|---|---|
| ▸ | 28 | Selection / expand caret |
| ID | 72 | `#12345`, monospace, links to ADO in a new tab (⌘-click safe) |
| Type | 90 | **Type badge** — ADO's own icon and type colour plus the type name (bug / task / PBI / user story / support request …). Non-Bug badges are outlined rather than filled, and carry a `best-effort fix` tooltip. Always rendered: with no type filter in the query, the badge is how you tell at a glance what you are looking at. |
| Title | flex | Title on line 1, `brief` on line 2 in `text-sm text-slate-500`, 2-line clamp |
| State | 110 | Pill, coloured by `state_category` |
| Pri/Sev | 80 | Two small badges: `P1` and `S2` |
| Signals | 90 | 📈 App Insights, 📎 attachments (with count), 🔁 prior run |
| Changed | 100 | Relative (`2h ago`), `title` attr = absolute ISO |
| | 120 | **Investigate** button (primary on hover/focus, ghost otherwise) |

There is no project (or org) column: every row is in the same project, so a column repeating `Portal` 50 times is pure noise. The org/project pair is stated once, in the app header next to the connected identity, where it belongs.

**Bugs first.** The default `smart` sort groups by `type_category` (bug-like first) and orders each group by `changed` desc, with a thin section divider and a muted `Other work items (7)` label before the second group. It is a sort, not a filter — scrolling past the divider shows everything else assigned to you, unhidden. Choosing any explicit sort column drops the grouping entirely.

**Best-effort marker on non-Bug types.** A row whose `fix_support` is `best_effort` renders a small outlined `best-effort` chip next to the Investigate button, tooltipped *"Investigation works normally. The fix stage is tuned for defects — for a Story or Task it will still propose and implement a change, but treat the Change Report as a draft."* The same marker follows the item into the Bug Workspace header and pre-fills a one-line caveat in the `start_fix` gate dialog, so the caveat appears at the moment of the decision rather than only in the list. Nothing is disabled: every type can be investigated, reported on, fixed and published.

**State pill colours** (by category, not name):

| Category | Class |
|---|---|
| `Proposed` | `bg-slate-100 text-slate-700 ring-slate-300` |
| `InProgress` | `bg-blue-50 text-blue-700 ring-blue-300` |
| `Resolved` | `bg-violet-50 text-violet-700 ring-violet-300` |
| `Completed` | `bg-emerald-50 text-emerald-700 ring-emerald-300` |
| `Removed` | `bg-zinc-100 text-zinc-500 line-through ring-zinc-300` |
| `Unknown` | `bg-amber-50 text-amber-700 ring-amber-300` |

The pill's exact ADO colour (`state_color`) is rendered as a 6 px leading dot inside the pill, so the categorical class gives semantics and the dot gives fidelity. Never rely on colour alone — the category name is the pill text.

**Priority/severity badges.** `P1` red-600, `P2` orange-500, `P3` slate-400, `P4` slate-300. Severity `1 - Critical` → `S1` with a filled badge, `2 - High` → outlined, `3/4` → muted. Both are `title`-tooltipped with the raw ADO value.

**Signals.** `has_appinsights_link` renders a small chart-line glyph with tooltip "App Insights link found in description"; it is the single strongest predictor that a run will produce something good, so it is also a one-click filter chip in the toolbar. `has_attachments` renders a paperclip with a superscript count. `last_run` renders a state-coloured dot with tooltip "Investigated 18 Aug — report ready" and clicking it jumps to that run rather than starting a new one.

### 5.2 Toolbar

`[ Search (⌘K) ]  [ State ▾ ] [ Type ▾ ] [ Tag ▾ ] [ 📈 Has AI ] [ Assigned: Me ▾ ]   … right: [ ↻ 2m ago ] [ + Add by id/URL ]`

- Filters are multi-select popovers with counts from `/facets`; active filters render as removable chips below the toolbar.
- Filter state is serialised into the URL query string, so a filtered inbox is bookmarkable and survives reload. TanStack Query key includes the parsed filter object.
- Search is debounced 200 ms, matches id / title / description text, and highlights matches in the title cell.
- The refresh control shows relative last-sync time; clicking syncs; shift-click forces full; it spins while `isFetching`; on error it turns amber with the error in a tooltip and the list keeps showing stale data (never blank the list on a sync failure).

### 5.3 Add by id or URL

A dialog (`+ Add` or `⌘⇧O`) with a single input that accepts an id, an ADO URL, or a blob of pasted text containing one. Live validation calls `parse_work_item_ref` client-side (the regex ladder is duplicated in TS — a shared JSON test-vector fixture keeps the two implementations honest) and shows a preview chip `#12345` before you commit — plus an amber "different project" warning if the pasted URL names anything other than the configured project, which is a rejection, not a prompt. Submit → `POST /api/workitems/resolve` → on success, the item is inserted at the top of the list with a "pinned / not assigned to you" marker and the Investigate button focused.

### 5.4 Keyboard

| Key | Action |
|---|---|
| `j` / `k`, `↓` / `↑` | move selection |
| `Enter` | open detail drawer |
| `i` | Investigate selected |
| `o` | open in ADO (new tab) |
| `/` or `⌘K` | focus search |
| `r` | refresh |
| `⌘⇧O` | add by id/URL |
| `Esc` | clear selection / close drawer |

Implemented with a single `useHotkeys` hook bound at page level, disabled while an input has focus. Roving `tabIndex` on rows, `aria-selected`, `role="row"`; the list is a real table with `role` overrides so screen readers announce column headers.

### 5.5 States

| State | Treatment |
|---|---|
| Loading (cold) | 8 skeleton rows with shimmer; toolbar disabled but visible |
| Loading (warm) | Existing data stays; a 2 px indeterminate bar under the toolbar |
| Empty (no items) | Illustration + "Nothing assigned to you right now." + [Refresh] + [Add by id/URL] |
| Empty (filters) | "No work items match these filters." + [Clear filters] |
| Not connected | Redirect to `/connect` (Phase 1) with a return URL |
| Auth expired | Amber banner "Your ADO session expired" + [Reconnect]; list shows stale cache, Investigate disabled |
| Sync error | Amber banner with the error message and a Retry; list shows stale cache |
| Stale (> 30 min) | Toolbar clock turns amber, tooltip "Last synced 47 minutes ago" |

### 5.6 Query hooks (`web/src/features/inbox/api.ts`)

```ts
import {
  useQuery, useMutation, useQueryClient, keepPreviousData,
  type UseQueryResult,
} from '@tanstack/react-query';
import { api } from '@/lib/api';

export interface WorkItem {
  id: number;
  project: string;          // constant in v1; not rendered as a column or facet
  title: string;
  type: string;
  typeCategory: 'bug' | 'other';
  fixSupport: 'full' | 'best_effort';
  state: string;
  stateCategory: 'Proposed' | 'InProgress' | 'Resolved' | 'Completed' | 'Removed' | 'Unknown';
  stateColor: string | null;
  assignedTo: { displayName: string; uniqueName: string; imageUrl: string | null } | null;
  tags: string[];
  priority: number | null;
  severity: string | null;
  brief: string;
  hasAppinsightsLink: boolean;
  hasAttachments: boolean;
  attachmentCount: number;
  changedDate: string;
  createdDate: string;
  url: string;
  lastRun: { id: string; state: string; createdAt: string } | null;
}

export interface InboxFilters {
  q?: string;
  state?: string[];
  category?: string[];
  type?: string[];
  tag?: string[];
  hasAppinsights?: boolean;
  assigned?: 'me' | 'any';
  sort?: 'smart' | 'changed' | 'created' | 'priority' | 'severity' | 'id' | 'title';  // default 'smart' = Bugs first, then changed
  dir?: 'asc' | 'desc';
}

export interface InboxPage {
  items: WorkItem[];
  nextCursor: string | null;
  sync: { lastSyncedAt: string | null; stale: boolean; lastError: string | null };
}

export const inboxKeys = {
  all: ['workitems'] as const,
  list: (f: InboxFilters) => ['workitems', 'list', f] as const,
  facets: () => ['workitems', 'facets'] as const,
  detail: (id: number) => ['workitems', 'detail', id] as const,
};

/** The list. `keepPreviousData` is what makes filter changes feel instant:
 *  the old rows stay on screen, dimmed, while the new query resolves. */
export function useWorkItems(filters: InboxFilters): UseQueryResult<InboxPage> {
  return useQuery({
    queryKey: inboxKeys.list(filters),
    queryFn: ({ signal }) => api.get<InboxPage>('/workitems', { params: filters, signal }),
    placeholderData: keepPreviousData,
    staleTime: 30_000,          // cache is server-side; 30 s stops refetch storms on filter toggles
    refetchOnWindowFocus: true,
  });
}

export function useFacets() {
  return useQuery({
    queryKey: inboxKeys.facets(),
    queryFn: () => api.get<Record<string, { value: string; count: number }[]>>('/workitems/facets'),
    staleTime: 5 * 60_000,
  });
}

/** Manual + background sync. Invalidate everything on settle so facets,
 *  list and the sync clock all move together. */
export function useSyncWorkItems() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (opts: { forceFull?: boolean } = {}) =>
      api.post<{ fetched: number; updated: number; departed: number; durationMs: number }>(
        '/workitems/sync', { force_full: opts.forceFull ?? false },
      ),
    onSettled: () => qc.invalidateQueries({ queryKey: inboxKeys.all }),
  });
}

/** Manual entry. Optimistically prepends the resolved item to every cached list page. */
export function useResolveWorkItem() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (input: string) => api.post<{ item: WorkItem }>('/workitems/resolve', { input }),
    onSuccess: ({ item }) => {
      qc.setQueriesData<InboxPage>({ queryKey: ['workitems', 'list'] }, (old) =>
        old && !old.items.some((w) => w.id === item.id)
          ? { ...old, items: [item, ...old.items] }
          : old,
      );
      qc.setQueryData(inboxKeys.detail(item.id), item);
    },
  });
}

/** One click from inbox row to Bug Workspace. 409 means a run already exists. */
export function useCreateRun() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (v: { workItemId: number }) =>
      api.post<{ id: string; state: string }>('/runs', {
        work_item_id: v.workItemId, mode: 'investigate',
      }),
    onSuccess: (_run, v) => {
      qc.invalidateQueries({ queryKey: inboxKeys.detail(v.workItemId) });
      qc.invalidateQueries({ queryKey: ['runs'] });
    },
  });
}

/** Server-pushed invalidation: the backend's background sync emits `inbox.synced`
 *  on the shared SSE channel, so the browser never polls. */
export function useInboxLiveRefresh() {
  const qc = useQueryClient();
  React.useEffect(() => {
    const es = new EventSource('/api/events?channel=app');
    es.addEventListener('inbox.synced', () => {
      qc.invalidateQueries({ queryKey: inboxKeys.all });
    });
    return () => es.close();
  }, [qc]);
}
```

---

## Tests

**Connector (`tests/connectors/test_ado.py`)** — `respx` mocks, no network:

| Test | Asserts |
|---|---|
| `test_wiql_builds_watermark_clause` | 2-min skew subtracted; ISO literal format exact (`.0000000Z`) |
| `test_every_route_is_project_scoped` | WIQL, batch, states, comments and attachments all carry the `/{project}` segment; no call escapes to org scope while `scope="project"` |
| `test_batch_chunks_at_200` | 451 ids → 3 POSTs, union preserved, `errorPolicy: omit` sent |
| `test_429_honours_retry_after` | mocked `Retry-After: 2` → sleep ≈ 2.25 s (patched clock), 1 retry, success |
| `test_429_x_ms_retry_after_ms` | ms header converted correctly |
| `test_401_raises_auth_error_no_retry` | exactly one request issued |
| `test_500_retries_five_times_then_raises` | attempt count == 5 |
| `test_304_returns_none_item` | conditional GET short-circuit |
| `test_continuation_token_pagination` | two pages, header consumed, no infinite loop when token repeats |
| `test_403_raises_auth_error_no_retry` | permission failure is not retried, not confused with a stale token |

**Parsing (`tests/test_urls.py`)** — a table-driven fixture `tests/fixtures/workitem-urls.json` shared with the frontend (`web/src/features/inbox/urls.test.ts` imports the same JSON). ~30 vectors covering every shape in §2.5 plus adversarial ones: `_workitems/edit/123/456`, a project literally named `_apis`, `?workitem=12345&workitem=999` (first wins), a URL inside a sentence, and non-ADO hosts (must return `None`). One extra vector asserts a URL naming a different project resolves to `out_of_scope` rather than being queried.

**HTML (`tests/test_html.py`)** — fixtures from real ADO descriptions (scrubbed): Word-paste markup, nested tables, `<img>` with `fileName` param, an App Insights URL inside an `<a href>` where the anchor text is "click here" (asserts the URL survives into the text), entity soup, 200 KB description (asserts < 20 ms).

**Sync (`tests/test_sync.py`)** — an in-memory SQLite plus a fake `AdoClient`:
- cold sync inserts N, sets watermark to max `ChangedDate`
- second sync with no changes issues 1 WIQL and 0 batch calls
- an item that leaves the WIQL is marked `in_inbox = 0`, not deleted, and its run FK survives
- concurrent `sync()` calls collapse to one WIQL (assert call count == 1)
- a 429 storm leaves `last_error` set and the cache intact

**API (`tests/api/test_workitems.py`)** — `httpx.ASGITransport` against the FastAPI app: filter combinations, keyset cursor stability across an insert, `q` FTS behaviour, `POST /runs` 409 on duplicate.

**Frontend** — Vitest + Testing Library with MSW: renders rows, applies a filter and asserts the URL query string, keyboard `j/k/i` navigation, error banner keeps stale rows, `useResolveWorkItem` optimistic prepend. Playwright smoke: load inbox → filter by "Has App Insights" → click Investigate → land on `/runs/:id`.

---

## Risks & mitigations

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | WIQL `@Me` resolves to a different identity than the PAT owner (PAT minted by a different account) | Empty or wrong inbox, very confusing | At connect time call `GET /_apis/connectionData` and display the resolved identity in the header. Sync stores the identity; a mismatch with the configured one raises a banner. |
| R2 | Custom process templates with unknown state names | Wrong pill colours, broken "exclude closed" | Category metadata is fetched per project+type and cached; `default_states_excluded` is config, not code; unknown → amber `Unknown` pill, still functional. |
| R3 | 200-item hydration trips TSTU throttling on a shared org | Slow or failed sync | Semaphore(8) + `Retry-After` + the incremental path means steady state hydrates ~0–5 items, not 200. |
| R4 | `ChangedDate` watermark misses an item (index lag / clock skew) | Silently stale row | 2-min skew margin, plus a forced full hydration every 24 h, plus the always-run id-only full WIQL for departure detection. |
| R5 | Huge descriptions (Word paste, base64 inline images) blow up the batch response and SQLite | Slow sync, fat DB | Cap stored `description_html` at 512 KB (truncate with a marker, full text fetched on demand in Phase 3); `brief` is always computed from the truncated copy — safe, since the App Insights link is virtually always near the top. |
| R6 | `has_appinsights_link` false negative because the link lives in `ReproSteps` | User doesn't see the best bugs | Documented as a hint; recomputed at harvest and the row updated; `ReproSteps` can be opted into the batch field set via config at the cost of sync size. |
| R7 | SSRF via the paste box | Token-holding process fetches attacker URLs | Only known shorteners are followed, max 3 hops, destination must match the ADO host allowlist, `HEAD` only, 5 s timeout. |
| R8 | A second project (or org) appears later and the single-scope assumption is baked in too deep | Rework across every layer | Scope lives in exactly two places: `ado.scope` in config and `AdoClient._url()`. `project` is still carried on the DTO and stored per item, so widening is a config flag plus re-enabling one facet — not a schema migration. |
| R9 | Background refresh burns tokens on an idle laptop | Rate-limit pressure, battery | Refresh suspends when no SSE client has been connected for 30 min and on `document.hidden`-driven disconnects. |
| R10 | Filter state lost on reload | Annoying | Filters live in the URL query string, which is also the TanStack Query key. |

---

## Effort

| Task | Estimate |
|---|---|
| `AdoClient` core (auth, retry, pagination, batching) | 3 h |
| Sync service + SQLModel cache + watermark logic | 2 h |
| HTML→text/brief + URL parser (+ shared fixtures) | 1.5 h |
| REST layer (`/workitems`, `/sync`, `/resolve`, `/facets`, `/runs`) | 1.5 h |
| Inbox UI (table, filters, keyboard, states) | 3 h |
| Add-by-URL dialog | 1 h |
| Tests | 2 h |

**≈ 0.5–1 day** with Claude Code doing the typing — consistent with the master plan's 0.5 day, the extra half-day being the connector work that Phases 3–8 then get for free.
