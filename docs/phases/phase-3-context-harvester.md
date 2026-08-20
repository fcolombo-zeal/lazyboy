# Phase 3 — Context Harvester

Part of [LazyBoy Master Design](../LazyBoy-Design.md).

> Depends on Phase 2 (`AdoClient`, work item cache) and Phase 1 (Azure credential for Log Analytics). Produces `BugContext` + `CandidateSymbols`, which Phase 4 (repo resolution) and Phase 5 (investigation agent) consume. This is the phase where LazyBoy stops being a nicer ADO inbox and starts being *useful*.

---

## Goal

Given a work item id, produce a single normalized, provenance-tagged `BugContext` that contains everything a competent engineer would have gathered by hand in twenty minutes: the actual text of the bug, every link someone dropped in it, the App Insights end-to-end transaction behind that portal URL, the exception stack with assembly/file/line, the screenshots downloaded and readable, the related and duplicate work items, what we tried last time — and a ranked list of *symbols* that tells Phase 4 which repositories to open.

**Two symbol pipelines, not one.** The environment is .NET/C# on the back end and JS/TS on the front end, in *separate repos* (master doc §8, answer 3). A single transaction therefore routinely carries two completely different kinds of evidence: server-side CLR `parsedStack` frames, and browser-side `pageViews` / `browserTimings` / `exceptions` rows with `client_Type == "Browser"` and a JavaScript stack. Both are harvested, both are normalised, and both emit `CandidateSymbols` — tagged with a `stack_kind` discriminator (§3.4) so Phase 4 can weight a minified bundle frame very differently from an assembly name. A full-stack bug that resolves to two repos is the *normal* outcome, and it starts here.

**Scope is one ADO org, one project** (master doc §8, answer 1). Every ADO call the harvester makes is project-scoped; there is no project disambiguation anywhere in this phase. Org-scope stays behind the `ado.scope` config flag and is not exercised in v1.

Hard requirement from P6 (deterministic shell, agentic brain): **the harvester contains no LLM calls.** It is regex, HTTP, and KQL. Determinism here is what makes the agent's later reasoning reproducible and cheap.

Second hard requirement: **partial failure is normal.** App Insights unreachable, attachment 404, a deleted linked work item, a portal URL format we've never seen — none of these fail the run. Each extractor is independently fallible and its failure is *recorded as evidence* ("we tried, here's why it didn't work"), because the agent needs to know the difference between "no stack trace exists" and "we couldn't fetch the stack trace".

## Definition of Done

- [ ] `POST /api/runs` transitions `created → harvesting → ready_to_investigate` (or `blocked_no_repo` handoff to Phase 4) and streams `harvest.*` events over SSE.
- [ ] `GET /api/runs/{id}/context` returns the full `BugContext`; `runs/<id>/context.json` on disk is the canonical copy.
- [ ] Every field in `BugContext` carries provenance: `source`, `confidence`, `extractor`, `extracted_at`, and where applicable a `locator` (URL, KQL, file path, field name).
- [ ] A work item containing an App Insights portal URL yields: `operation_Id`, a parent/child transaction tree, ≥1 exception with `parsedStack` frames, `cloud_RoleName`, `application_Version`, and an impact profile (count, dcount users, first seen, last seen).
- [ ] A work item whose telemetry is *browser-side* yields: the `pageViews` row(s) with `url`/`name`, the `browserTimings` row when present, ≥1 `exceptions` row with `client_Type == "Browser"`, a parsed JS stack (Chrome, Firefox or Safari shape), the bundle file names, the route path, and `application_Version`.
- [ ] Every emitted stack structure carries `stack_kind: "clr" | "js"`, and `CandidateSymbols` keeps browser-derived symbols in their own typed buckets (`bundles`, `routes`, `components`) — never mixed into `assemblies`.
- [ ] A work item with *no* App Insights link still yields a usable context from text mining alone, and says so explicitly.
- [ ] Attachments (≤ 20 files, ≤ 25 MB total, ≤ 8 MB each) are downloaded to `runs/<id>/attachments/`, type-detected, and images are made available to the agent as image content.
- [ ] Inline `<img>` in description/repro HTML is rewritten to local paths.
- [ ] `CandidateSymbols` is produced, ranked, deduplicated, and non-empty whenever any of {stack frames, role names, assembly mentions, file paths} were found.
- [ ] Harvest of a typical bug (1 AI transaction of ~300 rows, 3 attachments) completes in ≤ 12 s p50 / ≤ 30 s p95.
- [ ] Re-harvest is idempotent: same inputs → byte-identical `context.json` modulo timestamps.
- [ ] Context UI panel renders evidence cards grouped by source, the transaction timeline, the attachment gallery, and the raw KQL with a copy button.

---

## Design

### 3.1 `BugContext` — the normalized structure

`BugContext` is a versioned, JSON-serialisable Pydantic model persisted at `runs/<id>/context.json`. It is the *only* thing Phase 4 and Phase 5 read; nothing downstream calls ADO or App Insights for primary context.

```
BugContext
├── schema_version: int                  # bump on breaking change; migrations in db/
├── run_id, org, project, work_item_id
├── harvested_at, duration_ms, harvester_version
├── work_item: WorkItemDetail            # title, type, state, priority, severity, tags,
│                                        # area/iteration, created/changed, assigned, url
├── narrative: Narrative
│     ├── description_text / description_html (sanitised, img rewritten)
│     ├── repro_steps_text / _html
│     ├── system_info_text
│     ├── acceptance_criteria_text
│     └── custom_fields: {ref_name: {label, value_text}}
├── links: Link[]                        # every URL found, classified
├── app_insights: AppInsightsContext | None
│     ├── resolved_from: Link            # which link got us here
│     ├── resource: {id, name, subscription_id, resource_group, environment}
│     ├── event_id, operation_id, timestamp, timespan
│     ├── queries: KqlQuery[]            # every KQL we ran, with row counts + duration
│     ├── telemetry: TelemetryRow[]      # the raw union result, capped
│     ├── tree: TransactionNode          # parent/child, rebuilt client-side
│     ├── exceptions: ExceptionInfo[]    # type, message, problemId, frames[], stack_kind
│     ├── browser: BrowserContext | None # §3.3b — pageViews, browserTimings, JS exceptions,
│     │                                  # bundles, routes, components, source-map status
│     ├── roles: RoleInfo[]              # cloud_RoleName + versions + instance count
│     └── impact: ImpactProfile | None   # count, users, first_seen, last_seen, trend[]
├── attachments: Attachment[]            # local_path, mime, size, kind, sha256, is_image
├── mined: MinedArtifacts                # stack traces, exception types, assemblies,
│                                        # file paths, error codes, correlation ids,
│                                        # sql fragments, http statuses, urls
├── related: RelatedWorkItem[]           # linked / parent / child / duplicate + comments
├── comments: Comment[]                  # this item's own thread, oldest first
├── prior_runs: PriorRun[]               # institutional memory
├── candidate_symbols: CandidateSymbols   # the Phase-4 input (§3.4)
├── evidence: Evidence[]                 # flat, provenance-tagged index over everything
└── failures: ExtractorFailure[]         # extractor, error, retryable, message
```

**Provenance is not optional.** Every `Evidence` row:

```python
class Evidence(BaseModel):
    id: str                       # stable: sha1(extractor|kind|value)[:12]
    kind: EvidenceKind            # stack_frame | assembly | role_name | file_path |
                                  # exception_type | url | error_code | correlation_id |
                                  # sql | http_status | version | user_impact | note
    value: str
    source: Source                # see table
    extractor: str                # "appinsights.exceptions", "text.stack_trace", …
    confidence: float             # 0.0–1.0, see §3.4
    locator: str | None           # field ref name, URL, KQL hash, file path, frame index
    context_snippet: str | None   # ±120 chars around the match, for the UI
    extracted_at: datetime
```

`Source` enum, in descending trust:

| Source | Meaning | Base confidence |
|---|---|---|
| `app_insights_exception` | `exceptions.details[].parsedStack[]` — structured, machine-emitted | 0.95 |
| `app_insights_telemetry` | other AI columns (`cloud_RoleName`, `resultCode`, `operation_Name`) | 0.90 |
| `ado_field_structured` | typed ADO fields (priority, tags, area path, `ArtifactLink` relations) | 0.85 |
| `ado_relation` | linked/child/duplicate work items | 0.80 |
| `attachment_text` | text mined from a `.txt`/`.log`/`.json` attachment | 0.70 |
| `ado_field_text` | free text mined from description / repro steps | 0.60 |
| `ado_comment` | mined from a discussion comment | 0.55 |
| `prior_run` | a previous LazyBoy finding | 0.50 |
| `attachment_image` | something only a vision model could read | 0.40 (agent may raise it) |
| `user_manual` | you typed it into the Context panel | 1.00 |

The confidence numbers are not decoration — Phase 4 ranks repo candidates by summed weighted evidence, and Phase 5's system prompt renders evidence in confidence order with the low-confidence tail explicitly labelled "weak signal, verify".

### 3.2 The pipeline: ordered extractors

The harvester is a list of `Extractor` objects run by a driver that owns timing, error isolation, event emission, and the shared mutable `BugContext`. Order matters because later extractors read what earlier ones wrote (link extraction feeds App Insights resolution; App Insights feeds candidate ranking).

| # | Extractor | Reads | Writes | Fallible? | Typical ms |
|---|---|---|---|---|---|
| 1 | `WorkItemFieldsExtractor` | ADO `$expand=all` | `work_item`, `narrative`, `comments` | hard-fail (no item = no run) | 300 |
| 2 | `LinkExtractor` | narrative HTML + comments + relations | `links[]` | no | 5 |
| 3 | `AttachmentExtractor` | `relations[]` where `rel == AttachedFile` | `attachments[]` | soft | 500–4000 |
| 4 | `InlineImageRewriter` | narrative HTML + `attachments[]` | rewritten `_html`, extra attachments | soft | 200 |
| 5 | `AppInsightsExtractor` | `links[]` (kind `app_insights`) | `app_insights` | **soft** | 1500–9000 |
| 5b | `BrowserTelemetryExtractor` | `app_insights.telemetry` rows | `app_insights.browser` | **soft** | 200–2500 |
| 6 | `TextMiningExtractor` | all text incl. attachment text + AI messages | `mined` | no | 30 |
| 7 | `RelatedItemsExtractor` | `relations[]` link types | `related[]` | soft | 400 |
| 8 | `PriorRunExtractor` | SQLite | `prior_runs[]` | no | 5 |
| 9 | `CandidateSymbolExtractor` | everything above | `candidate_symbols` | no | 10 |

Extractors 3, 5 and 7 are network-bound and mutually independent → run concurrently in a `TaskGroup`, with 4, 5b and 6 gated on their completion. 5b runs immediately after 5 because it consumes the same `telemetry[]` rows — it issues at most one extra KQL (the browser-detail query, §3.3b) and only when browser rows are actually present, so the pure back-end case pays nothing for it. The driver emits `harvest.extractor.started/finished/failed` events so the UI shows a live checklist rather than a spinner.

**Partial-failure semantics.** A soft-fail extractor that raises is caught, recorded in `failures[]`, and the pipeline continues. The rule, stated once so it isn't relitigated later: *the only hard failure is being unable to read the work item itself.* App Insights down, Log Analytics 403, a portal URL from 2019, an attachment the server lost — all soft. `failures[]` is rendered prominently in the UI and injected into the agent prompt, because "we could not reach App Insights" changes how the agent should reason.

### 3.3 Extractor detail

#### 1. Work item fields

`GET /wit/workitems/{id}?$expand=all` (conditional, reusing Phase 2's ETag). Consumed:

| Field ref | Into |
|---|---|
| `System.Title`, `System.State`, `System.WorkItemType`, `System.AssignedTo`, `System.Tags`, `System.AreaPath`, `System.IterationPath` | `work_item` |
| `System.Description` | `narrative.description_*` |
| `Microsoft.VSTS.TCM.ReproSteps` | `narrative.repro_steps_*` — *the highest-yield field*, where the App Insights URL usually hides |
| `Microsoft.VSTS.TCM.SystemInfo` | `narrative.system_info_text` — browser, OS, build number, environment |
| `Microsoft.VSTS.Common.AcceptanceCriteria` | `narrative.acceptance_criteria_text` |
| `Microsoft.VSTS.Build.FoundIn`, `Microsoft.VSTS.Build.IntegrationBuild` | evidence `kind=version` |
| `Custom.*` / any field not in the known set with a non-empty scalar value | `narrative.custom_fields` |

Custom fields are discovered, not hardcoded: any `fields` key matching `^(Custom|WEF_|.*\.Custom)\.` or simply unknown-and-non-empty is captured with its human label resolved once per project from `GET /wit/fields?api-version=7.1` (cached 24 h). Organisations put "Environment", "Affected Customer", "Sentry Link" in custom fields constantly; ignoring them throws away the best context in the item.

Comments: `GET /{project}/_apis/wit/workItems/{id}/comments?api-version=7.1-preview.4`, paginated, oldest first. Comment HTML goes through the same sanitiser and link extractor as the description.

#### 2. Link extraction

Every `<a href>` and every bare URL in every text field, plus `relations[]` of `rel == "Hyperlink"` and `rel == "ArtifactLink"`. Classified by an ordered matcher into:

| Kind | Detection |
|---|---|
| `app_insights` | host `portal.azure.com` and fragment contains `AppInsightsExtension` / `DetailsV2Blade` / `applicationinsights`, **or** any `portal.azure.com` URL containing `eventId`/`operationId` |
| `kusto` | `dataexplorer.azure.com`, `*.kusto.windows.net`, or `portal.azure.com` + `LogsBlade` (carries an embedded, base64+gzip encoded query — decoded, see below) |
| `azure_resource` | `portal.azure.com` + `/resource/subscriptions/...` |
| `ado_work_item` | Phase 2's `parse_work_item_ref` matches |
| `ado_repo` / `ado_commit` / `ado_pr` / `ado_build` | `_git/{repo}`, `_git/{repo}/commit/{sha}`, `_git/{repo}/pullrequest/{n}`, `_build/results?buildId=` — also produced from `ArtifactLink` relations, whose `url` is a `vstfs:///Git/Commit/{proj}%2F{repoId}%2F{sha}` URI that is *parsed*, not fetched |
| `github` | `github.com/{owner}/{repo}[/…]` |
| `attachment` | `_apis/wit/attachments/{guid}` |
| `other` | anything else, kept verbatim |

`ArtifactLink` relations deserve emphasis: `vstfs:///Git/Commit/…` and `vstfs:///Build/Build/{id}` are *direct, structured* statements that this bug is associated with a specific repo/commit/build. That is `ado_field_structured` confidence (0.85) and it frequently short-circuits Phase 4 entirely. Parser:

```python
VSTFS_COMMIT = re.compile(
    r"vstfs:///Git/Commit/(?P<project>[^%/]+)%2F(?P<repo_id>[^%/]+)%2F(?P<sha>[0-9a-f]{40})", re.I)
VSTFS_PR = re.compile(
    r"vstfs:///Git/PullRequestId/(?P<project>[^%/]+)%2F(?P<repo_id>[^%/]+)%2F(?P<pr>\d+)", re.I)
VSTFS_BUILD = re.compile(r"vstfs:///Build/Build/(?P<build_id>\d+)", re.I)
```

**Kusto deep links.** `portal.azure.com/#blade/Microsoft_OperationsManagementSuite_Workspace/AnalyticsBlade/…/query/<encoded>` embeds the query as base64 of a gzip (or deflate) stream. Decode best-effort; a recovered KQL is high-value evidence because someone already did the analysis:

```python
def decode_portal_kusto_query(url: str) -> str | None:
    m = re.search(r"/query/([^/]+)", urlparse(url).fragment)
    if not m:
        return None
    raw = unquote(m.group(1))
    for pad in ("", "=", "==", "==="):
        try:
            blob = base64.b64decode(raw + pad)
        except Exception:
            continue
        for dec in (lambda b: gzip.decompress(b), lambda b: zlib.decompress(b, -15), lambda b: b):
            try:
                return dec(blob).decode("utf-8")
            except Exception:
                continue
    return None
```

#### 3. App Insights resolution

The centrepiece. Uses the decoder and KQL from [`reference/external-apis.md` §3](../reference/external-apis.md) verbatim — this section extends it, it does not restate it.

```
link(kind=app_insights)
  → parse_ai_portal_url()                          [reference §3.1]
      ├─ resource_id, event_id, timestamp, operation_id?
      └─ on failure → EVENT_ID regex fallback + resource from config default
  → if operation_id is None:
        KQL A: itemId == event_id, timespan = ts ± 5 min      [reference §3.2]
        → operation_Id, operation_Name, cloud_RoleName, itemType
  → KQL B: operation_Id == opId, timespan = ts ± 30 min       [reference §3.3]
        → up to 1000 telemetry rows
  → build_transaction_tree(rows)                    §Code
  → parse exceptions[].details[].parsedStack[]      §Code
  → KQL C (impact): problemId over 7d               [reference §3.5]
  → KQL D (onset):  problemId hourly bins over 30d  [reference §3.5]
  → roles: distinct cloud_RoleName × application_Version × dcount(cloud_RoleInstance)
```

**The resource registry.** There are *a few* App Insights resources, split by environment (master doc §8, answer 2) — not one, and not dozens. That shape is what the registry is built for: a short, hand-maintained list in `config.yaml`, keyed by resource **name**, each entry carrying an `environment` tag and the subscription/RG needed to build an ARM id. The portal URL identifies which resource; config supplies the rest.

```yaml
app_insights:
  default: portal-prod                    # used when nothing else identifies a resource
  resources:
    - name: portal-prod
      environment: prod                   # prod | staging | test | dev — free text, matched case-insensitively
      subscription_id: 8f2a…
      resource_group: rg-portal-prod
      role_names: [portal-api, portal-web, checkout-api]
    - name: portal-staging
      environment: staging
      subscription_id: 8f2a…
      resource_group: rg-portal-staging
      role_names: [portal-api, portal-web]
```

Because the same `cloud_RoleName` legitimately appears in several resources (that is what "split by environment" *means*), a role-name match alone never picks a resource — it must be disambiguated by environment. Resolution order:

1. `resource_id` parsed from the portal URL's `ComponentId` → matched to a registry entry by name; an unknown id is resolved once via ARM (`GET /subscriptions/.../providers/Microsoft.Insights/components?api-version=2020-02-02`), cached 24 h, and offered to the UI as "add this resource to config".
2. An **environment hint** mined from the work item — `Microsoft.VSTS.TCM.SystemInfo`, the area path, WI tags, a custom "Environment" field, or an environment token in a URL host (`staging.portal.…`) — narrowed to registry entries whose `environment` matches.
3. A `cloud_RoleName` mentioned in text ∩ `resources[].role_names`, *within* the environment narrowed by (2); if (2) produced nothing and the role name matches >1 resource, the candidates are all recorded and the `default` is used with `resource_guessed: true`.
4. `config.app_insights.default`.

Each degradation is recorded so the UI can say "we guessed the resource — prod, because no environment was stated", and the Context panel's resource chip is a picker: changing it re-runs extractor 5 only. Querying the wrong environment is the most likely way to get confidently wrong telemetry, so the environment used is rendered next to every KQL query, not buried in the resource id.

**Timespan discipline.** Log Analytics bills and times out on scan volume. KQL A uses ±5 min around the URL timestamp; KQL B uses ±30 min; C and D are explicitly bounded in the query and use `timespan=None` with the `where timestamp between` clause carrying the bound. If KQL A returns zero rows, widen once to ±12 h and retry — portal timestamps are occasionally the *blade open* time, not the event time — and record `widened: true`. If still zero, fail soft with `event_id_not_found`.

**Every query is recorded.** `KqlQuery{name, kql, timespan_start, timespan_end, resource_id, row_count, duration_ms, partial, error}`. The UI shows them with a copy button; the agent gets them so it can adapt one for `appi_query`. This is the difference between a tool you trust and a tool you don't.

**`LogsQueryPartialResult`** is handled explicitly: partial rows are kept, `partial=true` is set, `resp.partial_error` is recorded as an `ExtractorFailure` with `retryable=true`. Truncation to 1000 rows is itself recorded (`truncated: true, total_estimated: N`) — an agent reasoning about a transaction must know it saw a prefix.

**Exception extraction.** `exceptions.details` arrives as a JSON *string* containing an array. Each element has `outerId`, `id`, `parsedStack[]`, `message`, `type`, `rawStack`. Frames carry `assembly` (e.g. `Zeal.Portal.Checkout, Version=3.4.1.0, Culture=neutral, PublicKeyToken=null`), `method` (`Zeal.Portal.Checkout.CheckoutService+<ProcessAsync>d__17.MoveNext`), `fileName` (`C:\agent\_work\1\s\src\Checkout\CheckoutService.cs`), `line`, `level`. All four are gold and all four need normalisation — build-agent absolute paths must become repo-relative, async state machines must become their originating method, assembly display names must become simple names.

#### 3b. Browser / front-end telemetry — the second symbol pipeline

Everything above assumes the exception was thrown by the CLR. Half of this environment is a JS/TS SPA in its own repo, and its telemetry arrives through the same App Insights resource with a completely different shape. `BrowserTelemetryExtractor` is the mirror of §3 for that half. It runs off the rows extractor 5 already fetched, so it costs one conditional extra query, not a second pipeline.

**Detection.** A run is "browser-involved" when any telemetry row satisfies *any* of:

- `client_Type == "Browser"` (the decisive one — set by the JS SDK on every row it emits),
- `itemType in ("pageView", "browserTiming")`,
- `cloud_RoleName` matches a catalog repo whose `stacks[]` contains `js` (Phase 4 supplies this; absent a catalog it is skipped),
- an `exceptions` row whose `details[].parsedStack[].assembly` is empty *and* whose `fileName` is an `http(s)://` URL.

Detection is recorded as evidence in its own right: "this transaction has a browser half" changes how the agent reasons even before any frame is parsed.

**The browser detail query.** `KQL_TRANSACTION` (§4.4) already unions `pageViews` and `browserTimings`, but it does not project the browser-specific columns. When detection fires, one extra query pulls them for the same `operation_Id`:

```kusto
let opId = "{operation_id}";
union isfuzzy=true pageViews, browserTimings, exceptions
| where operation_Id == opId and (client_Type == "Browser" or itemType in ("pageView","browserTiming"))
| project timestamp, itemType, itemId, operation_Id, operation_ParentId,
          name, url = column_ifexists("url", ""),
          duration = column_ifexists("duration", real(null)),
          networkDuration     = column_ifexists("networkDuration", real(null)),
          sendDuration        = column_ifexists("sendDuration", real(null)),
          receiveDuration     = column_ifexists("receiveDuration", real(null)),
          processingDuration  = column_ifexists("processingDuration", real(null)),
          totalDuration       = column_ifexists("totalDuration", real(null)),
          type = column_ifexists("type",""), outerMessage = column_ifexists("outerMessage",""),
          details = column_ifexists("details",""), problemId = column_ifexists("problemId",""),
          client_Type, client_Browser = column_ifexists("client_Browser",""),
          client_OS = column_ifexists("client_OS",""), client_City = column_ifexists("client_City",""),
          appVersion = column_ifexists("application_Version",""),
          cloud_RoleName, user_Id, session_Id = column_ifexists("session_Id",""),
          customDimensions
| order by timestamp asc
| take 500
```

What each row type contributes:

| Row type | Fields consumed | Why it matters for repo resolution |
|---|---|---|
| `pageViews` | `name`, `url`, `duration`, `client_Browser`, `client_OS`, `application_Version` | `url` → **route path** (`/orders/9912/checkout`), the single most reliable front-end join key when frames are minified. `name` is usually the SPA route name the app itself set. |
| `browserTimings` | `networkDuration`, `sendDuration`, `receiveDuration`, `processingDuration`, `totalDuration`, `url` | Rarely names code, but "processing 8.4 s of a 9 s page load" localises the bug to client-side work rather than the API, which is a *repo-selection* signal: it argues for the front-end repo and against the back-end one. |
| `exceptions` (`client_Type == "Browser"`) | `type`, `outerMessage`, `details[].parsedStack[]`, `problemId`, `url` in `customDimensions` | The JS stack. Parsed by §3b's frame parser, not by `parse_parsed_stack`'s CLR rules. |
| `dependencies` (`client_Type == "Browser"`) | `target`, `name`, `resultCode` | The browser's own XHR/fetch calls — this is the **seam** between the two repos: a browser dependency whose `target` is the API host, failing with 500, is what makes a bug full-stack. Recorded as a `CrossTierEdge` (below). |

**The cross-tier edge.** When a browser `dependency` row and a server `request` row share `operation_Id` (and usually link through `operation_ParentId` → `id`), the harvester records an explicit `CrossTierEdge{client_row, server_row, target, result_code, client_role, server_role}`. This is the structure that tells Phase 4 "expect two repos, one per side" rather than making it infer that from two unrelated piles of symbols. It is also what the UI uses to draw the browser and server halves of the timeline as two swimlanes.

**JS stack trace formats.** The App Insights JS SDK populates `details[].parsedStack[]` with `{level, method, assembly, fileName, line}`, but `assembly` is empty and `fileName` is a URL — the CLR parser would produce nonsense. Worse, when the exception was captured from a `window.onerror` string or a non-`Error` throw, `parsedStack` is empty and only `rawStack` (or `outerMessage`) survives, so the raw-string parser is not an edge case — it is the common path. Three engine dialects, matched in order:

```python
# Chrome / V8 / Edge / Node:  "    at fnName (https://host/assets/main.a3f9c1.js:1:24601)"
#                             "    at https://host/assets/main.a3f9c1.js:1:24601"
#                             "    at Object../src/App.tsx (http://host/static/js/bundle.js:1234:5)"
JS_CHROME = re.compile(
    r"^\s*at\s+(?:(?P<fn>[^\s(][^(]*?)\s+)?\(?"
    r"(?P<file>(?:https?://|file://|webpack(?:-internal)?://|/|[a-zA-Z]:\\)[^\s()]+?)"
    r":(?P<line>\d+):(?P<col>\d+)\)?\s*$", re.M)

# Firefox / SpiderMonkey:     "fnName@https://host/assets/main.a3f9c1.js:1:24601"
#                             "@https://host/assets/main.a3f9c1.js:1:24601"   (anonymous)
#                             "fnName/<@https://…:1:24601"                    (nested closure)
JS_FIREFOX = re.compile(
    r"^\s*(?P<fn>[^@\s]*?)@"
    r"(?P<file>(?:https?://|file://|webpack(?:-internal)?://|resource://)[^\s]+?)"
    r":(?P<line>\d+):(?P<col>\d+)\s*$", re.M)

# Safari / JavaScriptCore:    "fnName@https://host/assets/main.a3f9c1.js:1:24601"
#                             "global code@https://…:1:24601"
#                             "eval code@[native code]"      → dropped
JS_SAFARI = re.compile(
    r"^\s*(?P<fn>(?:global code|eval code|module code|[\w$.<>\[\]]+)?)@"
    r"(?P<file>[^\s@]+?)(?::(?P<line>\d+):(?P<col>\d+))?\s*$", re.M)

# Bundle basename + content hash:  main.a3f9c1.js · vendor-4f8b2e.chunk.js · index.9c1a.mjs
JS_BUNDLE = re.compile(
    r"(?P<base>[A-Za-z0-9_\-.~]+?)"
    r"(?:[.\-](?P<hash>[0-9a-f]{6,20}))?"
    r"(?P<mid>\.chunk|\.bundle|\.esm|\.min)?\.(?P<ext>m?js|cjs|jsx?|tsx?)$", re.I)

# Webpack/Vite dev or sourced frames that already name real source
JS_SOURCE_FRAME = re.compile(
    r"(?:webpack(?:-internal)?://[^/]*/|/@fs/|/src/)(?P<path>[\w./\-@]+\.(?:tsx?|jsx?|vue|svelte))"
    r"(?::(?P<line>\d+):(?P<col>\d+))?", re.I)
```

Firefox and Safari are matched separately despite the near-identical `fn@file:line:col` shape, because Safari emits bare-word pseudo-frames (`global code`, `module code`) and frames with **no** `:line:col` at all, and a Firefox-only regex silently drops them. Order matters: Chrome first (it is the only one anchored on `at `), then Firefox (requires `:line:col`), then Safari (tolerates their absence). A frame matching none of the three is kept verbatim as `raw` with `parsed=false` — never dropped, because an unparsed frame is still a string the agent can read.

**Framework noise, JS edition.** The equivalent of the CLR framework denylist:

```python
JS_VENDOR = re.compile(
    r"(?:^|/)(?:node_modules/|vendor[-.]|chunk-vendors|runtime[-.]|polyfill)"
    r"|(?:^|/)(?:react|react-dom|scheduler|zone\.js|core-js|lodash|moment|rxjs|axios"
    r"|@angular|vue|svelte|jquery|@microsoft/applicationinsights)[-./]", re.I)
```

Vendor frames are flagged `is_framework=True` and excluded from repo-resolution symbols, retained as hints. A stack that is *entirely* `react-dom` frames is the JS analogue of a framework-only CLR stack and heads for the same fallback.

**Minified frames and source maps — what we can and cannot recover.** State this plainly because it governs the whole front-end half of Phase 4.

Given `main.a3f9c1.js:1:24601` and **no source map**, we can recover:

- the **bundle base name** `main` and its content hash `a3f9c1` — a weak repo signal (many SPAs ship a `main`), but a *strong* build-identity signal when combined with `application_Version`;
- the **origin host** — a good repo signal via the catalog's `url_patterns`;
- the **route path** from the sibling `pageView.url` — the best front-end repo signal available;
- the **exception type and message** — often the only genuinely diagnostic content, and it survives minification untouched;
- the **component name** when the app uses React error boundaries or `componentStack` (below).

We cannot recover, and must not pretend to: the original file name, the original function name (mangled to `t`, `n`, `Ur`), the original line (everything is line 1), or any meaningful column mapping. `line: 1, col: 24601` is *not* a location; it is a byte offset into a build artifact that no longer exists once the next deploy ships.

Given a source map, all of that is recoverable. The mechanism, and its honest ordering problem:

1. Phase 3 records `application_Version` from the exception row and the bundle name + hash from the frame. Together these identify a build.
2. It **cannot** fetch the map yet, because the map lives in a repo's build output and *which repo* is precisely what Phase 4 has not yet decided. The chicken-and-egg is real and is resolved by deferral, not by cloning speculatively.
3. So Phase 3 emits frames with `source_map_status: "unmapped"` and a `remap_hint {bundle, hash, app_version, origin}`, and Phase 4 proceeds on the unmapped (weaker) evidence.
4. Once a front-end repo is accepted, `POST /api/runs/{id}/context/remap` runs a second pass: look for `*.js.map` next to the built bundle in the repo's build output (`dist/`, `build/`, `wwwroot/`, or the artifact of the build definition whose `buildNumber` matches `application_Version`, fetched via the ADO artifacts API), load it with the `sourcemap` package, and call `map.lookup(line, col)` per frame. Mapped frames are rewritten to `{source_file, source_line, source_name}` with `source_map_status: "mapped"`, `stack_kind` stays `"js"`, and the confidence of the derived symbols is raised from `0.45` to `0.85`. The re-map emits `harvest.remap.finished` and re-runs extractor 9 only.
5. If no map is retrievable — the usual case for a production build that strips them, or a build whose artifacts have been retention-purged — the status becomes `"unavailable"` with a reason, and the note *"front-end frames are minified and no source map was retrievable; repo resolution for the front-end half relies on bundle name, origin host and route path"* is injected into the agent prompt verbatim. The agent must not be allowed to reason as though `Ur@main.a3f9c1.js:1:24601` names a function.

Configuration: `harvest.source_maps: auto | off | require_repo` (default `auto`), plus `harvest.source_map_max_bytes` (default 32 MB — maps are frequently larger than the bundle).

**Component names from error boundaries.** React (and Vue/Angular equivalents) put a `componentStack` into the error info, and teams that wire an error boundary into App Insights land it in `customDimensions`. It is the highest-value front-end symbol available *because it survives minification when `displayName` is preserved* — and it is a direct name-match against files in the front-end repo:

```
The above error occurred in the <CheckoutSummary> component:
    at CheckoutSummary (https://app.contoso.com/assets/main.a3f9c1.js:1:24601)
    at ErrorBoundary (…)
    at CartPage (…)
```

Mined with `COMPONENT_STACK = re.compile(r"^\s*(?:at\s+)?(?P<name>[A-Z][A-Za-z0-9_]{2,})\s*(?:\(|$)", re.M)` over any `customDimensions` key matching `componentStack|component_stack|react.*stack`, and over the `outerMessage` when it carries the "The above error occurred in the `<X>` component" preamble. Names matching `JS_VENDOR`-adjacent framework components (`Suspense`, `Fragment`, `Router`, `Provider`, `ErrorBoundary`) are dropped. Surviving names become `components[]` candidates at `0.75` — below a CLR assembly, well above a minified frame.

**Route paths.** From `pageViews.url` and any browser `dependency` whose target is the app's own origin. Normalised so they are matchable against a repo's router config: strip origin and query/fragment, lowercase, then replace path segments that are numeric, GUID, or ≥20 chars of base64-ish text with `:id`. `/orders/9912/checkout?ref=email` → `/orders/:id/checkout`. Both the raw and the parameterised form are kept — the raw for display, the parameterised for matching.

**What this pipeline emits.** Browser evidence produces `CandidateSymbols` entries in buckets that are *deliberately separate from assemblies*, because a bundle name is not an assembly name and conflating them would let a weak front-end signal borrow the weight of a strong back-end one:

| Bucket | Example | Base confidence | Notes |
|---|---|---|---|
| `bundles[]` | `main` (hash `a3f9c1`), `vendor` | 0.45 | Weak alone; strong in combination with origin host |
| `routes[]` | `/orders/:id/checkout` | 0.70 | The workhorse when frames are minified |
| `components[]` | `CheckoutSummary` | 0.75 | Only when a `componentStack` was captured |
| `js_source_files[]` | `src/pages/CheckoutSummary.tsx` | 0.85 | Only from mapped frames or dev-mode/`webpack://` frames |
| `origins[]` | `app.contoso.com` | 0.60 | Matched against catalog `url_patterns` |
| `npm_packages[]` | `@contoso/portal-ui` | 0.65 | From `customDimensions`, import paths in mapped frames, or vendor chunk names |

`exception_types` and `cloud_role_names` are shared with the CLR pipeline — a JS `TypeError` and a browser role name go in the same buckets they always did, tagged by their source.

#### 4. Attachments

From `relations[]` where `rel == "AttachedFile"`. `relation.url` is `…/_apis/wit/attachments/{guid}`; `relation.attributes` carries `name`, `resourceSize`, `comment`.

```
runs/<id>/attachments/
├── 001_error-screenshot.png
├── 002_stacktrace.txt
├── 003_har-export.har
└── _manifest.json
```

Rules:

| Rule | Value | Why |
|---|---|---|
| Max files | 20 | Beyond that it's a dump, not evidence |
| Max per file | 8 MB | |
| Max total | 25 MB | Keeps `runs/` from becoming a disk problem |
| Timeout | 30 s per file, 4 concurrent | |
| Naming | `{index:03d}_{slug(original_name)}{ext}` | Ordered, collision-free, safe on all filesystems; original name kept in the manifest |
| Type detection | extension → `mimetypes`, then magic-byte sniff (`python-magic` if present, else a 16-byte signature table) | Never trust the extension; ADO happily stores `log.png` |
| Integrity | sha256 recorded; identical sha256 across attachments deduped to one file with two manifest entries | |
| Skip | executables (`.exe/.dll/.msi/.ps1/.bat/.sh`), archives > 2 MB, anything failing the size cap — recorded as `skipped` with reason, never silently dropped | |

Kinds: `image` (png/jpg/gif/webp/bmp), `text` (txt/log/md/csv/json/xml/yaml/har/config), `document` (pdf/docx/xlsx), `archive` (zip — *listed*, not extracted), `binary`, `skipped`.

Text attachments ≤ 512 KB are read and appended to the text-mining corpus with `source=attachment_text` (0.70) — a `stacktrace.txt` attachment is often better than anything in the description. Larger text files are head+tail sampled (first 64 KB + last 64 KB) with a marker.

**Images and the agent.** Screenshots are frequently the *only* record of the error message, and a stack trace pasted as a screenshot is common and infuriating. Two delivery paths, both used:

1. **Read-tool path (primary).** The image sits at an absolute path inside `runs/<id>/attachments/`. The Claude Agent SDK's built-in `Read` tool natively renders image files as image content blocks. The investigator's `cwd` is `runs/<id>/worktrees/`, so the path jail in `GateKeeper` is widened by exactly one read-only exception: `runs/<id>/attachments/**` is readable, nothing else outside the worktrees is. The harvest summary injected into the prompt lists each image with its path, original filename, dimensions, and the uploader's comment, and instructs the agent to `Read` any image that might contain an error message. This is lazy and cheap — the agent pays for pixels only when it decides they matter.
2. **Eager path (opt-in).** For ≤ 3 images ≤ 1.5 MB each, `config.harvest.eager_images: true` pre-attaches them as image content blocks in the first user message. Useful when the bug is literally "look at this screenshot".

Guardrails: images larger than 2000 px on the long edge are downscaled with Pillow (preserving aspect ratio, `LANCZOS`) into `attachments/_resized/` and *that* path is what the agent is told about — a 4K screenshot costs a lot of tokens for no extra legibility. Originals are kept for the UI gallery. Animated GIFs contribute frame 0 only.

#### 5. Inline image rewriting

Description HTML contains `<img src="https://dev.azure.com/{org}/{project}/_apis/wit/attachments/{guid}?fileName=x.png">`. These need the ADO auth header, so neither the browser (which has no ADO cookie for a `dev.azure.com` REST route in this context) nor a vision model can load them. LazyBoy:

1. Scans sanitised HTML for `<img src>` matching the attachment URL shape.
2. For each guid: if already downloaded as a relation attachment, reuse; else download it (counts against the same caps) with `inline=true`.
3. Rewrites `src` to `/api/runs/{run_id}/attachments/{local_name}` — a backend route that streams the local file, so the SPA renders the description faithfully with zero credentials in the browser.
4. Additionally writes a `data-local-path` attribute with the absolute filesystem path, which the agent-facing HTML→text conversion turns into `[image: 001_error.png @ /abs/path]`.

Non-attachment `<img>` (external hotlinks) are left alone but flagged `external_image` — do not proxy arbitrary remote URLs through an authenticated process.

#### 6. Text mining

Runs over a single concatenated corpus: description text, repro steps, system info, acceptance criteria, custom field values, every comment, every text attachment, and every `message`/`outerMessage`/`rawStack` from App Insights. Each match records its offset so `context_snippet` can be produced.

| Artifact | Regex | Notes |
|---|---|---|
| .NET stack frame | `^\s*at\s+(?P<method>[\w\.\+<>`,\[\]]+)\s*\((?P<args>[^)]*)\)(?:\s+in\s+(?P<file>[^:]+):line\s+(?P<line>\d+))?` | `re.M`; the `in …:line N` tail is optional (release builds without PDBs) |
| .NET exception type | `\b(?P<type>(?:[A-Z][A-Za-z0-9_]*\.)+[A-Z][A-Za-z0-9_]*(?:Exception\|Error\|Fault))\b` | namespaced only, avoids matching the word "Exception" |
| Bare exception type | `\b(?P<type>[A-Z][A-Za-z0-9_]{2,}(?:Exception\|Error))\b` | lower confidence |
| JS stack frame | `^\s*at\s+(?P<fn>[\w$.<>\[\]]+)?\s*\(?(?P<file>(?:https?://\|file://\|/\|[a-zA-Z]:\\)[^\s)]+):(?P<line>\d+):(?P<col>\d+)\)?` | |
| Python traceback | `File "(?P<file>[^"]+)", line (?P<line>\d+), in (?P<fn>\S+)` | |
| Java frame | `^\s*at\s+(?P<cls>[\w.$]+)\.(?P<method>[\w$<>]+)\((?P<file>[\w.]+):(?P<line>\d+)\)` | |
| Assembly (display name) | `\b(?P<name>[A-Z][A-Za-z0-9_]*(?:\.[A-Za-z0-9_]+)+)\s*,\s*Version=(?P<ver>\d+(?:\.\d+){1,3})` | from `parsedStack.assembly` and from `Could not load file or assembly` messages |
| Assembly/file (dll) | `\b(?P<name>[A-Za-z0-9_.]+)\.(?:dll\|exe)\b` | |
| Source file path | `(?P<path>(?:[A-Za-z]:\\\|/)[^\s:"'<>|]*?[/\\][^\s:"'<>|]+\.(?:cs\|vb\|fs\|ts\|tsx\|js\|jsx\|py\|java\|kt\|go\|rb\|php\|sql\|razor\|cshtml\|xaml\|json\|yaml\|yml\|config))` | |
| HTTP status | `\b(?:HTTP\s*)?(?P<code>[45]\d{2})\b(?:\s+(?P<reason>[A-Z][A-Za-z ]{2,30}))?` | requires an HTTP-ish neighbourhood word within 40 chars to cut false positives (`status`, `code`, `HTTP`, `response`, `returned`) |
| SQL error | `\b(?:SQL\s*Error\|Msg)\s*(?P<num>\d{3,5})\b\|\bORA-(?P<ora>\d{5})\b\|SqlException.*?Number[=:]\s*(?P<n2>\d+)` | |
| SQL fragment | `(?is)\b(SELECT\s+.{0,400}?\s+FROM\s+[\w\.\[\]]+\|INSERT\s+INTO\s+[\w\.\[\]]+\|UPDATE\s+[\w\.\[\]]+\s+SET\|DELETE\s+FROM\s+[\w\.\[\]]+)` | truncated to 500 chars, used for "which table/proc" signals |
| GUID / correlation id | `\b[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}\b` | classified as `operation_id` when preceded within 40 chars by `operation\|correlation\|trace\|request\|activity\|itemId\|eventId` |
| W3C trace id | `\b00-(?P<trace>[0-9a-f]{32})-(?P<span>[0-9a-f]{16})-[0-9a-f]{2}\b` | |
| App error code | `\b(?P<code>[A-Z]{2,6}[-_]?\d{3,6})\b` | e.g. `ERR-4032`, `PAY_10021`; requires ≥1 digit-run and ≥2 uppercase |
| Build/version | `\b(?P<ver>\d+\.\d+\.\d+(?:\.\d+)?(?:-[A-Za-z0-9.]+)?)\b` near `version\|build\|release\|deployed` | |
| Namespace | derived from exception types and methods: longest common dotted prefix of length ≥ 2 | |

Every regex is compiled once at module import, and the whole corpus is capped at 2 MB before mining (guard against a 50 MB log attachment turning a 30 ms pass into a 30 s pass). Matches are deduped case-sensitively for identifiers, case-insensitively for paths, and counted — frequency is a ranking input.

#### 7. Related work items

From `relations[]`:

| ADO rel | Meaning |
|---|---|
| `System.LinkTypes.Duplicate-Forward` / `-Reverse` | duplicate — *the* highest-value relation; if a duplicate is `Closed` with a fix, the answer may already exist |
| `System.LinkTypes.Hierarchy-Forward` / `-Reverse` | child / parent |
| `System.LinkTypes.Related` | related |
| `Microsoft.VSTS.Common.TestedBy-*` | test case coverage |
| `ArtifactLink` | commit/PR/build (handled by the link extractor) |

Related items are hydrated with one `workitemsbatch` call (ids only, summary fields), capped at 25. For **duplicates and children only**, comments are also fetched (capped at 20 per item), because that's where "we fixed this by …" lives. Related items' descriptions feed the text-mining corpus at `source=ado_relation` confidence, but are marked `secondary=true` so Phase 4 doesn't over-weight a cousin bug's stack trace.

#### 8. Prior runs — institutional memory

Query SQLite for runs whose `work_item_id` matches, or whose work item is in `related[]`, or which produced overlapping `candidate_symbols` (a symbol-overlap join on the persisted `run_symbols` table), ordered by recency, capped at 5. For each: run id, date, state, outcome, the investigation report's *summary* section (not the whole report), the repos it resolved, and whether its fix was merged.

This is cheap and disproportionately valuable: the second time a bug in `Zeal.Portal.Checkout` shows up, the agent starts from "last month, run r_01H… found the null came from `CartSnapshot.Items` being lazily loaded outside the scope" instead of from zero. Prior-run evidence is confidence 0.50 and *always* rendered with its date, because stale institutional memory is worse than none.

### 3.4 `CandidateSymbols` — the Phase 4 contract

Phase 4 ([phase-4-repo-resolution.md](phase-4-repo-resolution.md) §4.1) declares the exact shape it consumes; that declaration is authoritative and this section produces it verbatim:

```python
StackKind = Literal["clr", "js"]          # the discriminator Phase 4 weights on


class StackAssembly(BaseModel):
    stack_kind: StackKind = "clr"         # always "clr" here; present so every
                                          # symbol structure is uniformly discriminated
    name: str                  # simple name, version stripped (§normalisation rule 1)
    depth: int                 # frame index; 0 = innermost throw site
    is_framework: bool         # matched the framework denylist
    method: str | None         # unwrapped from async state machines / closures
    file_name: str | None      # repo-relative path when a build root was cut
    line: int | None


class BrowserFrame(BaseModel):
    stack_kind: StackKind = "js"
    depth: int                 # frame index; 0 = innermost throw site
    is_framework: bool         # matched JS_VENDOR
    dialect: Literal["chrome", "firefox", "safari", "raw"]
    function: str | None       # mangled unless mapped — never trust it when minified
    bundle: str | None         # "main"
    bundle_hash: str | None    # "a3f9c1"
    origin: str | None         # "app.contoso.com"
    url: str | None            # full frame URL, verbatim
    line: int | None           # 1 for a minified bundle — meaningless, kept for remapping
    column: int | None
    source_map_status: Literal["unmapped", "mapped", "unavailable"] = "unmapped"
    source_file: str | None    # populated only when mapped
    source_line: int | None
    source_name: str | None
    raw: str                   # the original frame text, always


class CandidateSymbols(BaseModel):
    stack_kinds: list[StackKind]          # which pipelines produced anything: ["clr"],
                                          # ["js"], or ["clr","js"] for a full-stack bug
    assemblies: list[StackAssembly]       # stack_kind == "clr"
    browser_frames: list[BrowserFrame]    # stack_kind == "js"
    bundles: list[str]                    # §3.3b — bundle base names
    routes: list[str]                     # parameterised route paths
    components: list[str]                 # from componentStack / error boundaries
    js_source_files: list[str]            # only from mapped or dev-mode frames
    origins: list[str]                    # hosts seen in browser frames / pageViews
    npm_packages: list[str]
    cross_tier_edges: list[CrossTierEdge] # browser dependency ↔ server request pairs
    source_map_status: Literal["unmapped", "mapped", "unavailable", "n/a"] = "n/a"
    exception_types: list[str]
    method_signatures: list[str]
    cloud_role_names: list[str]
    file_paths: list[str]
    package_ids: list[str]
    links: list[str]
    ado_artifacts: list[AdoArtifact]      # {kind: commit|pr|build, repo_id, project, id}
    text_tokens: list[str]
    application_version: str | None
    area_path: str | None
    ranked: RankedSymbols                 # Phase-3 enrichment, see below
```

**Why the discriminator exists.** Phase 4 must be able to say "this token came from a minified JS frame, discount it" without re-deriving that from the token's shape. `stack_kind` is carried on the frame structures *and* summarised in `stack_kinds`, so Phase 4 can branch its matcher set (CLR matchers vs. JS matchers) in one read rather than sniffing every string. `stack_kinds == ["clr", "js"]` is the explicit signal that this is a full-stack bug and that a **multi-repo result is expected, not suspicious** — Phase 4 uses it to suppress the usual "two high-confidence candidates means something is wrong" instinct. `source_map_status` is lifted to the top level for the same reason: it is a single input to Phase 4's JS weight schedule, not something to be recomputed per frame.

Note the division of labour, which is deliberate: **Phase 3 does not score repositories, and Phase 4 does not parse stack traces.** Phase 3 owns normalisation (assembly display names, async state machines, build-agent paths, framework classification — and, on the JS side, dialect detection, bundle/hash splitting, route parameterisation, vendor classification and source-map application) and hands Phase 4 clean tokens plus `depth` and `stack_kind`; Phase 4 owns the catalog matchers and their weights. `is_framework`, `depth`, `stack_kind` and `source_map_status` are computed here precisely because Phase 4's `depth_factor`, denylist and JS weight schedule depend on them.

`ranked` is an *additive* companion — Phase 4's matchers work off the flat lists above and can ignore it, but it carries the provenance and confidence the Context UI and the Phase 5 prompt need, and it is what the agent sees:

```python
class SymbolCandidate(BaseModel):
    value: str                 # normalized
    raw_values: list[str]      # every surface form seen
    score: float               # 0..1
    occurrences: int
    sources: list[Source]
    evidence_ids: list[str]
    top_frame: bool = False    # appeared as frame index 0..2 of an exception
    stack_kind: StackKind | None = None   # "clr" | "js" when it came from a stack at all

class RankedSymbols(BaseModel):
    assemblies:  list[SymbolCandidate]
    namespaces:  list[SymbolCandidate]
    types:       list[SymbolCandidate]
    methods:     list[SymbolCandidate]
    file_names:  list[SymbolCandidate]   # basename + best-effort repo-relative path
    role_names:  list[SymbolCandidate]   # cloud_RoleName — the natural catalog join key
    repos_hinted: list[SymbolCandidate]  # from ArtifactLink / repo URLs — near-certain
    bundles:     list[SymbolCandidate]   # js — bundle base names
    routes:      list[SymbolCandidate]   # js — parameterised route paths
    components:  list[SymbolCandidate]   # js — error-boundary component names
    origins:     list[SymbolCandidate]   # js — hosts
    generated_at: datetime
    notes: list[str]                     # e.g. "no App Insights link; symbols from text only"
```

The flat lists are projections of `ranked`: `assemblies` is `ranked.assemblies` in score order mapped to `StackAssembly` (keeping framework entries, flagged, because Phase 4's denylist wants to see and discount them rather than have them silently missing); `cloud_role_names`, `file_paths`, `exception_types` and `method_signatures` are the `value` fields of their ranked counterparts in score order. Order is meaningful and deterministic — Phase 4 relies on "deepest non-framework frame wins", which is `depth`, and on list order for tiebreaks.

**Normalisation rules** (applied before dedup — this is where most of the accuracy lives):

1. **Assembly display name → simple name.** `Zeal.Portal.Checkout, Version=3.4.1.0, Culture=neutral, PublicKeyToken=null` → `Zeal.Portal.Checkout`. Version captured separately as `version` evidence.
2. **Async/iterator state machines → originating method.** `Zeal.Portal.Checkout.CheckoutService+<ProcessAsync>d__17.MoveNext` → type `Zeal.Portal.Checkout.CheckoutService`, method `ProcessAsync`. Also handles `<>c__DisplayClass12_0.<Handle>b__0` (lambda closures) and `<<Foo>g__Local|3_0>` (local functions).
3. **Generic arity stripped.** `Repository\`1.GetAsync` → `Repository.GetAsync`.
4. **Build-agent paths → repo-relative.** `C:\agent\_work\1\s\src\Checkout\CheckoutService.cs` → `src/Checkout/CheckoutService.cs` by cutting at the first known build-root marker (`\_work\<n>\s\`, `/home/vsts/work/1/s/`, `/_work/`, `/src/` as last resort) and normalising separators. Both the basename and the relative path become candidates; the basename is what actually matches during repo resolution, the relative path is a strong confirmation.
5. **Framework noise dropped.** Assemblies/namespaces matching `^(System|Microsoft|Newtonsoft|mscorlib|netstandard|Azure|Serilog|NLog|AutoMapper|MediatR|Polly|EntityFramework|Dapper|xunit|NUnit|Moq|FluentValidation|Swashbuckle)(\.|$)` are excluded from `assemblies`/`namespaces` but retained as `framework_hints` evidence — "the exception is inside EF Core" is useful, just not for choosing a repo. `System.Private.CoreLib` frames are dropped entirely.
6. **Case-preserving dedup** on identifiers; case-insensitive on file paths and role names.
7. **Bundle name → base + hash.** `main.a3f9c1.js` → `bundle="main"`, `hash="a3f9c1"`; `vendor-4f8b2e.chunk.js` → `vendor`. The hash is captured separately as build-identity evidence and is *never* used as a match token — it changes every deploy.
8. **Route parameterisation.** `/orders/9912/checkout?ref=email` → `/orders/:id/checkout`; numeric, GUID and long opaque segments become `:id`. Both forms retained (raw for display, parameterised for matching).
9. **Minified-frame suppression.** A `BrowserFrame` with `source_map_status != "mapped"` contributes **no** `function` or `source_file` symbol at all — its `function` is a mangled identifier and emitting it would poison the token space with `t`, `n`, `Ur`. Only `bundle`, `origin`, and (via the sibling `pageView`) `route` survive. This is the single rule that keeps unmapped front-end telemetry honest.

**Scoring.** For each distinct normalized symbol:

```
score = clamp01(
      base(source_max)                      # highest-trust source that produced it
    + 0.15 * top_frame                      # appeared in the first 3 stack frames
    + 0.10 * min(occurrences, 5) / 5        # repetition
    + 0.10 * multi_source                   # ≥2 distinct sources agree
    + 0.10 * exact_role_match               # equals a cloud_RoleName we also saw
    - 0.20 * secondary_only                 # only ever seen on a related work item
    - 0.30 * framework_adjacent             # matched the noise list but survived as a hint
    - 0.25 * minified_js                    # stack_kind == "js" and source map unavailable
)
```

The `minified_js` penalty is the scoring expression of normalisation rule 9: an unmapped browser frame is machine-emitted (so it starts from `app_insights_exception`'s 0.95) but is *epistemically* far weaker than a CLR frame from the same source, and without the penalty it would outrank a perfectly good back-end assembly. Mapped frames take no penalty — once a source map has been applied, `src/pages/CheckoutSummary.tsx:42` is exactly as trustworthy as `CheckoutService.cs:117`.

Ties break on: top-frame first, then occurrences, then alphabetical (determinism matters — Phase 4's output must be reproducible).

`role_names` deserve a special note: `cloud_RoleName` is the single cleanest join key to the repo catalog (master doc §5.2 `catalog_lookup`), and it comes from `app_insights_telemetry` at 0.90 with zero parsing ambiguity. When a role name is present, Phase 4's job is usually a dictionary lookup.

### 3.5 No App Insights link at all

Roughly half of real bugs. The harvester must not degrade into uselessness. Escalating fallbacks:

1. **Mine for a raw operation/correlation GUID** in the text. If one is found, run KQL B directly against the default (or role-matched) resource with a ±24 h window. A support engineer pasting "correlation id 3fa8…" is common, and this turns it into a full transaction.
2. **Mine for a `problemId` or an exception type + role name.** Run a *discovery* query against the default resource:
   ```kusto
   exceptions
   | where timestamp > ago(14d)
     and type has "{exception_type}"
     and (isempty("{role}") or cloud_RoleName == "{role}")
   | summarize count(), any(operation_Id), min(timestamp), max(timestamp) by problemId, cloud_RoleName
   | top 5 by count_ desc
   ```
   The top `operation_Id` is then fed into the normal pipeline, with everything it produces marked `confidence *= 0.7` and `inferred=true` — we found *an* instance, not *the* instance, and the UI says so in those words.
3. **Text-only mode.** `app_insights = None`, `notes: ["No App Insights telemetry was located. Symbols derived from work item text and attachments only."]`, and the note is injected into the agent prompt. `CandidateSymbols` then leans on mined file paths, exception types, and repo hints from `ArtifactLink`.
4. If even that yields nothing — no symbols at all — the run goes to Phase 4, which will produce `blocked_no_repo`, and the harvest failure list explains exactly what was missing. That is a legitimate, useful outcome: LazyBoy tells you *what to paste in*.

The discovery query in (2) is opt-in via `config.harvest.appi_discovery: true` (default on) and is skipped if no default resource is configured.

### 3.6 Caching & re-harvest

| Layer | Key | TTL | Notes |
|---|---|---|---|
| Work item detail | `(org, id, etag)` | conditional | Phase 2's ETag path |
| Field metadata (labels, states) | `(org, project)` | 24 h | |
| KQL results | `sha256(resource_id + kql + timespan)` | 6 h, per-run copy is permanent | Log Analytics is the slow, rate-limited dependency; re-harvesting a run inside 6 h costs zero queries |
| Attachments | content sha256 | permanent on disk | Re-harvest re-uses the file if sha matches the manifest |
| Portal URL decode | pure function | — | |

`POST /api/runs/{id}/reharvest` re-runs the pipeline in place, writing `context.json` and keeping `context.{n}.json` history so a re-harvest that loses evidence is diffable. `?extractors=appinsights,attachments` re-runs a subset — this is what the UI's per-extractor "Retry" button calls when App Insights was down.

Idempotency: extractors sort their outputs deterministically and the serialiser excludes `extracted_at`/`duration_ms` from a `context_hash` recorded alongside. Same inputs → same hash. This is directly testable and is the guard against the harvester quietly becoming nondeterministic.

---

## Code

### 4.1 Pipeline driver (`stages/harvest/pipeline.py`)

```python
from __future__ import annotations

import asyncio
import time
from typing import Protocol

import structlog

from lazyboy.models.context import BugContext, ExtractorFailure

log = structlog.get_logger("lazyboy.harvest")


class Extractor(Protocol):
    name: str
    soft_fail: bool

    async def run(self, ctx: BugContext, deps: "HarvestDeps") -> None: ...


class HarvestPipeline:
    """Ordered extractors over a shared, mutable BugContext.

    Contract: the ONLY hard failure is being unable to read the work item.
    Everything else is recorded in ctx.failures and the run continues, because
    an agent that knows 'App Insights was unreachable' reasons very differently
    from one that thinks 'this bug has no telemetry'.
    """

    def __init__(self, deps: "HarvestDeps", events: "EventBus") -> None:
        self.deps, self.events = deps, events

    async def run(self, ctx: BugContext) -> BugContext:
        t0 = time.perf_counter()

        await self._stage(ctx, self.deps.work_item_fields)     # 1  hard
        await self._stage(ctx, self.deps.links)                # 2

        # 3, 5, 7 are network-bound and independent — overlap them.
        async with asyncio.TaskGroup() as tg:
            tg.create_task(self._stage(ctx, self.deps.attachments))
            tg.create_task(self._stage(ctx, self.deps.app_insights))
            tg.create_task(self._stage(ctx, self.deps.related_items))

        await self._stage(ctx, self.deps.inline_images)        # 4  needs attachments
        await self._stage(ctx, self.deps.text_mining)          # 6  needs 3 + 5
        await self._stage(ctx, self.deps.prior_runs)           # 8
        await self._stage(ctx, self.deps.candidates)           # 9  needs everything

        ctx.duration_ms = int((time.perf_counter() - t0) * 1000)
        ctx.context_hash = ctx.stable_hash()
        return ctx

    async def _stage(self, ctx: BugContext, ex: Extractor) -> None:
        t0 = time.perf_counter()
        await self.events.emit(ctx.run_id, "harvest.extractor.started", {"extractor": ex.name})
        try:
            await ex.run(ctx, self.deps)
        except Exception as exc:  # noqa: BLE001 — isolation is the point
            ms = int((time.perf_counter() - t0) * 1000)
            log.warning("extractor failed", extractor=ex.name, error=str(exc), exc_info=True)
            ctx.failures.append(
                ExtractorFailure(
                    extractor=ex.name,
                    error_type=type(exc).__name__,
                    message=str(exc)[:500],
                    retryable=_is_retryable(exc),
                    duration_ms=ms,
                )
            )
            await self.events.emit(
                ctx.run_id, "harvest.extractor.failed",
                {"extractor": ex.name, "error": str(exc)[:200], "soft": ex.soft_fail},
            )
            if not ex.soft_fail:
                raise
        else:
            ms = int((time.perf_counter() - t0) * 1000)
            await self.events.emit(
                ctx.run_id, "harvest.extractor.finished",
                {"extractor": ex.name, "duration_ms": ms, "summary": ex_summary(ctx, ex)},
            )
```

### 4.2 Stack-frame parser & normaliser (`stages/harvest/frames.py`)

```python
from __future__ import annotations

import json
import re
from dataclasses import dataclass, field

# Async state machine:  Ns.Type+<MethodName>d__17.MoveNext
ASYNC_SM = re.compile(r"^(?P<type>.+?)\+<(?P<method>[^>]+)>d__\d+\.MoveNext$")
# Lambda closure:       Ns.Type+<>c__DisplayClass12_0.<Handle>b__0
CLOSURE = re.compile(r"^(?P<type>.+?)\+<>c(?:__DisplayClass[\w]*)?\.<(?P<method>[^>]+)>b__[\w]+$")
# Local function:       Ns.Type.<Outer>g__Local|3_0
LOCAL_FN = re.compile(r"^(?P<type>.+?)\.<(?P<outer>[^>]+)>g__(?P<method>[^|]+)\|[\w_]+$")
# Plain:                Ns.Type.Method
PLAIN = re.compile(r"^(?P<type>.+)\.(?P<method>[^.]+)$")
GENERIC_ARITY = re.compile(r"`\d+")

ASSEMBLY_DISPLAY = re.compile(r"^(?P<name>[^,]+?)(?:\s*,\s*Version=(?P<version>[\d.]+).*)?$")

# Build-agent roots, most specific first.
BUILD_ROOTS = [
    re.compile(r"^.*?[\\/]_work[\\/]\d+[\\/]s[\\/]", re.I),          # Azure Pipelines (Windows+Linux)
    re.compile(r"^.*?[\\/]home[\\/]vsts[\\/]work[\\/]\d+[\\/]s[\\/]", re.I),
    re.compile(r"^.*?[\\/]actions-runner[\\/]_work[\\/][^\\/]+[\\/][^\\/]+[\\/]", re.I),
    re.compile(r"^.*?[\\/]build[\\/]sources[\\/]", re.I),
    re.compile(r"^[A-Za-z]:[\\/]", re.I),                             # last resort: drop the drive
]

FRAMEWORK = re.compile(
    r"^(System|Microsoft|mscorlib|netstandard|Newtonsoft|Azure|Serilog|NLog|AutoMapper"
    r"|MediatR|Polly|EntityFramework|Dapper|xunit|NUnit|Moq|FluentValidation|Swashbuckle"
    r"|Castle|StackExchange|IdentityServer|Hangfire|Quartz)(\.|$)",
    re.I,
)


@dataclass(slots=True)
class StackFrame:
    index: int
    assembly: str | None          # simple name, no version
    assembly_version: str | None
    declaring_type: str | None    # Ns.Type, arity stripped, state machine unwrapped
    namespace: str | None
    method: str | None
    file_path: str | None         # repo-relative when we could cut a build root
    file_name: str | None         # basename
    line: int | None
    raw_method: str | None
    is_framework: bool = False


@dataclass(slots=True)
class ExceptionInfo:
    type: str | None
    message: str | None
    outer_message: str | None
    problem_id: str | None
    item_id: str | None
    timestamp: str | None
    role_name: str | None
    app_version: str | None
    frames: list[StackFrame] = field(default_factory=list)
    raw_stack: str | None = None


def normalize_assembly(raw: str | None) -> tuple[str | None, str | None]:
    """'Zeal.Portal.Checkout, Version=3.4.1.0, Culture=…' -> ('Zeal.Portal.Checkout','3.4.1.0')"""
    if not raw:
        return None, None
    m = ASSEMBLY_DISPLAY.match(raw.strip())
    if not m:
        return raw.strip() or None, None
    name = m.group("name").strip()
    # Assemblies are sometimes reported as file names.
    name = re.sub(r"\.(dll|exe)$", "", name, flags=re.I)
    return (name or None), m.group("version")


def split_method(raw: str | None) -> tuple[str | None, str | None]:
    """Unwrap compiler-generated names into (declaring_type, method)."""
    if not raw:
        return None, None
    s = GENERIC_ARITY.sub("", raw.strip())
    s = re.sub(r"\(.*\)$", "", s)          # drop an argument list if present
    for pat, keys in (
        (ASYNC_SM, ("type", "method")),
        (CLOSURE, ("type", "method")),
        (LOCAL_FN, ("type", "method")),
        (PLAIN, ("type", "method")),
    ):
        if (m := pat.match(s)):
            t = m.group(keys[0]).replace("+", ".")
            return t or None, m.group(keys[1]) or None
    return None, s or None


def normalize_source_path(raw: str | None) -> tuple[str | None, str | None]:
    """Build-agent absolute path -> (repo-relative path, basename)."""
    if not raw:
        return None, None
    p = raw.strip().replace("\\", "/")
    for root in BUILD_ROOTS:
        cut = root.sub("", p)
        if cut != p:
            p = cut
            break
    p = p.lstrip("/")
    return (p or None), (p.rsplit("/", 1)[-1] if p else None)


def parse_parsed_stack(details_raw: str | list | None) -> list[ExceptionInfo]:
    """Parse App Insights `exceptions.details`.

    `details` arrives as a JSON *string* holding an array of
    {id, outerId, type, message, parsedStack:[{assembly, method, level, line, fileName}]}.
    Frames are ordered innermost-first by `level` ascending; we preserve that,
    because frame 0 is what Phase 4 weights most heavily.
    """
    if not details_raw:
        return []
    details = json.loads(details_raw) if isinstance(details_raw, str) else details_raw
    if isinstance(details, dict):
        details = [details]

    out: list[ExceptionInfo] = []
    for d in details:
        frames: list[StackFrame] = []
        parsed = d.get("parsedStack") or []
        parsed = sorted(parsed, key=lambda f: f.get("level", 0))
        for i, f in enumerate(parsed):
            asm, ver = normalize_assembly(f.get("assembly"))
            dtype, meth = split_method(f.get("method"))
            path, base = normalize_source_path(f.get("fileName"))
            ns = ".".join(dtype.split(".")[:-1]) if dtype and "." in dtype else None
            frames.append(
                StackFrame(
                    index=i,
                    assembly=asm,
                    assembly_version=ver,
                    declaring_type=dtype,
                    namespace=ns,
                    method=meth,
                    file_path=path,
                    file_name=base,
                    line=_int(f.get("line")),
                    raw_method=f.get("method"),
                    is_framework=bool(
                        (asm and FRAMEWORK.match(asm)) or (ns and FRAMEWORK.match(ns))
                    ),
                )
            )
        out.append(
            ExceptionInfo(
                type=d.get("type"),
                message=d.get("message"),
                outer_message=d.get("outerMessage"),
                problem_id=None,
                item_id=None,
                timestamp=None,
                role_name=None,
                app_version=None,
                frames=frames,
                raw_stack=d.get("rawStack"),
            )
        )
    # Innermost exception first: it is the actual throw site.
    return out


def _int(v) -> int | None:
    try:
        return int(v)
    except (TypeError, ValueError):
        return None
```

### 4.3 Transaction tree builder (`stages/harvest/transaction.py`)

```python
from __future__ import annotations

from collections import defaultdict
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any


@dataclass(slots=True)
class TransactionNode:
    key: str                       # `id` when present, else itemId
    item_id: str
    item_type: str                 # request | dependency | exception | trace | customEvent | …
    name: str
    timestamp: datetime
    duration_ms: float | None
    success: bool | None
    result_code: str | None
    target: str | None
    role_name: str | None
    role_instance: str | None
    message: str | None
    parent_key: str | None
    depth: int = 0
    offset_ms: float = 0.0         # from transaction start — drives the timeline UI
    children: list["TransactionNode"] = field(default_factory=list)
    orphaned: bool = False


def build_transaction_tree(rows: list[dict[str, Any]]) -> tuple[list[TransactionNode], dict]:
    """Rebuild the App Insights end-to-end transaction from a flat union result.

    Linkage is `operation_ParentId -> id` (reference/external-apis.md §3.3).
    Real data breaks this constantly, so:
      * exceptions/traces usually have no `id` of their own -> keyed by itemId,
        attached to the parent named by operation_ParentId
      * the root's operation_ParentId is empty, equal to its own id, or points at
        a row that isn't in our (capped, time-windowed) result set
      * multiple roots happen when the transaction spans a sampled boundary
      * cycles happen when instrumentation is misconfigured — we break them
    Anything we cannot place becomes a synthetic root flagged `orphaned`, never dropped.
    """
    nodes: dict[str, TransactionNode] = {}
    by_item: dict[str, TransactionNode] = {}

    for r in rows:
        item_id = str(r.get("itemId") or "")
        key = str(r.get("id") or "") or item_id
        if not key:
            continue
        node = TransactionNode(
            key=key,
            item_id=item_id,
            item_type=str(r.get("itemType") or "unknown"),
            name=str(r.get("name") or r.get("problemId") or r.get("type") or r.get("itemType") or ""),
            timestamp=_ts(r.get("timestamp")),
            duration_ms=_f(r.get("duration")),
            success=_b(r.get("success")),
            result_code=_s(r.get("resultCode")),
            target=_s(r.get("target")),
            role_name=_s(r.get("cloud_RoleName")),
            role_instance=_s(r.get("cloud_RoleInstance")),
            message=_s(r.get("message")) or _s(r.get("outerMessage")),
            parent_key=_s(r.get("operation_ParentId")),
        )
        # Collisions on `id` are possible (retried dependency); disambiguate by itemId.
        if key in nodes:
            node.key = key = f"{key}#{item_id}"
        nodes[key] = node
        if item_id:
            by_item[item_id] = node

    roots: list[TransactionNode] = []
    children: dict[str, list[TransactionNode]] = defaultdict(list)

    for node in nodes.values():
        p = node.parent_key
        parent = nodes.get(p) if p else None
        if parent is None and p:
            parent = by_item.get(p)
        if parent is None or parent.key == node.key:
            roots.append(node)
        else:
            children[parent.key].append(node)

    # Cycle-safe attach with an explicit stack; depth-cap at 64.
    def attach(node: TransactionNode, depth: int, seen: frozenset[str]) -> None:
        node.depth = depth
        if depth >= 64:
            return
        kids = sorted(children.get(node.key, []), key=lambda n: n.timestamp)
        for k in kids:
            if k.key in seen:
                k.orphaned = True
                roots.append(k)
                continue
            node.children.append(k)
            attach(k, depth + 1, seen | {k.key})

    roots.sort(key=lambda n: n.timestamp)
    placed: set[str] = set()
    for r in list(roots):
        if r.key in placed:
            continue
        placed.add(r.key)
        attach(r, 0, frozenset({r.key}))

    if roots:
        t0 = min(n.timestamp for n in nodes.values())
        for n in nodes.values():
            n.offset_ms = (n.timestamp - t0).total_seconds() * 1000.0

    stats = {
        "node_count": len(nodes),
        "root_count": len(roots),
        "orphan_count": sum(1 for n in nodes.values() if n.orphaned),
        "roles": sorted({n.role_name for n in nodes.values() if n.role_name}),
        "failed_count": sum(1 for n in nodes.values() if n.success is False),
        "exception_count": sum(1 for n in nodes.values() if n.item_type == "exception"),
        "total_duration_ms": (
            max((n.offset_ms + (n.duration_ms or 0)) for n in nodes.values()) if nodes else 0.0
        ),
    }
    return roots, stats
```

### 4.4 App Insights extractor (`stages/harvest/appinsights.py`, excerpt)

```python
KQL_RESOLVE_EVENT = """
union isfuzzy=true requests, exceptions, dependencies, traces, customEvents, pageViews,
                   availabilityResults, browserTimings
| where itemId == "{event_id}"
| project itemId, itemType, timestamp, operation_Id, operation_Name, operation_ParentId,
          cloud_RoleName, cloud_RoleInstance
| take 1
"""

KQL_TRANSACTION = """
let opId = "{operation_id}";
union isfuzzy=true requests, exceptions, dependencies, traces, customEvents, pageViews,
                   availabilityResults
| where operation_Id == opId
| project timestamp, itemType, itemId, operation_Id, operation_ParentId, id,
          name = coalesce(column_ifexists("name", ""), column_ifexists("problemId", "")),
          message      = column_ifexists("message", ""),
          type         = column_ifexists("type", ""),
          target       = column_ifexists("target", ""),
          resultCode   = column_ifexists("resultCode", ""),
          success      = column_ifexists("success", ""),
          duration     = column_ifexists("duration", real(null)),
          outerMessage = column_ifexists("outerMessage", ""),
          details      = column_ifexists("details", ""),
          problemId    = column_ifexists("problemId", ""),
          assembly     = column_ifexists("assembly", ""),
          method       = column_ifexists("method", ""),
          cloud_RoleName, cloud_RoleInstance,
          appVersion   = column_ifexists("application_Version", ""),
          client_Type  = column_ifexists("client_Type", ""),
          user_Id      = column_ifexists("user_Id", ""),
          customDimensions
| order by timestamp asc
| take 1000
"""

KQL_IMPACT = """
exceptions
| where timestamp between (ago(7d) .. now()) and problemId == "{problem_id}"
| summarize occurrences = count(), users = dcount(user_Id),
            first_seen = min(timestamp), last_seen = max(timestamp),
            roles = make_set(cloud_RoleName, 10),
            versions = make_set(application_Version, 10)
"""

KQL_ONSET = """
exceptions
| where problemId == "{problem_id}" and timestamp > ago(30d)
| summarize c = count() by bin(timestamp, 1h), cloud_RoleName
| order by timestamp asc
"""


class AppInsightsExtractor:
    name = "appinsights"
    soft_fail = True   # telemetry is valuable, not mandatory

    async def run(self, ctx: BugContext, deps: HarvestDeps) -> None:
        link = next((l for l in ctx.links if l.kind == "app_insights"), None)
        parsed = parse_ai_portal_url(link.url) if link else None
        if parsed is None:
            parsed = await self._fallback_locate(ctx, deps)
            if parsed is None:
                ctx.notes.append(
                    "No App Insights telemetry was located. "
                    "Symbols derived from work item text and attachments only."
                )
                return

        resource_id = parsed.get("resource_id") or deps.config.default_resource_id()
        ts = _parse_ts(parsed.get("timestamp")) or utcnow()
        op_id = parsed.get("operation_id")

        ai = AppInsightsContext(
            resolved_from=link, resource=deps.config.describe(resource_id),
            event_id=parsed.get("event_id"), timestamp=ts,
        )
        ctx.app_insights = ai

        # eventId -> operation_Id, tight window first, widen once.
        if not op_id and parsed.get("event_id"):
            for window in (timedelta(minutes=5), timedelta(hours=12)):
                rows, q = await deps.appi.query(
                    resource_id,
                    KQL_RESOLVE_EVENT.format(event_id=parsed["event_id"]),
                    timespan=(ts - window, ts + window),
                    name="resolve_event",
                )
                ai.queries.append(q)
                if rows:
                    op_id = rows[0].get("operation_Id")
                    ai.widened = window > timedelta(minutes=5)
                    break
        if not op_id:
            raise AppInsightsNotFound(f"eventId {parsed.get('event_id')} not found in {resource_id}")

        ai.operation_id = op_id
        rows, q = await deps.appi.query(
            resource_id, KQL_TRANSACTION.format(operation_id=op_id),
            timespan=(ts - timedelta(minutes=30), ts + timedelta(minutes=30)),
            name="transaction",
        )
        ai.queries.append(q)
        ai.telemetry = rows
        ai.truncated = len(rows) >= 1000
        ai.tree, ai.tree_stats = build_transaction_tree(rows)

        for r in rows:
            if r.get("itemType") != "exception":
                continue
            for exc in parse_parsed_stack(r.get("details")):
                exc.problem_id = r.get("problemId") or None
                exc.item_id = r.get("itemId")
                exc.timestamp = r.get("timestamp")
                exc.role_name = r.get("cloud_RoleName")
                exc.app_version = r.get("appVersion")
                exc.type = exc.type or r.get("type")
                exc.outer_message = exc.outer_message or r.get("outerMessage")
                ai.exceptions.append(exc)

        ai.roles = _summarize_roles(rows)

        if ai.exceptions and (pid := ai.exceptions[0].problem_id):
            try:
                impact, qi = await deps.appi.query(
                    resource_id, KQL_IMPACT.format(problem_id=_kql_escape(pid)),
                    timespan=None, name="impact",
                )
                ai.queries.append(qi)
                ai.impact = ImpactProfile.from_row(impact[0]) if impact else None

                onset, qo = await deps.appi.query(
                    resource_id, KQL_ONSET.format(problem_id=_kql_escape(pid)),
                    timespan=None, name="onset",
                )
                ai.queries.append(qo)
                ai.onset_series = [(r["timestamp"], r["cloud_RoleName"], r["c"]) for r in onset]
            except Exception as exc:            # impact is a bonus, never a blocker
                ctx.failures.append(ExtractorFailure(
                    extractor="appinsights.impact", error_type=type(exc).__name__,
                    message=str(exc)[:300], retryable=True,
                ))

        ctx.add_evidence_from_appinsights(ai)


def _kql_escape(s: str) -> str:
    """KQL string literals: backslash and double-quote escaping. problemId is
    attacker-adjacent data (it derives from an exception message), so it is
    escaped rather than interpolated raw."""
    return s.replace("\\", "\\\\").replace('"', '\\"')
```

### 4.5 Candidate symbol ranking (`stages/harvest/candidates.py`, excerpt)

```python
BASE_CONFIDENCE = {
    Source.APP_INSIGHTS_EXCEPTION: 0.95,
    Source.APP_INSIGHTS_TELEMETRY: 0.90,
    Source.ADO_FIELD_STRUCTURED:   0.85,
    Source.ADO_RELATION:           0.80,
    Source.ATTACHMENT_TEXT:        0.70,
    Source.ADO_FIELD_TEXT:         0.60,
    Source.ADO_COMMENT:            0.55,
    Source.PRIOR_RUN:              0.50,
    Source.ATTACHMENT_IMAGE:       0.40,
    Source.USER_MANUAL:            1.00,
}


def rank(bucket: dict[str, _Acc]) -> list[SymbolCandidate]:
    out: list[SymbolCandidate] = []
    for value, acc in bucket.items():
        score = BASE_CONFIDENCE[max(acc.sources, key=lambda s: BASE_CONFIDENCE[s])]
        score += 0.15 if acc.top_frame else 0.0
        score += 0.10 * min(acc.occurrences, 5) / 5
        score += 0.10 if len(acc.sources) > 1 else 0.0
        score += 0.10 if acc.role_match else 0.0
        score -= 0.20 if acc.secondary_only else 0.0
        score -= 0.30 if acc.framework_adjacent else 0.0
        out.append(SymbolCandidate(
            value=value, raw_values=sorted(acc.raw), score=max(0.0, min(1.0, score)),
            occurrences=acc.occurrences, sources=sorted(acc.sources, key=lambda s: s.value),
            evidence_ids=sorted(acc.evidence_ids), top_frame=acc.top_frame,
        ))
    # Deterministic ordering: score desc, top_frame, occurrences desc, value asc.
    out.sort(key=lambda c: (-c.score, not c.top_frame, -c.occurrences, c.value))
    return out
```

---

## API

### `POST /api/runs`

```jsonc
{"work_item_id":12345,"mode":"investigate",
 "options":{"eager_images":false,"appi_discovery":true,"appi_resource":null}}
→ 201 {"id":"r_01J8…","state":"created"}
```

Org and project are not parameters — there is exactly one of each, taken from `config.yaml` (master doc §8, answer 1). `appi_resource` names a registry entry (§3.3) when you want to force an environment.

Returns immediately; harvest runs as a background task. The client subscribes to `GET /api/events?run_id=r_01J8…&after=0` and watches `run.state_changed` / `harvest.*`.

Harvest events, in order:

| Event | Payload |
|---|---|
| `run.state_changed` | `{from:"created", to:"harvesting"}` |
| `harvest.extractor.started` | `{extractor}` |
| `harvest.extractor.finished` | `{extractor, duration_ms, summary}` — summary is extractor-specific, e.g. `{"links":7,"app_insights":1}` |
| `harvest.extractor.failed` | `{extractor, error, soft}` |
| `harvest.browser.detected` | `{reason, page_views, browser_exceptions, cross_tier_edges}` — fired only when the transaction has a browser half |
| `harvest.browser.unparsed_frames` | `{count, sample}` — drift detector for R14 |
| `harvest.evidence` | `{kind, value, source, confidence, stack_kind}` — streamed so the Context panel fills in live |
| `harvest.finished` | `{duration_ms, evidence_count, candidate_count, failure_count, context_hash}` |
| `run.state_changed` | `{from:"harvesting", to:"resolving"}` |

### `GET /api/runs/{id}/context`

Full `BugContext` JSON. `?include=` trims the heavy parts for the list view: `telemetry` (raw rows) and `tree` are excluded unless requested. `?format=agent` returns the flattened, prompt-ready markdown rendition the agent receives — exposed deliberately so you can see exactly what the model sees.

### `POST /api/runs/{id}/reharvest`

```jsonc
{"extractors":["appinsights"]}      // omit for a full re-harvest
→ 202 {"run_id":"r_01J8…","extractors":["appinsights"]}
```

### `POST /api/runs/{id}/context/remap`

The deferred source-map pass (§3.3b step 4). Called by Phase 4 once a front-end repo has been accepted and its worktree exists, or manually from the UI.

```jsonc
{"repo":"contoso-portal-web","worktree":"/…/runs/r_01J8…/worktrees/contoso-portal-web"}
→ 202 {"frames":18,"mapped":0,"status":"pending"}
```

Emits `harvest.remap.started` / `harvest.remap.finished` `{frames, mapped, unavailable, reason}` and re-runs extractor 9 only. Idempotent: re-running against the same repo and `application_Version` is a no-op unless `?force=true`.

### `POST /api/runs/{id}/context/evidence`

Manual evidence entry — the "add evidence manually" affordance.

```jsonc
{"kind":"file_path","value":"src/Checkout/CartSnapshot.cs","note":"I think it's this one"}
→ 201 {"id":"ev_…","source":"user_manual","confidence":1.0}
```

Manual evidence is `confidence 1.0`, re-triggers candidate ranking (extractor 9 only), and is never overwritten by a re-harvest.

### `GET /api/runs/{id}/attachments/{name}`

Streams a local attachment with the right `Content-Type` and `Content-Disposition: inline` for images, `attachment` otherwise. Path is validated against the manifest (no traversal — the name must be a key in `_manifest.json`, not a path).

### `GET /api/runs/{id}/context/kql`

The `KqlQuery[]` list, for the copy-button UI. Also available inside the context payload; separate route so the UI can poll it cheaply while a harvest is running.

---

## UI

The **Context panel** is the left column of the Bug Workspace (`/runs/:id`), above the agent transcript. Four sections, collapsible, all rendered from a single `useRunContext(runId)` query invalidated by the `harvest.*` SSE events.

### 6.1 Evidence cards, grouped by source

One card per source group, ordered by base confidence descending, each with a count badge and a confidence meter (a 5-segment bar, not a percentage — false precision is worse than none). Inside a card, evidence rows show `kind` icon, value in monospace, occurrence count, and a `⌄` that expands `context_snippet` with the match highlighted and a "jump to origin" link (scrolls the description, opens the attachment, or focuses the transaction node).

Framework-adjacent and `secondary` evidence collapses behind "Show 14 low-signal items" — present, not shouting.

The `failures[]` list renders at the top of the panel as amber cards: *"App Insights — LogsQueryError: 403 Forbidden on portal_eus_appinsight. [Retry] [Choose resource]"*. Retryable failures get a Retry button wired to the per-extractor re-harvest.

### 6.2 Transaction timeline

A horizontal Gantt/waterfall, the shape everyone already knows from the Azure portal, but faster and local:

- X axis: `offset_ms` from transaction start, with a duration ruler; log-scale toggle for transactions with one 30 s outlier.
- One row per node, indented by `depth`, collapsible subtrees, virtualised past 200 rows.
- Bar colour by `itemType`: request slate-600, dependency sky-500, exception red-600 (rendered as a 4 px marker, not a bar — exceptions are instants), trace zinc-400, customEvent violet-500.
- Failures (`success === false` or `resultCode` 4xx/5xx) get a red left border and the result code inline.
- Role swimlane grouping toggle: group rows by `cloud_RoleName` to make a cross-service transaction legible; each lane header shows the role and its `application_Version`.
- **Tier split.** When `stack_kinds` contains both `clr` and `js`, the timeline defaults to two top-level lanes — **Browser** (pageView, browserTiming, browser dependencies) and **Server** (requests, dependencies, exceptions) — with each `CrossTierEdge` drawn as a connector between the browser dependency and the server request it caused. This is the picture that makes a full-stack bug legible in one glance, and it is the visual argument for why Phase 4 will propose two repos.
- Browser frames render with a **minified badge** when `source_map_status != "mapped"`: the frame shows `main.a3f9c1.js:1:24601` struck through with the tooltip "minified — no source map; function name is not meaningful", and the panel offers [Apply source maps] once a front-end repo has a worktree (calls `/context/remap`).
- Hover → tooltip with name, target, duration, result code, role, instance. Click → detail drawer with the full row including `customDimensions` as a key/value table and, for exceptions, the parsed frames as a clickable list (clicking a frame copies `file:line` and, once Phase 4 has run, offers "Open in worktree").
- Orphaned nodes render in a separate "Unlinked telemetry" group below the tree with an explanatory line — never hidden, because an unlinked exception is often the interesting one.
- A truncation banner when `truncated` is true: *"Showing the first 1000 telemetry rows of this transaction."*

An **impact strip** sits above the timeline when `impact` exists: `312 occurrences · 87 users · first seen 12 Aug 09:14 · last seen 19 Aug 08:02`, with the `onset_series` rendered as a 60 px sparkline. A visible step in that sparkline is the single most useful "which deploy did this" signal, so the sparkline is annotated with `application_Version` change points when the versions are available.

### 6.3 Attachment gallery

Grid of thumbnails for images (lazy `loading="lazy"`, served from `/api/runs/{id}/attachments/{name}`), file chips for everything else with icon, name, size, and the uploader's comment. Click an image → lightbox with zoom and a "Send to agent" button that injects that image as a content block into the next agent turn. Text attachments open in a `<pre>` viewer with the mined matches highlighted. Skipped attachments render greyed with their reason ("8.4 MB exceeds the 8 MB cap").

### 6.4 Raw KQL

A "Queries" disclosure listing each `KqlQuery`: name, row count, duration, timespan, partial flag, and the KQL in a monospace block with syntax highlighting and a **copy button**, plus an "Open in Azure portal" link that builds the `LogsBlade` deep link for the resource. Two reasons this exists and neither is decoration: you will want to tweak a query by hand, and being able to see exactly what LazyBoy asked is what makes its conclusions auditable.

### 6.5 Add evidence manually

An "+ Add evidence" button at the bottom of the panel opens a small form: kind (select), value (text), note (optional). It also accepts a paste of a stack trace, which is run through the same `TextMiningExtractor` and previews the extracted symbols before commit — so pasting a stack trace from Teams into LazyBoy takes four seconds and instantly upgrades the candidate ranking. Manually-added evidence renders with a distinct "you" badge and confidence 1.0.

### 6.6 States

| State | Treatment |
|---|---|
| Harvesting | The extractor checklist renders live: each extractor a row with a spinner → check → duration, evidence counting up |
| No App Insights | A neutral (not amber) card: "No telemetry found. LazyBoy searched the description, repro steps, comments and attachments for a portal link or correlation id." + [Add App Insights link] |
| Full-stack transaction | An info card above the timeline: "This transaction crosses the browser and the server. Phase 4 will likely propose two repositories." + the `CrossTierEdge` list |
| Minified front-end | Amber card: "Front-end frames are minified and no source map is available. Repo resolution for the front-end will use the bundle name, the origin host and the route path." + [Apply source maps] once a front-end worktree exists |
| Inferred telemetry | Amber-bordered card: "This transaction was inferred from a matching exception, not from a link in the work item." + the discovery query |
| Empty context | "LazyBoy found no usable evidence." + a list of what it looked for + [Add evidence manually] |
| Stale context | If the work item changed since `harvested_at`, a banner offers [Re-harvest] |

---

## Tests

Fixtures live in `tests/fixtures/harvest/` and are scrubbed captures of real payloads.

| Fixture | Contents |
|---|---|
| `wi-12345-full.json` | `$expand=all` response: description with an App Insights `<a>`, repro steps with an inline `<img>`, 3 `AttachedFile` relations, a `Duplicate-Forward`, an `ArtifactLink` commit |
| `wi-nolinks.json` | Bug with prose only, no URLs, one pasted stack trace |
| `ai-resolve-event.json` | KQL A result, one row |
| `ai-transaction-300.json` | 300-row union: 1 request, 12 dependencies, 2 exceptions, traces; one orphan; one cycle |
| `ai-transaction-cycle.json` | Deliberately self-parenting rows |
| `ai-exception-details.json` | `details` string with a 24-frame `parsedStack`, async state machines, closures, framework frames |
| `ai-impact.json` / `ai-onset.json` | Impact + hourly bins |
| `portal-urls.txt` | 14 App Insights portal URL variants: `#blade` and `#view`, doubly-encoded, `operationId`-only, extra `TimeContext` segment, sanitizer-mangled |
| `stacktraces/*.txt` | .NET (with and without PDB line info), JS bundle, Python, Java |
| `ai-browser-transaction.json` | Full-stack transaction: `pageView` + `browserTiming` + a browser `dependency` to the API + the server `request` it maps to + a server `exception` — the `CrossTierEdge` fixture |
| `ai-browser-exception.json` | `exceptions` row with `client_Type == "Browser"`, minified `parsedStack`, `componentStack` in `customDimensions` |
| `jsstacks/chrome.txt`, `firefox.txt`, `safari.txt`, `raw-onerror.txt`, `webpack-dev.txt` | One file per dialect, including anonymous frames, `global code`, nested closures (`fn/<@`), `[native code]`, and frames with no `:line:col` |
| `sourcemaps/main.a3f9c1.js.map` | A real (small) source map plus the bundle it maps, for the remap pass |
| `attachments/` | png, 12 MB png (over cap), `log.png` that is really text, zip, `.exe` (skipped), duplicate-content pair |

**Unit**

- `test_frames.py` — `split_method` over every compiler-generated shape; `normalize_assembly` over display names and `.dll` forms; `normalize_source_path` over 6 build-root layouts including a Linux hosted agent and a path with no recognisable root (must return the input, not crash).
- `test_transaction.py` — happy tree shape and depth; single orphan promoted to root with `orphaned=True`; a 3-node cycle terminates and produces exactly one extra root; duplicate `id` collision disambiguated; `offset_ms` monotone; empty input returns `([], stats with zeros)`; 1000 rows builds in < 50 ms.
- `test_mining.py` — every regex against positive and negative corpora. Specifically asserted negatives: a version string `4.500` must not match HTTP status; the word "Exception" alone must not match exception type; a GUID in a URL path must not become a correlation id unless the neighbourhood test passes.
- `test_portal_urls.py` — all 14 variants through `parse_ai_portal_url` + the `EVENT_ID` regex fallback; asserts the fallback fires exactly when the structured parse fails.
- `test_js_frames.py` — each dialect fixture through the three regexes in order: asserts the right dialect is chosen, that Safari's `global code` and no-`:line:col` frames survive, that `[native code]` is dropped, that an unmatched frame is kept with `parsed=false`, and that a mangled `function` never reaches `CandidateSymbols` while unmapped (normalisation rule 9).
- `test_bundles_routes.py` — `JS_BUNDLE` over `main.a3f9c1.js`, `vendor-4f8b2e.chunk.js`, `index.9c1a.mjs`, `app.js` (no hash); route parameterisation over numeric, GUID, base64 and mixed segments, asserting the raw form is retained alongside.
- `test_sourcemap.py` — remap pass over `sourcemaps/`: mapped frames get `source_file`/`source_line`/`source_name` and `source_map_status == "mapped"`, confidence rises, `stack_kind` stays `"js"`; a missing map yields `"unavailable"` with a reason and *no* symbol upgrade; an oversized map is refused by `source_map_max_bytes`.
- `test_cross_tier.py` — `ai-browser-transaction.json` produces exactly one `CrossTierEdge`, `stack_kinds == ["clr","js"]`, and both a browser and a server role name.
- `test_ai_registry.py` — resource selection over the registry: URL-identified resource wins; environment hint from `SystemInfo` narrows staging vs prod; an ambiguous role name across two environments falls back to `default` with `resource_guessed: true`; unknown resource id triggers exactly one ARM lookup and is cached.
- `test_candidates.py` — golden ranking: given a fixed evidence set, the ranked list is byte-identical across runs and across dict-ordering perturbations (determinism test, run with `PYTHONHASHSEED` varied).
- `test_html_rewrite.py` — inline `<img>` rewritten to `/api/runs/.../attachments/...`, external images untouched and flagged, `data-local-path` present.

**Integration** (`respx` for ADO, a stubbed `LogsQueryClient` returning fixture tables)

- Full pipeline over `wi-12345-full.json` → asserts the DoD checklist item by item, plus `context_hash` stability across two runs.
- App Insights 403 → run still reaches `ready_to_investigate`, `failures[]` has one retryable entry, `candidate_symbols` non-empty from text alone.
- `LogsQueryPartialResult` → rows kept, `partial=true`, failure recorded.
- Attachment 404 on one of three → other two present, one `skipped` with reason.
- `wi-nolinks.json` with `appi_discovery: true` → discovery query issued, inferred transaction, all derived confidences multiplied by 0.7.
- Concurrency: attachments + App Insights + related run in overlap; total wall time < sum of parts (assert with a fake clock).
- Re-harvest with `extractors=["appinsights"]` → attachments not re-downloaded (assert zero HTTP calls to the attachment route).

**Property tests** (Hypothesis) — `build_transaction_tree` over randomly generated parent/child graphs including cycles, self-loops, missing parents, duplicate ids: invariants are *no row is lost* (`sum(nodes in all trees) == len(input rows)` after orphan promotion), *no infinite recursion*, *depth ≤ 64*.

**Frontend** — timeline renders a fixture tree, collapses subtrees, groups by role; KQL copy button writes to clipboard; evidence cards group and expand; manual-evidence paste of a stack trace previews the mined symbols.

---

## Risks & mitigations

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | **PII in telemetry** — App Insights rows contain `user_Id`, emails, tokens in `customDimensions`, request URLs with query strings | Sensitive data lands in `context.json`, in the prompt, and potentially in an ADO comment | A redaction pass runs before persistence: email regex, JWT (`eyJ[\w-]+\.[\w-]+\.[\w-]+`), bearer tokens, ADO PAT shape, credit-card Luhn, `?(code\|token\|sig\|key)=` query params → `«redacted:kind»`. `user_Id` is hashed to a stable short id (impact counts still work, identity doesn't leak). Redaction is on by default and configurable per-field, never off silently — a "3 values redacted" chip renders on the card. |
| R2 | **Huge transactions** — a batch job with 40 000 telemetry rows | Slow query, blown memory, useless timeline | `take 1000` in the query itself (not client-side), `truncated` flag surfaced in UI *and* prompt, timeline virtualised, and a "narrow to the failing subtree" affordance that re-queries with `operation_ParentId` scoping. |
| R3 | **Portal URL format drift** — Microsoft changes the blade route again | App Insights extraction silently stops working | Three layers: structured parse → `EVENT_ID`/`operationId` regex fallback → config default resource + mined correlation id. Each degradation emits a distinct `note` and the UI names which layer fired, so drift is visible on day one rather than discovered in a month. A single unit test file holds every URL shape ever seen; new shapes are added there. |
| R4 | Prompt injection via work item text / telemetry messages | Agent follows attacker instructions | Master doc §7.1: everything the harvester produces is untrusted and is delivered inside `<untrusted>` fences; nothing in this path reaches an irreversible action. The harvester additionally strips zero-width and bidi-override characters, which are the cheap way to hide an instruction in an ADO description. |
| R5 | Attachment bombs (zip bomb, 500 MB "log") | Disk / memory exhaustion | Hard caps enforced *before* download via `resourceSize`, streaming download with a byte counter that aborts mid-stream, archives listed but never extracted. |
| R6 | `parsedStack` absent (release build, no PDBs, non-.NET) | No file/line, weak symbols | `rawStack` is mined with the text regexes as a fallback; assembly + type + method still resolve repos in most cases; the note "no source line information available" reaches the agent. |
| R7 | Wrong App Insights resource guessed | Confidently wrong telemetry | Guessing is always recorded and always visible ("resource inferred from config default"); the UI offers a resource picker; inferred telemetry gets the 0.7 confidence multiplier. |
| R8 | Log Analytics throttling (200 req/30 s) | Harvest stalls | ≤ 4 queries per harvest, 6 h result cache keyed by query hash, `tenacity` on 429 with `Retry-After`, and the impact/onset queries are best-effort. |
| R9 | Non-determinism creeping in (set iteration, dict order) | Phase 4 results wobble between runs | `context_hash` asserted stable in CI with a varied `PYTHONHASHSEED`; all rankings have explicit total orderings. |
| R10 | KQL injection via `problemId` / exception text | Data exfil from the workspace, expensive queries | All interpolated values pass `_kql_escape`; only fixed query templates are used; `appi_query` (the agent-facing ad-hoc tool) is row-capped and read-only by construction. |
| R11 | Duplicate work item chains fan out unboundedly | Slow harvest | Related items capped at 25, one hop only, comments only for duplicates/children, capped at 20 each. |
| R12 | Image token cost | Expensive runs | Lazy Read-tool path by default, downscale > 2000 px, eager attachment opt-in and capped at 3 × 1.5 MB. |
| R13 | **Unmapped minified front-end frames are near-useless for repo resolution** — `Ur@main.a3f9c1.js:1:24601` names no file, no function and no line. If production builds ship without retrievable source maps, the entire JS symbol pipeline collapses to bundle name + origin host + route path, and the front-end half of Phase 4 resolves on far weaker evidence than the back-end half. | The front-end repo is mis-resolved or not resolved at all; a full-stack bug silently becomes a back-end-only run; worse, the agent treats a mangled identifier as a real function name and reasons confidently from noise. | **Validate this against one real front-end exception before Phase 3 starts** — this is a prerequisite, not a follow-up. Pull one browser `exceptions` row from the prod resource and answer three questions: (a) is `details[].parsedStack[]` populated at all, or only `rawStack`; (b) is `application_Version` set and does it correspond to a build whose artifacts still exist; (c) does a `.js.map` exist in that build's output or the repo's `dist/`. The answers decide whether the source-map path in §3.3b is a primary mechanism or dead code, and whether Phase 4's JS matcher weights are the "mapped" or "unmapped" schedule. In the meantime the design degrades explicitly, never silently: normalisation rule 9 suppresses mangled symbols entirely, the `minified_js` scoring penalty applies, `source_map_status` is surfaced in the UI and injected into the agent prompt in words, and the deferred `remap` pass (§3.3b step 4) upgrades the frames the moment a front-end repo is accepted. If the validation comes back negative on all three questions, the honest response is to reduce the front-end pipeline's ambition to route + origin + component matching and say so in the docs rather than shipping a source-map path that never fires. Tracked as the first open item in master doc §8.2. |
| R14 | **JS stack dialect drift** — a browser or SDK version emits a fourth stack shape, or `parsedStack` arrives with `assembly` populated with something unexpected | Frames silently unparsed, front-end evidence disappears | Three ordered dialect regexes with a `raw`/`parsed=false` fallback that *keeps* the frame text; a fixture file per engine (§Tests); a `harvest.browser.unparsed_frames` count emitted as an event so drift shows up as a number rather than as an absence. |
| R15 | **Wrong environment queried** — the same `cloud_RoleName` exists in prod, staging and test resources | Confidently wrong telemetry from the wrong environment | The registry is keyed by resource name with an explicit `environment` tag; a role-name match alone never selects a resource (§3.3); the environment in use is rendered beside every KQL query and on the resource chip, and switching it re-runs extractor 5 only. |

---

## Effort

| Task | Estimate |
|---|---|
| `BugContext` models + provenance/evidence plumbing + persistence | 2 h |
| Pipeline driver, event emission, partial-failure semantics | 1.5 h |
| Work item fields + custom field discovery + comments | 1.5 h |
| Link extraction + classification + vstfs/Kusto decoding | 1.5 h |
| App Insights extractor (decode, resolve, transaction, impact) | 3 h |
| Stack-frame parser + normalisation (CLR) | 2 h |
| Browser telemetry extractor: detection, browser KQL, pageViews/browserTimings, cross-tier edges | 2 h |
| JS stack parser (3 dialects + raw fallback), bundle/route/component normalisation | 2 h |
| Source-map awareness: `remap_hint`, deferred remap pass, artifact lookup, degradation notes | 1.5 h |
| App Insights resource registry (environment tags, selection order, ARM fallback, picker) | 1 h |
| Transaction tree builder (+ property tests) | 1.5 h |
| Attachments (download, caps, sniffing, images, resize) | 2 h |
| Inline image rewriting + serving route | 1 h |
| Text mining regex suite | 1.5 h |
| Related items + prior runs | 1 h |
| `CandidateSymbols` ranking | 1.5 h |
| Redaction pass | 1 h |
| REST + SSE surface | 1 h |
| Context UI (cards, timeline, gallery, KQL, manual evidence) | 4 h |
| Tests + fixtures | 3 h |

**≈ 2–2.5 days**, matching the revised master-plan figure (§9). The increase over the original 1.5–2 days is entirely the second symbol pipeline: browser telemetry, the JS stack parser, and source-map awareness. The timeline visualisation remains the piece most likely to overrun — it is also the piece you can ship as a plain nested list on day one and prettify later. The source-map remap pass is the piece most likely to be *cut*: if the R13 validation says maps are not retrievable, it does not get built at all and half a day comes back.
