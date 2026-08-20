# Phase 6 — Review & Publish to ADO

Part of [LazyBoy Master Design](../LazyBoy-Design.md).

---

## Goal

Take an agent-authored report (`investigation.md` + `findings.json` from [Phase 5](phase-5-investigation.md), later `change-report.md` from Phase 7), let you **read it, edit it, and approve it**, then publish it to the Azure DevOps work item as a well-formed comment — with rendered images, working links, a LazyBoy signature, and exactly-once semantics.

This phase also owns the two generic mechanisms everything downstream reuses:

1. **The Gate system** — the codified form of P4 (*irreversible = approved*).
2. **The ADO publisher** — markdown → ADO's HTML subset, attachment upload, idempotent posting. Phase 8 (PR description) and the `lazyboy:needs-repo` blocked path both call into it.

Build it right once. It's the smallest phase in the plan and the most reused.

Two settled constraints from [master §8](../LazyBoy-Design.md#8-environment-constraints-answered) shape it: **one org, one project** — so `AdoClient` is already project-scoped and no call here passes a `project` argument — and **separate front-end and back-end repos**, so a report routinely covers two or more repos and the comment has to be readable when it does.

---

## Definition of Done

| # | Criterion |
|---|---|
| D1 | A `Gate` row of kind `post_report` is created when a report becomes reviewable; the run cannot post without it being `approved`. |
| D2 | No agent tool — built-in or MCP — can post to ADO. Publishing is only reachable via `POST /api/gates/{id}/decide` followed by a backend publisher call. A test asserts the MCP server exposes no write-to-ADO tool. |
| D3 | The review UI renders the markdown with syntax-highlighted code and clickable `file:line` references that open the worktree viewer or the ADO file view at the pinned sha. |
| D4 | You can edit the report inline; a diff view shows agent draft vs your edits; the edited version is what gets posted, and both versions are retained. |
| D5 | Rejecting requires a reason; the reason is stored and pre-fills the re-run instruction box for Phase 5. |
| D6 | **Dry-run/preview** shows the exact HTML payload, the rendered comment, the attachment list, and the target URL — byte-for-byte what will be sent. |
| D7 | Markdown → ADO HTML conversion handles headings, lists, code blocks, inline code, tables, links, images, blockquotes, emphasis, and horizontal rules; everything else is degraded, never dropped silently. |
| D8 | Reports over the size limit are chunked into N ordered comments with `(1/N)` markers, split at heading boundaries. |
| D9 | Screenshots are rendered headlessly with Playwright/Chromium, uploaded via the ADO attachments API, referenced from the comment HTML, **and** added as `AttachedFile` relations. |
| D10 | Posting twice — by double-click, retry, or restart — produces exactly one comment. Enforced by a deterministic marker checked against existing comments before posting. |
| D11 | The blocked path posts the `lazyboy:needs-repo` tag plus an evidence comment through the same gate + publisher. |
| D12 | Everything published is recorded in `PublishRecord` and `audit.jsonl`, with the comment id, URL, HTML sha256, and attachment ids. |

---

## Design

### 6.1 The Gate system

A gate is a *typed pending decision* that blocks a state transition. It exists because P4 says irreversible actions require an explicit click, and because "the agent asked nicely and we said yes in the prompt" is not a control.

```python
class GateKind(str, Enum):
    START_INVESTIGATION = "start_investigation"   # auto-approved by default policy
    POST_REPORT         = "post_report"
    MARK_NEEDS_REPO     = "mark_needs_repo"
    START_FIX           = "start_fix"             # Phase 7
    COMMIT              = "commit"                # Phase 8
    CREATE_PR           = "create_pr"             # Phase 8

class Gate(SQLModel, table=True):
    id: str = Field(primary_key=True)
    run_id: str = Field(foreign_key="run.id", index=True)
    kind: GateKind
    state: Literal["pending", "approved", "rejected", "expired", "superseded"] = "pending"
    payload: dict = Field(sa_column=Column(JSON))   # everything needed to execute
    payload_sha: str                                # sha256 of payload — approval binds to content
    preview: dict | None = Field(sa_column=Column(JSON))
    reason: str | None = None                       # rejection reason / approval note
    created_at: datetime
    decided_at: datetime | None = None
    decided_by: str = "local-user"
    idempotency_key: str                            # carried into the publisher
```

`GateKeeper` (the same class that owns `can_use_tool` in Phase 5 — one security object, two surfaces):

```python
class GateKeeper:
    async def open(self, run: Run, kind: GateKind, payload: dict,
                   preview: dict | None = None) -> Gate:
        await self.supersede_pending(run.id, kind)          # only one live gate per kind
        gate = Gate(
            id=new_id("g"), run_id=run.id, kind=kind, payload=payload,
            payload_sha=sha256_json(payload), preview=preview,
            idempotency_key=self.idempotency_key(run, kind, payload),
            created_at=utcnow(),
        )
        await self.db.insert(gate)
        await self.bus.publish(run.id, "gate.opened", gate.public())
        return gate

    async def decide(self, gate_id: str, approve: bool, *,
                     reason: str | None = None, payload_sha: str) -> Gate:
        gate = await self.db.get(Gate, gate_id)
        if gate.state != "pending":
            raise Conflict(f"gate already {gate.state}")
        if payload_sha != gate.payload_sha:
            # the report was edited after the UI loaded the gate
            raise Conflict("gate payload changed; reload and re-approve")
        if not approve and not reason:
            raise BadRequest("rejection requires a reason")
        gate.state = "approved" if approve else "rejected"
        gate.reason, gate.decided_at = reason, utcnow()
        await self.db.save(gate)
        await self.bus.publish(gate.run_id, "gate.decided", gate.public())
        return gate

    def idempotency_key(self, run: Run, kind: GateKind, payload: dict) -> str:
        return f"lazyboy:{run.work_item_id}:{kind.value}:{payload['content_sha'][:16]}"
```

Three properties worth naming:

- **Approval binds to content.** `payload_sha` is checked at decision time; editing the report invalidates a stale approval instead of silently posting the old text.
- **One live gate per kind per run.** Re-opening supersedes; the UI never shows two competing approvals.
- **The gate carries the idempotency key.** The key is derived from work item + kind + content hash, so a retry after a network blip cannot double-post, and a *genuinely different* report gets a different key and posts as a new comment.

Why irreversible actions are backend endpoints rather than agent tools, stated once so it's not relitigated: an agent tool is reachable from any token the model emits, and the model's input contains untrusted text. An HTTP endpoint gated on a `Gate` row that you flipped in a browser is reachable only from a human click. The blast radius of a prompt injection collapses from "posts to your customer's work item" to "writes a paragraph you'll read".

### 6.2 Report review

`Report` gains two bodies: `body_draft` (agent output, immutable) and `body_final` (yours, defaults to a copy). Publishing always uses `body_final`. Both are kept — the diff between them is the honest measure of how good the agent is, and after twenty runs it tells you what to fix in the prompt.

Rejection is a first-class outcome: `reason` is stored on the gate, appended to `Run.review_notes`, and offered as the pre-filled `extra_instructions` for a Phase 5 re-run. That closes the loop without you retyping "you missed the retry handler in `StockRepo`".

### 6.3 Markdown → ADO HTML

ADO work item comments are HTML and are sanitised server-side. Anything outside the accepted subset is stripped — **silently**, which is the trap. LazyBoy converts deliberately and validates before sending.

**Allowed tags** (the set LazyBoy emits; anything else is converted or dropped by our own sanitiser, so what you preview is what survives ADO):

```
div, p, br, hr,
h3, h4, h5,                      -- h1/h2 are demoted; ADO's comment CSS makes them absurd
strong, b, em, i, u, s, code, pre, blockquote,
ul, ol, li,
a[href, title],
img[src, alt, width, height],
table, thead, tbody, tr, th, td,
details, summary,                -- collapsible; supported in modern ADO comment rendering
span[style]                      -- style limited to color/background-color/font-family
```

**Conversion rules:**

| Markdown | HTML emitted | Notes |
|---|---|---|
| `# ` / `## ` | `<h3>` | Demoted. Deeper levels shift accordingly; `#####+` becomes `<p><strong>`. |
| Fenced code block | `<pre><code>…</code></pre>` wrapped in `<div style="…monospace…">` | ADO renders no syntax highlighting. We do **not** ship inline-styled token spans — they look worse than plain and blow the size budget. Language is emitted as a small `<em>` caption above the block. |
| Inline code | `<code>` | |
| Table | `<table><thead>…` with inline `style` for borders | ADO's default table CSS has no borders; without inline styles a findings table is unreadable. Widths are omitted — let it flow. |
| Link | `<a href="…">` | `http`/`https`/`mailto` only. Relative links are resolved against the ADO web UI base. `rel`/`target` are stripped by ADO; don't bother. |
| Image | `<img src="{attachment url}" alt="…" width="…">` | Only URLs we uploaded, or ADO attachment URLs. Anything else is converted to a link (external images break for readers behind different auth). |
| Blockquote | `<blockquote>` | |
| Task list `- [x]` | `<li>✅ …</li>` | ADO strips `<input>`. |
| `---` | `<hr>` | |
| Footnotes, definition lists, HTML passthrough, mermaid | Rendered to an image (see §6.5) or degraded to a fenced code block with a warning event | Never dropped silently. |
| `file.cs:142` | `<a href="{ado file url}&line=142">file.cs:142</a>` | Linkified against the run's repo/sha index — this is the single most useful thing in the whole comment. |

Implementation: `markdown-it-py` with a custom renderer (not `markdown` + regex post-processing — the renderer hook is where the demotion and linkification belong), then `bleach.clean` against the allow-list above as a final gate, then a validation pass that asserts nothing was removed by the sanitiser. If bleach strips anything, that's a bug in our renderer and we emit `publish.sanitiser_stripped` with the delta rather than shipping mystery output.

### 6.4 Comment template

```html
<div>
  <!-- lazyboy:marker:v1 {"key":"lazyboy:12345:post_report:9f3c1ab27de4","part":"1/1","run":"r_01J…"} -->
  <p><strong>🛋️ LazyBoy — Investigation Report</strong></p>
  <p><em>Outcome:</em> <strong>Root cause identified</strong> ·
     <em>Confidence:</em> 0.82 · <em>Repos (2):</em> <code>Checkout.Api</code> (.NET),
     <code>checkout-web</code> (TS)</p>
  <hr>
  … converted report body: shared sections first (Summary, Root cause, Impact) …
  <hr>
  <h3>Checkout.Api <sub>.NET · base <code>release/2026.08</code> @ <code>a1b2c3d</code></sub></h3>
  … findings, affected files and proposed changes for this repo only …
  <h3>checkout-web <sub>TypeScript · base <code>main</code> @ <code>9f81e42</code></sub></h3>
  … findings, affected files and proposed changes for this repo only …
  <hr>
  <details>
    <summary>Evidence &amp; methodology (12 findings, 41 files inspected)</summary>
    … findings table, files-touched list, KQL used, worktree shas …
  </details>
  <p><sub>Generated by LazyBoy for francesco.colombo@zealitconsultants.ai ·
     run <code>r_01J…</code> · claude-opus-4-6 · 2026-08-19 10:31 UTC ·
     reviewed and edited by a human before posting.</sub></p>
</div>
```

The **marker comment** is the idempotency mechanism (§6.7) and is invisible in the ADO UI. The **details block** keeps the comment scannable: the top third is the answer, the rest is available on demand. The **footer** is a courtesy to whoever else reads the work item — it says a machine wrote it and a human approved it, which is exactly true.

**Grouping by repo.** Because the back end and the front end live in separate repositories ([master §8](../LazyBoy-Design.md#8-environment-constraints-answered) constraint 3), a full-stack report routinely covers two or more of them, and an ungrouped list of `path:line` references is unreadable to the person who owns only one. So the converter emits **repo-agnostic sections once** (Summary, Root cause, Impact, Open questions) and then **one `<h3>` block per repo**, in resolver-confidence order, each carrying that repo's stack, its base branch and pinned sha, and only its own findings, affected files and proposed changes. Every code reference inside a block is linkified against *that* repo's sha index (§6.3), which is what makes the links correct rather than plausible. Single-repo runs still go through the same code path — one block, and the header reads `Repos (1)` — so there is no second template to keep in sync.

For a **Change Report** the same grouping applies, plus a per-repo verification row (restore / build / typecheck) and, at the very top of the comment, the unmissable **⚠ Unverified — compiled only, tests not run** banner carried through from the change-report template ([agent-contracts §6.2](../reference/agent-contracts.md)). It is rendered as a `<p><strong>` line, not a `<details>`, because a reader who skims must not miss it.

### 6.5 Images

Some things are worth a picture: a findings table with confidence bars, a call-path diagram, a syntax-highlighted diff hunk, a mermaid sequence of the failing transaction. ADO renders none of them well as HTML. So we render them locally and upload PNGs.

Pipeline:

```
report section / diff / mermaid
      │  Jinja → self-contained HTML (inline CSS, inline fonts, no network)
      ▼
Playwright Chromium (headless, local)
      │  page.set_content(html); locator("#shot").screenshot(scale="css")
      ▼  device_scale_factor=2 → crisp on retina
runs/<id>/publish/<name>.png   (also kept for the preview UI)
      │
      ▼  POST /_apis/wit/attachments?fileName=…&api-version=7.1
{ id, url }
      │
      ├─► <img src="{url}" alt="…" width="720"> in the comment HTML
      └─► JSON-Patch: add relation { rel: "AttachedFile", url, attributes:{comment:"LazyBoy: findings table"} }
```

```python
class ShotRenderer:
    def __init__(self, run: Run):
        self.dir = run.dir / "publish"; self.dir.mkdir(exist_ok=True)

    async def render(self, html: str, name: str, width: int = 760) -> Path:
        out = self.dir / f"{name}.png"
        async with async_playwright() as pw:
            browser = await pw.chromium.launch(args=["--no-sandbox", "--disable-dev-shm-usage"])
            page = await browser.new_page(viewport={"width": width, "height": 600},
                                          device_scale_factor=2)
            await page.set_content(html, wait_until="load")   # no networkidle: assets are inline
            await page.wait_for_timeout(120)                  # font/layout settle
            el = page.locator("#shot")
            await (el if await el.count() else page).screenshot(path=out, scale="css")
            await browser.close()
        if out.stat().st_size > MAX_IMAGE_BYTES:              # 4 MB
            _downscale(out, MAX_IMAGE_BYTES)
        return out
```

Rules that keep this from becoming a liability:

- **Inline everything.** The HTML template embeds CSS and base64 fonts; no `networkidle` wait, no external fetches, no way for the renderer to leak or hang on a network call.
- **Images are additive, never load-bearing.** Every image has an HTML equivalent above or below it (the findings table exists as a real `<table>` too). If Chromium is missing or the upload fails, we emit `publish.image_skipped` and post the text-only comment — publishing never fails because a screenshot didn't render.
- **Both `<img>` and `AttachedFile`.** The inline image is for reading; the relation makes it discoverable in the work item's Attachments tab and survives comment edits.
- **Cap:** 6 images per comment, 4 MB each. Beyond that, attach only.

### 6.6 The blocked path (`lazyboy:needs-repo`)

Same gate, same publisher, different payload. When Phase 4 lands a run in `blocked_no_repo`, LazyBoy opens a `MARK_NEEDS_REPO` gate whose payload contains a comment listing *what it did find* (stack frames, `cloud_RoleName`, assemblies, catalog near-misses with scores) and *what it needs* (a repo name or URL). On approval:

1. `GET` the work item, read `System.Tags`, append `lazyboy:needs-repo` if absent (tags are a `;`-separated string — read-modify-write, and use the `rev` field for optimistic concurrency; on a 409 re-read and retry once).
2. `PATCH` with the tag change.
3. Post the evidence comment through the normal publisher (marker, footer, idempotency).

The tag is what makes the work item findable in a query, which is the actual ask. The comment is what makes it actionable.

---

## Code

### 6.7 Idempotency

```python
MARKER_RE = re.compile(r"<!--\s*lazyboy:marker:v1\s+(\{.*?\})\s*-->", re.S)

@dataclass(frozen=True)
class Marker:
    key: str; part: str; run: str
    def render(self) -> str:
        return f"<!-- lazyboy:marker:v1 {json.dumps(asdict(self), separators=(',',':'))} -->"

class AdoCommentPublisher:
    def __init__(self, ado: AdoClient, db: Database, bus: EventBus): ...

    async def publish(self, *, run: Run, work_item_id: int,
                      html_parts: list[str], key: str,
                      attachments: list[UploadedAttachment],
                      dry_run: bool = False) -> PublishResult:
        # No `project` argument anywhere in this phase: AdoClient is constructed for the
        # one configured org+project and scopes every route itself (phase-2 §2.1).
        existing = await self.ado.list_comments(work_item_id)
        already = {m.part for c in existing if (m := _parse_marker(c.text)) and m.key == key}
        if already and len(already) == len(html_parts) and not dry_run:
            await self.bus.publish(run.id, "publish.skipped",
                                   {"reason": "already posted", "key": key})
            return PublishResult(skipped=True, comment_ids=_ids_for(existing, key))

        posted: list[PostedComment] = []
        for i, body in enumerate(html_parts, start=1):
            part = f"{i}/{len(html_parts)}"
            if part in already:
                continue                                  # resume a partial multi-part post
            marker = Marker(key=key, part=part, run=run.id)
            payload = marker.render() + body
            if dry_run:
                posted.append(PostedComment(id=None, url=None, html=payload))
                continue
            c = await self.ado.post_comment(work_item_id, payload)   # project-scoped; retries on 429/5xx
            posted.append(PostedComment(id=c["id"], url=c["url"], html=payload))
            await self.db.insert(PublishRecord(
                run_id=run.id, work_item_id=work_item_id, kind="comment",
                comment_id=c["id"], part=part, key=key,
                html_sha=sha256(payload.encode()).hexdigest(),
                attachment_ids=[a.id for a in attachments], posted_at=utcnow(),
            ))
            await self.bus.publish(run.id, "publish.comment_posted",
                                   {"comment_id": c["id"], "part": part, "url": c["url"]})
        return PublishResult(skipped=False, comments=posted)
```

The check-then-post is not atomic — ADO has no conditional comment create — but the window is milliseconds and the failure mode of a double-post is cosmetic, whereas the failure mode of *not* checking is a work item with four identical reports after a flaky network. A local `asyncio.Lock` keyed by `(work_item_id, key)` closes the in-process double-click race entirely.

### 6.8 Chunking

ADO's documented comment limit is generous but not infinite, and long HTML comments render badly regardless. LazyBoy targets **30 000 characters of HTML per comment**, hard cap 60 000.

```python
def chunk_html(sections: list[Section], limit: int = 30_000) -> list[str]:
    """Split at heading boundaries; never split a <pre> or a <table>."""
    parts, cur, size = [], [], 0
    for s in sections:                         # Section = (html, is_atomic)
        if size + len(s.html) > limit and cur:
            parts.append("".join(cur)); cur, size = [], 0
        if len(s.html) > limit and not s.is_atomic:
            for sub in _split_paragraphs(s.html, limit):
                parts.append(sub)
            continue
        if len(s.html) > limit and s.is_atomic:      # a single enormous code block
            parts.append(_truncate_pre(s.html, limit,
                         note="… truncated; full text attached as report.md"))
            continue
        cur.append(s.html); size += len(s.html)
    if cur:
        parts.append("".join(cur))
    return parts
```

When chunking triggers, LazyBoy also uploads the full `investigation.md` as an attachment and links it from part 1 — so the complete text is always one click away, whatever the HTML did.

### 6.9 Attachment upload

```python
async def upload_attachment(self, path: Path,
                            *, content_type="application/octet-stream") -> UploadedAttachment:
    # Project-scoped: AdoClient holds the one configured project (phase-2 §2.1).
    r = await self.http.post(
        f"{self.base}/{self.project}/_apis/wit/attachments",
        params={"fileName": path.name, "api-version": "7.1"},
        content=path.read_bytes(),
        headers={"Content-Type": content_type},
    )
    r.raise_for_status()
    j = r.json()
    return UploadedAttachment(id=j["id"], url=j["url"], name=path.name,
                              sha=sha256(path.read_bytes()).hexdigest())

async def link_attachment(self, work_item_id: int, att: UploadedAttachment, comment: str):
    await self.http.patch(
        f"{self.base}/_apis/wit/workitems/{work_item_id}",
        params={"api-version": "7.1"},
        headers={"Content-Type": "application/json-patch+json"},
        json=[{"op": "add", "path": "/relations/-", "value": {
            "rel": "AttachedFile", "url": att.url,
            "attributes": {"comment": comment}}}],
    )
```

Uploads are also idempotent-by-content: `PublishRecord` stores each attachment's sha, so a retry reuses the existing attachment URL instead of creating an orphan.

### 6.10 Orchestration

```python
class PublishStage:
    async def prepare(self, run: Run) -> Gate:
        report = await self.db.get_report(run.id, "investigation")
        html_parts, images = await self.builder.build(run, report.body_final,
                                                      report.findings)
        payload = {
            "work_item_id": run.work_item_id,          # no project key: one configured project
            "repos": [r.repo_name for r in run.selected_repos],   # the comment groups by these
            "content_sha": sha256(report.body_final.encode()).hexdigest(),
            "parts": len(html_parts), "images": [i.name for i in images],
        }
        preview = {"html_parts": html_parts,
                   "images": [str(i.path) for i in images],
                   "target_url": run.work_item_url}
        return await self.gates.open(run, GateKind.POST_REPORT, payload, preview)

    async def execute(self, gate: Gate) -> PublishResult:
        assert gate.state == "approved"                      # endpoint checks first; assert is a tripwire
        run = await self.db.get(Run, gate.run_id)
        uploaded = [await self.ado.upload_attachment(p)      # project-scoped client
                    for p in self._image_paths(gate)]
        result = await self.publisher.publish(
            run=run, work_item_id=gate.payload["work_item_id"],
            html_parts=await self._rebuild_parts(run, gate), key=gate.idempotency_key,
            attachments=uploaded)
        for a in uploaded:
            await self.ado.link_attachment(gate.payload["work_item_id"], a,
                                           comment=f"LazyBoy run {run.id}")
        await self.db.set_state(run.id, RunState.report_posted)
        return result
```

`_rebuild_parts` re-derives the HTML from `body_final` at execution time and asserts the `content_sha` still matches the gate payload — approval binds to content, checked twice.

---

## API

| Method | Path | Body | Behaviour |
|---|---|---|---|
| `GET` | `/api/runs/{id}/report` | — | `{body_draft, body_final, findings, edited: bool, gate: Gate|null}` |
| `PUT` | `/api/runs/{id}/report` | `{body_final}` | Saves your edits; supersedes any pending `POST_REPORT` gate (its `payload_sha` no longer matches). |
| `POST` | `/api/runs/{id}/report/reset` | — | `body_final = body_draft`. |
| `POST` | `/api/runs/{id}/publish/prepare` | `{include_images?: bool, sections?: string[]}` | Builds HTML + images, opens the `POST_REPORT` gate, returns the gate with `preview`. |
| `GET` | `/api/runs/{id}/publish/preview` | `?refresh=1` | The dry-run: `{html_parts[], rendered_html, images[], attachment_plan[], target_url, char_counts[], warnings[]}`. |
| `GET` | `/api/gates?run_id=&state=` | — | List. |
| `POST` | `/api/gates/{id}/decide` | `{approve, reason?, payload_sha}` | Records the decision. **Does not publish** — that's the next call, so an approval is never a side-effecting request. |
| `POST` | `/api/runs/{id}/publish/execute` | `{gate_id}` | 409 unless the gate is `approved` and fresh. Idempotent by `idempotency_key`. |
| `POST` | `/api/runs/{id}/needs-repo/prepare` | `{note?}` | Opens the `MARK_NEEDS_REPO` gate with tag + comment preview. |
| `POST` | `/api/runs/{id}/needs-repo/execute` | `{gate_id}` | Tag patch + comment, same publisher. |
| `GET` | `/api/runs/{id}/publications` | — | `PublishRecord[]` — what was posted, when, which comment id, sha, attachments. |
| `GET` | `/api/publish/images/{name}` | — | Serves rendered PNGs for the preview pane. |

Events on the run's SSE stream: `gate.opened`, `gate.decided`, `publish.preview_built`, `publish.image_rendered`, `publish.image_skipped`, `publish.attachment_uploaded`, `publish.comment_posted`, `publish.skipped`, `publish.failed`, `publish.sanitiser_stripped`.

---

## UI

Route: `/runs/:runId/report`. Two panes plus a sticky action bar.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Investigation Report · Bug 12345      [Read] [Edit] [Diff] [Preview post]     │
├───────────────────────────────────────────────┬──────────────────────────────┤
│                                               │ OUTLINE                      │
│  ## Root cause                                │  · Summary                   │
│  `CheckoutService.Reserve` dereferences …     │  · Root cause                │
│  ┌───────────────────────────────────────┐    │  · Evidence                  │
│  │ CheckoutService.cs:142                │    │  · Ruled out                 │
│  │ 140  var stock = _repo.Find(sku);     │    │  · Not verified              │
│  │ 142  return stock.Quantity >= qty;    │    │                              │
│  └───────────────────────────────────────┘    │ FINDINGS (3)                 │
│  → CheckoutService.cs:142  [worktree] [ADO]   │  ● root_cause  0.82          │
│                                               │  ○ contributing 0.55         │
├───────────────────────────────────────────────┴──────────────────────────────┤
│ edited · 6 changes            [Reject…]        [Preview post]  [Approve & post]│
└──────────────────────────────────────────────────────────────────────────────┘
```

### 6.11 Rendering and file:line links

```tsx
const rehypeFileRefs: Plugin = () => (tree) =>
  visit(tree, "text", (node, i, parent) => {
    // Foo/Bar.cs:142  |  Bar.cs:142:8  |  src/x.ts:9
    const re = /([\w./\\-]+\.(cs|ts|tsx|py|js|sql|json|yaml|yml)):(\d+)(?::(\d+))?/g;
    // ...split the text node, replacing matches with <FileRef/> elements
  });

function FileRef({ path, line }: { path: string; line: number }) {
  const { data: idx } = useFileIndex();            // path -> {repo, sha, adoUrl}
  const hit = idx?.resolve(path);
  return (
    <span className="inline-flex items-center gap-1 font-mono text-sm">
      <Link to={`/runs/${runId}/files?path=${encodeURIComponent(path)}&line=${line}`}
            className="text-sky-600 hover:underline">{path}:{line}</Link>
      {hit && (
        <a href={`${hit.adoUrl}&line=${line}&lineEnd=${line + 1}&lineStartColumn=1`}
           target="_blank" rel="noreferrer" title="Open in Azure DevOps"
           className="opacity-50 hover:opacity-100"><ExternalLink size={12} /></a>
      )}
      {!hit && <span title="not in a resolved repo" className="text-amber-500">?</span>}
    </span>
  );
}
```

The worktree viewer is a light `CodeMirror` read-only pane fed by `GET /api/runs/{id}/file?path=&` — it opens scrolled to the line with it highlighted. The ADO link uses the sha pinned in the run manifest, so it stays correct after the branch moves. Unresolvable references get an amber `?` rather than a dead link; a report full of `?` is a signal the agent invented paths, and you want to see that before posting.

Code blocks: `rehype-highlight` with a Tailwind-themed stylesheet, a copy button, and a language chip.

### 6.12 Edit and diff

- **Edit** swaps the pane for a markdown editor (CodeMirror, markdown mode, live preview on wide screens). Autosaves to `body_final` on a 800 ms debounce via `PUT /report` with TanStack Query optimistic update. A dirty indicator sits in the action bar.
- **Diff** renders `body_draft` vs `body_final` with `react-diff-viewer-continued` in split mode, word-level highlighting, collapsed unchanged regions. The tab label carries the change count.
- **Reset to agent draft** is available and confirms first.
- Editing while a `POST_REPORT` gate is pending marks the gate `superseded` server-side and the action bar swaps back to "Preview post" — you cannot approve text and then post different text.

### 6.13 Preview (dry-run) modal

Three tabs, all populated from `GET /publish/preview`:

| Tab | Content |
|---|---|
| **Rendered** | The exact converted HTML in a sandboxed `<iframe srcdoc>` styled to approximate ADO's comment CSS. This is what your colleagues will see. |
| **HTML** | The literal payload, syntax-highlighted, with per-part character counts and a red bar when a part exceeds the 30 000 target. |
| **Attachments** | Each PNG at full size with its filename, byte size, and the `<img>` tag that will reference it. Individual images can be excluded, which re-prepares the gate. |

Above the tabs: the target — org / project / work item id / title / the comment API URL — and any `warnings[]` (`sanitiser stripped <foo>`, `3 unresolved file references`, `chunked into 2 comments`, `Chromium unavailable — posting without images`). The primary button is **Approve & post**; it calls `decide` then `execute` and streams `publish.*` events into a small progress list. Below it, in small type: *"This posts a comment as you (francesco.colombo@…) to work item 12345. It cannot be undone from LazyBoy."* — because that's true and it should be said.

### 6.14 Reject

A small dialog with a required reason and quick chips (*wrong root cause*, *missing evidence*, *too long*, *wrong repo*, *needs more investigation*). On submit: gate → `rejected`, run stays `report_ready`, and a card appears offering **Re-run investigation with this feedback** — pre-filling the Phase 5 `extra_instructions` box with the reason. One click from "this is wrong" to "try again knowing why".

### 6.15 After posting

The report page shows a green banner: comment id(s), a deep link to the work item's discussion, the posted-at timestamp, and a **View exactly what was posted** link opening the stored HTML from `PublishRecord`. Re-posting is disabled unless the report changed (different `content_sha` → different key → allowed, and the UI says "this will add a second comment").

---

## Tests

| Layer | Test | Mechanism |
|---|---|---|
| Gate | approve/reject transitions; double-decide → 409; stale `payload_sha` → 409; reject without reason → 400; supersede on re-open; edit invalidates a pending gate | unit |
| **Agent cannot publish** | Assert the `lazyboy` MCP server's tool list contains no ADO-write tool; assert `can_use_tool` denies any `Bash` invoking `curl`/`az`; assert `/publish/execute` 409s with no approved gate | unit — this is the P4 regression suite |
| Markdown → HTML | Golden fixtures for each construct (headings, nested lists, tables, fenced code with backticks inside, images, autolinks, `file.cs:142`, mermaid fallback, raw HTML injection attempt `<script>`); assert output equals golden **and** that `bleach.clean` is a no-op on it | golden files |
| Chunking | 120 KB report → parts split at headings, no split inside `<pre>`/`<table>`, markers carry `1/3`…`3/3`, full markdown attached | unit |
| Idempotency | Post; post again → `skipped`; post part 1 then crash → second run posts only part 2; changed content → new key → posts | fake `AdoClient` with an in-memory comment store |
| ADO client | `respx` cassettes for comment create, attachment upload, relation patch, tag patch with `rev` conflict → re-read + retry | recorded, scrubbed |
| ShotRenderer | Real Chromium in CI (`playwright install chromium`): renders a findings table, asserts non-empty PNG, dimensions, and < 4 MB; a test with Chromium force-disabled asserts the text-only path still publishes | integration |
| Needs-repo | Existing tags preserved and deduped; `lazyboy:needs-repo` added once; comment posted with evidence | integration |
| Preview | `/publish/preview` output is byte-identical to what `execute` sends (same builder, asserted by sha) | integration — the whole point of a dry-run |
| UI | Vitest: file-ref plugin linkifies the six path shapes; diff shows edits; approving with stale sha surfaces the reload prompt; excluded image re-prepares the gate | component |

---

## Risks & mitigations

| Risk | Why it bites | Mitigation |
|---|---|---|
| **Double-post** on retry/restart/double-click | Noisy work item, embarrassing | Deterministic marker + list-comments check + per-key `asyncio.Lock` + `PublishRecord`. Preview is read-only; only `execute` writes. |
| **ADO silently strips HTML** | You preview one thing, colleagues see another | We sanitise to our own allow-list *before* sending and assert the sanitiser is a no-op; the preview iframe renders the exact post-sanitiser payload. Any strip is a `publish.sanitiser_stripped` warning surfaced in the preview. |
| Posting a wrong or embarrassing analysis | It's a customer-visible work item | Mandatory human gate, inline edit, diff vs draft, explicit "cannot be undone" copy, and the footer that says a human reviewed it. Rejection feedback loops into the re-run prompt so the same mistake doesn't recur. |
| **Prompt injection reaching ADO** | Untrusted work item text influencing the report body | No agent tool posts anything; the body you post is the body you read and edited; markdown is sanitised so injected HTML/JS is inert; `prompt_injection` findings are rendered in red at the top of the review. |
| Chromium missing / Playwright not installed | Publish fails on a fresh machine | Images are strictly additive; missing Chromium → `publish.image_skipped` + a preview warning, text still posts. `lazyboy doctor` checks for Chromium and offers `playwright install chromium`. |
| Attachment orphans | Failed post leaves uploaded blobs | Content-sha dedupe reuses uploads; orphans are harmless (unlinked attachments are invisible) and `lazyboy gc` lists them. |
| 429 / TSTU throttling during a multi-part post | Half a report posted | `tenacity` honours `Retry-After` (see [external-apis §4](../reference/external-apis.md)); the marker's `part` field makes resumption exact — the retry posts only the missing parts. |
| Tag write race (`System.Tags` read-modify-write) | Concurrent edit drops someone's tag | Send the work item `rev` for optimistic concurrency; on 409, re-read and retry once; never replace the tag string wholesale without merging. |
| Comment too long / unreadable | Nobody reads a 40 KB wall | 30 KB target, chunking at headings, `<details>` for methodology, full markdown as an attachment. |
| Gate approved for stale content | You approve, edit, then post the old text | `payload_sha` bound at open, checked at decide **and** at execute; editing supersedes. |

---

## Effort

| Work | Estimate |
|---|---|
| Gate model, `GateKeeper.open/decide`, endpoints, events | 0.2 day |
| Markdown → ADO HTML converter + sanitiser + golden fixtures | 0.3 day |
| Publisher: markers, idempotency, chunking, `PublishRecord` | 0.2 day |
| Attachments + Playwright shot renderer + `AttachedFile` relations | 0.2 day |
| Needs-repo path (tag + evidence comment) | 0.1 day |
| Review UI: render, file:line links, edit, diff, preview modal, reject flow | 0.4 day |
| Tests | 0.1 day |
| **Total** | **~1.5 days** (master doc says 1; the preview modal and the file-ref linkifier are what push it) |

**Depends on:** Phase 1 (ADO credential), Phase 2/3 (work item + project context), Phase 5 (a report to review) — though the publisher can be built and tested against fixtures before Phase 5 lands, which is why the master doc's sequencing suggests starting it early.
**Unblocks:** Phase 7's Change Report review (same UI, different `Report` kind) and Phase 8's PR description + commit/PR gates (same `Gate` machinery, same markdown converter).
