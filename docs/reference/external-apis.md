# Reference — External API Cookbook

Companion to [`../LazyBoy-Design.md`](../LazyBoy-Design.md). Everything LazyBoy needs from Azure DevOps and Azure Monitor, with the exact calls.

---

## 1. Authentication

### 1.1 Entra SSO (preferred)

Azure DevOps has a fixed first-party application id used as the token audience:

```
499b84ac-1321-427f-aa17-267ca6975798        # Azure DevOps resource id
scope: 499b84ac-1321-427f-aa17-267ca6975798/.default
```

Azure Monitor / Log Analytics:

```
scope: https://api.loganalytics.io/.default          # Log Analytics query
scope: https://management.azure.com/.default          # ARM (resource discovery)
```

Credential chain (Python, `azure-identity`):

```python
from azure.identity.aio import (
    ChainedTokenCredential, AzureCliCredential,
    DeviceCodeCredential, InteractiveBrowserCredential,
)

credential = ChainedTokenCredential(
    AzureCliCredential(),                    # reuse `az login` — silent, MFA already satisfied
    InteractiveBrowserCredential(            # falls back to a browser popup
        tenant_id=cfg.tenant_id,
        cache_persistence_options=TokenCachePersistenceOptions(name="lazyboy"),
    ),
    DeviceCodeCredential(tenant_id=cfg.tenant_id),   # headless / restricted-browser fallback
)

token = await credential.get_token("499b84ac-1321-427f-aa17-267ca6975798/.default")
# → Authorization: Bearer <token.token>
```

Notes that matter in an enterprise tenant:

- `AzureCliCredential` shells out to `az account get-access-token`. It is the cheapest path and inherits any Conditional Access session already satisfied by `az login`. **Try it first, always.**
- Some tenants block third-party device-code flows. If `DeviceCodeCredential` fails with `AADSTS7000218` / `AADSTS50076`, fall back to PAT and say so plainly in the UI.
- Tokens are short-lived (~1h). Cache the `AccessToken` and refresh when `expires_on - now < 5 min`. Never persist the raw token — persist nothing, re-acquire from the credential.

### 1.2 PAT fallback

A classic ADO Personal Access Token, Basic-auth encoded with an empty username:

```python
import base64
b = base64.b64encode(f":{pat}".encode()).decode()
headers = {"Authorization": f"Basic {b}"}
```

Minimum scopes for LazyBoy:

| Scope | Why |
|---|---|
| Work Items **Read & Write** | list, read, comment, tag |
| Code **Read & Write** | clone, push branch |
| Code — Pull Request **Contribute** | create PR |
| Build **Read** *(optional)* | associate build definitions with repos in the catalog |

Store with `keyring.set_password("lazyboy", "ado_pat", pat)`. Record the expiry date so the UI can warn you a week out (`GET https://vssps.dev.azure.com/{org}/_apis/tokens/pats?api-version=7.1-preview.1`).

### 1.3 Git credentials for clone/push

Use the same credential, injected per-command rather than written to `~/.git-credentials`:

```bash
git -c http.extraheader="Authorization: Bearer $TOKEN" clone https://dev.azure.com/{org}/{project}/_git/{repo}
```

Pass the header via `-c http.extraheader` on each invocation and keep the token out of the remote URL, out of `git config`, and out of the process title where practical (use `GIT_CONFIG_COUNT`/`GIT_CONFIG_KEY_0`/`GIT_CONFIG_VALUE_0` env form to avoid it appearing in `ps`).

---

## 2. Azure DevOps REST API

Base: `https://dev.azure.com/{organization}`
API version: `?api-version=7.1` (comments and search need preview versions, noted below).

### 2.1 Work items assigned to me

WIQL query — `@Me` resolves server-side to the authenticated identity, which is exactly the "extension of me" semantic you want:

```http
POST https://dev.azure.com/{org}/_apis/wit/wiql?api-version=7.1
Content-Type: application/json

{
  "query": "SELECT [System.Id] FROM WorkItems
            WHERE [System.AssignedTo] = @Me
              AND [System.State] NOT IN ('Closed','Removed','Done')
            ORDER BY [System.ChangedDate] DESC"
}
```

Returns `workItems: [{id, url}]` only. Hydrate in one batch call (max 200 ids):

```http
POST https://dev.azure.com/{org}/_apis/wit/workitemsbatch?api-version=7.1
{
  "ids": [12345, 12346],
  "fields": ["System.Id","System.Title","System.State","System.WorkItemType",
             "System.AssignedTo","System.Tags","System.ChangedDate",
             "Microsoft.VSTS.Common.Priority","Microsoft.VSTS.Common.Severity"]
}
```

Human-facing link: `https://dev.azure.com/{org}/{project}/_workitems/edit/{id}`

> **Cross-project:** run the WIQL at org scope (no `{project}` segment) to catch items across every project you touch. The response's `url` field tells you which project each item belongs to.

### 2.2 Full work item detail

```http
GET https://dev.azure.com/{org}/_apis/wit/workitems/{id}?$expand=all&api-version=7.1
```

`$expand=all` gives you `fields`, `relations`, and `_links` in one shot. What LazyBoy consumes:

| Field | Use |
|---|---|
| `System.Description` | HTML → text + extracted links/images |
| `Microsoft.VSTS.TCM.ReproSteps` | HTML; often where the App Insights URL hides |
| `Microsoft.VSTS.TCM.SystemInfo` | environment/build info |
| `System.Tags` | `;`-separated; LazyBoy writes `lazyboy:needs-repo` here |
| `relations[]` | `AttachedFile`, `Hyperlink`, `System.LinkTypes.*`, `ArtifactLink` (commits/PRs/builds) |

### 2.3 Comments

```http
GET  https://dev.azure.com/{org}/{project}/_apis/wit/workItems/{id}/comments?api-version=7.1-preview.4
POST https://dev.azure.com/{org}/{project}/_apis/wit/workItems/{id}/comments?api-version=7.1-preview.4
{ "text": "<div>…HTML…</div>" }
```

Comment bodies are HTML. LazyBoy renders its markdown report to a constrained HTML subset (`h3/h4, p, ul/ol/li, code, pre, a, img, strong, em, table`) — ADO strips the rest.

### 2.4 Attachments

**Download** (from a `relations[]` entry with `rel == "AttachedFile"`):

```http
GET {relation.url}?download=true            # relation.url is .../_apis/wit/attachments/{guid}
```

Inline images embedded in `System.Description` are `<img src="https://dev.azure.com/{org}/{project}/_apis/wit/attachments/{guid}?fileName=x.png">` — **they require the same auth header**. Naïvely handing that URL to a browser or a vision model fails; LazyBoy downloads them server-side into `runs/<id>/attachments/` and rewrites the HTML to local paths.

**Upload** (for posting screenshots back with a report):

```http
POST https://dev.azure.com/{org}/{project}/_apis/wit/attachments?fileName=diff.png&api-version=7.1
Content-Type: application/octet-stream
<binary>
→ { "id": "...", "url": "https://dev.azure.com/.../_apis/wit/attachments/{guid}" }
```

Then reference that url in an `<img src>` inside the comment HTML, and/or add it as an `AttachedFile` relation via a JSON-Patch update to the work item.

### 2.5 Updating fields / tags (JSON Patch)

```http
PATCH https://dev.azure.com/{org}/_apis/wit/workitems/{id}?api-version=7.1
Content-Type: application/json-patch+json

[
  {"op":"add","path":"/fields/System.Tags","value":"existing-tag; lazyboy:needs-repo"},
  {"op":"add","path":"/fields/System.History","value":"LazyBoy could not identify a repository."}
]
```

`System.History` writes a discussion entry — an alternative to the comments API, and the one that shows in the classic discussion feed.

### 2.6 Repositories, branches, commits

```http
GET  /{org}/{project}/_apis/git/repositories?api-version=7.1
GET  /{org}/{project}/_apis/git/repositories/{repoId}/refs?filter=heads/&api-version=7.1
GET  /{org}/{project}/_apis/git/repositories/{repoId}/commits?searchCriteria.itemVersion.version={branch}&$top=20&api-version=7.1
```

### 2.7 Pull requests

```http
POST /{org}/{project}/_apis/git/repositories/{repoId}/pullrequests?api-version=7.1
{
  "sourceRefName": "refs/heads/bug/12345-null-ref-in-checkout",
  "targetRefName": "refs/heads/develop",
  "title": "Bug 12345: fix null reference in CheckoutService",
  "description": "…Change Report summary…\n\nAB#12345",
  "reviewers": [{"id": "<identity-guid>"}],
  "workItemRefs": [{"id": "12345"}]
}
```

- `AB#12345` in the description auto-links the work item; `workItemRefs` links it explicitly. Do both.
- Set `"isDraft": true` when you want the PR parked for review.
- Optional follow-up: `PATCH .../pullrequests/{prId}` with `completionOptions` for auto-complete — **not** enabled by default (P4).

### 2.8 Code Search (catalog-miss fallback)

Different host — `almsearch`, not `dev.azure.com`:

```http
POST https://almsearch.dev.azure.com/{org}/_apis/search/codesearchresults?api-version=7.1
{
  "searchText": "class CheckoutService",
  "$top": 25,
  "filters": { "Project": ["MyProject"] },
  "includeFacets": false
}
```

Requires the **Code Search** extension installed in the org. Detect its absence (404) at connect time and degrade gracefully to catalog-only resolution.

---

## 3. Application Insights

### 3.1 Decoding the "end-to-end transaction details" URL

Your example:

```
https://portal.azure.com/#blade/AppInsightsExtension/DetailsV2Blade
  /ComponentId/%7B%22SubscriptionId%22:%225cff65c6-…%22,%22ResourceGroup%22:%22rg_b2c_eus_portalprod_app%22,%22Name%22:%22portal_eus_appinsight%22%7D
  /DataModel/%7B%22eventId%22:%22b1ae1f70-974b-11f1-8e64-000d3a8ba0f6%22,%22timestamp%22:%222026-08-13T19:17:57.5128664Z%22%7D
```

The fragment is a `/`-separated sequence of `key/value` pairs where each value is URL-encoded JSON. Parser:

```python
from urllib.parse import urlparse, unquote
import json, re

def parse_ai_portal_url(url: str) -> dict:
    frag = urlparse(url).fragment           # "blade/AppInsightsExtension/DetailsV2Blade/ComponentId/{…}/DataModel/{…}"
    parts = frag.split("/")
    out = {}
    for key in ("ComponentId", "DataModel"):
        if key in parts:
            raw = unquote(parts[parts.index(key) + 1])
            out[key] = json.loads(raw)
    c, d = out["ComponentId"], out["DataModel"]
    return {
        "resource_id": (
            f"/subscriptions/{c['SubscriptionId']}"
            f"/resourceGroups/{c['ResourceGroup']}"
            f"/providers/microsoft.insights/components/{c['Name']}"
        ),
        "event_id": d.get("eventId"),
        "timestamp": d.get("timestamp"),
        "operation_id": d.get("operationId"),   # sometimes present instead of eventId
    }
```

Handle these variants defensively, because the portal emits several:

- Older `#blade/…` vs newer `#view/AppInsightsExtension/DetailsV2Blade/…`
- `DataModel` containing `{"eventId","timestamp"}` (your case), `{"operationId"}`, or `{"eventId","timestamp","operationId"}`
- Doubly-encoded fragments (`%2522`) when the link was pasted through another tool — `unquote` twice if the first pass still contains `%22`.
- Extra segments (`ResourceId`, `TimeContext`) — the index-based lookup above tolerates them.

**Regex fallback**, for when the URL is mangled by ADO's HTML sanitizer:

```python
EVENT_ID = re.compile(r'"?eventId"?\s*[:=]\s*"?([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})')
```

### 3.2 eventId → operation_Id

`eventId` in the portal URL is the **`itemId`** column in the App Insights schema. Resolve it with a resource-centric query:

```kusto
union requests, exceptions, dependencies, traces, customEvents, pageViews, availabilityResults, browserTimings
| where itemId == "b1ae1f70-974b-11f1-8e64-000d3a8ba0f6"
| project itemId, itemType, timestamp, operation_Id, operation_Name, operation_ParentId,
          cloud_RoleName, cloud_RoleInstance, appName = cloud_RoleName
| take 1
```

Scope the query window tightly around the URL's `timestamp` (±5 minutes) — it turns a full-retention scan into a partition hit.

### 3.3 operation_Id → the full end-to-end transaction

```kusto
let opId = "{operation_Id}";
union isfuzzy=true requests, exceptions, dependencies, traces, customEvents, pageViews, availabilityResults
| where operation_Id == opId
| project timestamp, itemType, itemId, operation_Id, operation_ParentId, id,
          name = coalesce(column_ifexists("name", ""), column_ifexists("problemId", "")),
          message   = column_ifexists("message", ""),
          type      = column_ifexists("type", ""),
          target    = column_ifexists("target", ""),
          resultCode= column_ifexists("resultCode", ""),
          success   = column_ifexists("success", ""),
          duration  = column_ifexists("duration", real(null)),
          outerMessage = column_ifexists("outerMessage", ""),
          details   = column_ifexists("details", ""),
          assembly  = column_ifexists("assembly", ""),
          method    = column_ifexists("method", ""),
          cloud_RoleName, cloud_RoleInstance, client_Type, user_Id,
          customDimensions
| order by timestamp asc
| take 1000
```

`exceptions.details` is the gold: a JSON array of `{ parsedStack: [{ assembly, method, level, line, fileName }] }`. That array is the single highest-signal input to repo resolution (Phase 4).

The tree is rebuilt client-side by linking `operation_ParentId → id`.

### 3.4 Running the query from Python

```python
from azure.monitor.query.aio import LogsQueryClient
from datetime import timedelta

client = LogsQueryClient(credential)
resp = await client.query_resource(
    resource_id=parsed["resource_id"],        # the /providers/microsoft.insights/components/... id
    query=KQL,
    timespan=(ts - timedelta(minutes=30), ts + timedelta(minutes=30)),
)
```

`query_resource` is the right call: it targets the App Insights component directly, so you don't need to know the backing Log Analytics workspace id, and it works for both classic and workspace-based resources. Handle `LogsQueryPartialResult` (partial data + `resp.partial_error`) as well as `LogsQueryResult`.

Package: [`azure-monitor-query`](https://pypi.org/project/azure-monitor-query/) (`LogsQueryClient`, and its `azure.monitor.query.aio` async twin).

### 3.5 Useful supplementary queries

The agent gets these as `appi_query` presets:

```kusto
// Same failure, wider blast radius — is it one user or everyone?
exceptions
| where timestamp between (ago(7d) .. now()) and problemId == "{problemId}"
| summarize count(), dcount(user_Id), min(timestamp), max(timestamp) by cloud_RoleName, problemId

// When did it start? (regression fingerprinting → likely offending deploy)
exceptions
| where problemId == "{problemId}" and timestamp > ago(30d)
| summarize count() by bin(timestamp, 1h), cloud_RoleName
| render timechart

// Which release/build introduced it
union requests, exceptions
| where operation_Id == "{opId}"
| project timestamp, cloud_RoleName, application_Version, customDimensions
```

### 3.6 Resource registry

`cloud_RoleName` is your natural join key between App Insights and the repo catalog. Config keeps a small map so a work item that names an app but not a resource still resolves:

```yaml
app_insights:
  default: portal_eus_appinsight
  resources:
    portal_eus_appinsight:
      subscription_id: 5cff65c6-d6f7-4a96-b7ed-64cbf1ed5e89
      resource_group: rg_b2c_eus_portalprod_app
      name: portal_eus_appinsight
      environment: prod
```

---

## 4. Rate limits & etiquette

| Service | Limit | LazyBoy behaviour |
|---|---|---|
| ADO REST | TSTUs; 429 + `Retry-After` | `tenacity` respects `Retry-After`, exponential backoff w/ jitter, cap 5 attempts |
| ADO batch | 200 ids per `workitemsbatch` | chunked |
| Log Analytics | 200 req/30s per user; 100 MB / 500k rows per query | `take 1000` caps, 30-min timespans, query results cached per run |
| Code Search | 1 req/s soft | serialized, 200 ms spacing |

All connectors share a single `httpx.AsyncClient` with HTTP/2, connection pooling, and a 30 s timeout.

---

## Sources

- [Claude Agent SDK — Python reference](https://code.claude.com/docs/en/agent-sdk/python)
- [azure-monitor-query on PyPI](https://pypi.org/project/azure-monitor-query/)
- [azure.monitor.query.LogsQueryClient](https://learn.microsoft.com/en-us/python/api/azure-monitor-query/azure.monitor.query.logsqueryclient?view=azure-python)
- [Investigate failures, performance, and transactions with Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/failures-performance-transactions)
- [Transaction Search and Diagnostics](https://learn.microsoft.com/en-us/azure/azure-monitor/app/transaction-search-and-diagnostics)
