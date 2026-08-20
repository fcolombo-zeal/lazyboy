# Phase 4 — Repo Resolution & Workspace

Part of [LazyBoy Master Design](../LazyBoy-Design.md).

**Status:** Design — v1.0 · **Depends on:** Phase 3 (Context Harvester) · **Unblocks:** Phase 5 (Investigation), Phase 7 (Fix)

---

## 1. Goal

Turn a `BugContext` into **an ordered, evidence-backed set of `RepoCandidate`s**, get the accepted ones onto local disk as a **per-run git worktree pinned to a base branch you chose**, and — when the evidence genuinely isn't there — fail *loudly and usefully* into `blocked_no_repo` rather than guessing.

Everything here obeys P6 (*deterministic shell, agentic brain*): the resolver is a scoring function over structured evidence. Claude is invoked only for the residual ambiguous cases, via the `repo-scout` subagent, and even then its output is a *proposal* that lands in the same `RepoCandidate` shape with `resolved_by="code-search"` and a confidence the UI can argue with.

### Environment facts this phase is built around

All from master doc §8, and all of them change the shape of this phase rather than just filling in a constant:

| Fact | Consequence here |
|---|---|
| **One ADO org, one project** | Every call is project-scoped. No project enumeration, no cross-project disambiguation, no project column in the picker. `ado.scope: project\|org` exists in config and defaults to `project`; the org branch is written but not exercised in v1. |
| **.NET/C# back end and JS/TS front end in separate repos** | Two inference paths in the scan, two matcher families in the resolver, and — decisively — **multi-repo resolution is the normal case**, not an edge case (§4.6). |
| **No existing service catalog; tribal knowledge only** | `lazyboy catalog scan` is a mandatory, fully-specified deliverable (§3.3), not a seeding convenience. Day one has an empty catalog and the phase must be useful anyway. |
| **30–100 repos** | Cloning everything up front is not viable. The scan reads manifests over the **Items API without cloning**, runs as a resumable background job with a concurrency limit and progress events, and the grep fallback uses bounded partial clones. |
| **Code Search extension: unknown** | Probed at runtime. Present → the primary bootstrap for an empty catalog. Absent → `LocalGrepSearcher` over partial clones. **Both paths are fully designed** (§5); neither is a stub. |
| **Base branch varies per repo** | The catalog stores `default_base_branch` and `default_pr_target` per repo; the UI pre-selects and **always asks** (§8). |

### Why this is the hard phase

Phase 3 hands you `parsedStack[].assembly` strings, a `cloud_RoleName`, some file paths from a build agent, a minified bundle name, a route path, and a work item written by a human at 4pm on a Friday. Nothing in that maps to a git repo without a mapping you own — and you do not have one. The catalog *is* that mapping, the scan is what bootstraps it out of nothing, the resolver is the fuzzy join, Code Search or local grep is the escape hatch, and `blocked_no_repo` is the honest failure. On day one the catalog is empty and every one of those pieces has to carry weight.

---

## 2. Definition of Done

| # | Criterion |
|---|---|
| D1 | `~/.lazyboy/repos.yaml` exists, is schema-validated on load (Pydantic), hot-reloads on change, and errors surface in the UI rather than crashing the process. |
| D2 | **`lazyboy catalog scan` is a shipped, first-class feature**, not a seeding script: it runs as a **resumable background job** over all 30–100 project repos with a bounded concurrency limit, streams `catalog.scan.*` progress events, reads manifests via the ADO Items API **without cloning**, infers .NET and JS/TS identities separately, discovers `cloud_RoleName` edges from telemetry, and ends in a **review-and-confirm UI**. Nothing reaches `repos.yaml` without an explicit click. |
| D2a | The scan completes over 100 repos in ≤ 5 min wall clock on a warm token, survives being interrupted at any point (`--resume` picks up from the checkpoint), and degrades per-repo: one repo that 403s or has no manifests does not fail the job. |
| D2b | Every repo record the scan proposes carries `stacks[]` (`dotnet` / `js` / both), `default_base_branch` and `default_pr_target` read from ADO, plus the evidence for each inferred key so the review UI can show *why*. |
| D3 | Given the Phase-3 `CandidateSymbols`, the resolver returns `RepoCandidate[]` ranked by `confidence ∈ [0,1]`, each with ≥1 human-readable `evidence[]` entry naming the matcher, the matched token, and the weight it contributed. |
| D3a | **Multi-repo is the expected result for a full-stack bug.** When `CandidateSymbols.stack_kinds == ["clr","js"]`, the resolver returns at least one candidate per stack where evidence supports it, the UI presents them as a grouped set rather than as competitors, and a base branch is chosen **per repo**. |
| D4 | The scoring-regression suite (≥25 fixture bugs × fixture catalog) passes: top-1 accuracy ≥ 0.9, and **zero** auto-accepts of a wrong repo (false-accept rate is the metric we actually defend). At least 6 fixtures are full-stack two-repo cases and are scored on *set* accuracy, not top-1. |
| D5 | Above `AUTO_ACCEPT` (0.72) with a ≥0.15 margin over runner-up → auto-accept. Between `NEEDS_CONFIRM` (0.35) and that → UI confirmation. Below → search fallback, then `blocked_no_repo`. The margin rule is applied **within a stack**, not across stacks (§4.6). |
| D6 | **Code Search availability is probed at runtime** and recorded in the health record. Present → the org-search fallback runs at ≤1 req/s from distinctive symbols. Absent → `LocalGrepSearcher` runs instead: bounded partial clones of catalog-known repos, `git grep` across worktrees in a worker pool, ranked into the same `RepoCandidate` shape. Both paths are implemented; neither is a stub, and the UI names which one ran. |
| D7 | `blocked_no_repo` sets local run state, renders an evidence panel, and offers one approval-gated ADO action (tag `lazyboy:needs-repo` + comment listing evidence) plus a manual-attach picker over live org repos. |
| D8 | Accepting a candidate manually (or correcting a wrong one) **writes back** to `repos.yaml` — the learning loop. Writeback is a diff the user sees before it's applied. |
| D9 | For each accepted repo the user picks a **base branch** (suggested from that repo's `default_base_branch`, never silently defaulted); the choice is persisted on the Run **per repo**, and a full-stack run therefore carries two independent base-branch decisions. `default_pr_target` is carried forward to Phase 8 the same way. |
| D10 | `GitWorkspace` produces `runs/<id>/worktrees/<repo>/` from a shared partial clone in ≤10 s on a warm cache for a 4 GB monorepo; cleans up on run completion; survives interrupted clones, stale `index.lock`, and dirty worktrees without manual intervention. |
| D11 | Disk accounting is visible in the UI and an LRU eviction keeps `~/.lazyboy/workspace/` under a configured cap. |

---

## 3. The catalog — `~/.lazyboy/repos.yaml`

### 3.1 Design stance

The catalog is **hand-owned, machine-seeded**. It is checked into a gist/repo of your own if you like, but LazyBoy treats it as local config. It is not a cache: nothing invalidates it but you. What *is* cached (in SQLite) is the ADO-side truth it points at — repo ids, default branches, ref lists — refreshed on a TTL.

Because no catalog exists today (master doc §8, answer 5), "machine-seeded" is doing the load-bearing work: the file below is what `catalog scan` (§3.3) *produces*, and the hand-owned part is the review pass over it. The schema is designed for that order — every key the scan can infer is inferable, and every key it cannot (`cloud_role_names` before telemetry, `weight`, `sparse_paths` on a repo the scan judged small) has a defensible default.

All repos live in the **single configured project**; `project` is still carried on each record because ADO's REST paths and the `vstfs://` artifact URIs contain it, not because more than one is expected.

Every match key is a *key into the same repo record*, so adding a new signal later is additive.

### 3.2 Annotated schema

```yaml
version: 1

defaults:                      # applied to every repo unless overridden
  organization: contoso
  project: Commerce            # the ONE project; ado.scope defaults to "project"
  default_base_branch: main    # per-repo override is the norm, not the exception
  default_pr_target: main      # what Phase 8 targets; often != the base you branch from
  staleness_ttl_minutes: 60    # how long a fetch is considered fresh
  weight: 1.0

denylist:                      # assemblies/namespaces that NEVER resolve to a repo
  assemblies:
    - "System.*"
    - "Microsoft.*"
    - "mscorlib"
    - "netstandard"
    - "Newtonsoft.Json*"
    - "Serilog*"
    - "AutoMapper*"
    - "MediatR*"
    - "Polly*"
    - "StackExchange.Redis*"
    - "Azure.*"
    - "NLog*"
    - "xunit*"
    - "Moq*"
    - "EntityFramework*"
    - "Dapper*"
    - "Swashbuckle.*"
  namespaces:
    - "System"
    - "Microsoft"
    - "Newtonsoft"
    - "Azure"
  bundles:                     # JS side: bundle base names that name no repo
    - "vendor*"
    - "runtime*"
    - "polyfill*"
    - "chunk-vendors*"
  npm_packages:                # never resolve a repo from a third-party package
    - "react"
    - "react-dom"
    - "@angular/*"
    - "vue"
    - "rxjs"
    - "lodash"
    - "axios"
    - "@microsoft/applicationinsights-*"

repos:
  # ── back end ────────────────────────────────────────────────────────────
  - name: contoso-checkout                     # catalog key; also the workspace dir name
    project: Commerce                          # the one configured project
    repo_id: 6a3d9f1e-4c22-4a0b-9f0e-1f2c3d4e5f60   # ADO GUID; authoritative, survives rename
    clone_url: https://dev.azure.com/contoso/Commerce/_git/contoso-checkout
    default_base_branch: main                  # what a fix branches FROM (varies per repo)
    default_pr_target: main                    # what Phase 8 opens the PR INTO
    release_branch_pattern: "release/(?P<version>\\d+\\.\\d+\\.\\d+)"  # for version→branch mapping
    stacks: [dotnet]                           # dotnet | js | python | … ; a list, not a scalar
    language: csharp
    owners: [francesco.colombo@zealitconsultants.ai, teamlead@contoso.com]
    build:
      definition_names: ["Commerce-Checkout-CI"]
      build_cmd: "dotnet build Contoso.Checkout.sln -c Debug"
      verify: compile_only                     # master doc §8 answer 4 — tests are never run
      requires_infra: false
    sparse_paths: []                           # empty = full checkout
    weight: 1.0                                # global multiplier on this repo's score

    match:
      assemblies:                              # glob-matched (fnmatch), case-insensitive
        - "Contoso.Checkout"
        - "Contoso.Checkout.*"
        - "Contoso.Payments.Gateway"
      namespaces:                              # longest-prefix wins across repos
        - "Contoso.Checkout"
        - "Contoso.Payments.Gateway"
      cloud_role_names:                        # exact, case-insensitive
        - "checkout-api"
        - "contoso-checkout-prod"
      path_hints:                              # repo-relative prefixes seen in stack fileNames
        - "src/Checkout"
        - "src/Payments"
      package_ids:                             # NuGet/npm ids this repo *publishes*
        - "Contoso.Checkout.Client"
      url_patterns:                            # regexes matched against links in the work item
        - "dev\\.azure\\.com/contoso/Commerce/_git/contoso-checkout"
        - "checkout\\.contoso\\.com"
      tags: [commerce, payments, api, dotnet]  # free-text; matched against WI tags/area path
      # JS keys are present-but-empty on a back-end repo. Explicit, not absent:
      # an empty list is a statement, a missing key is a scan that never looked.
      bundles: []
      routes: []
      components: []
      npm_packages: []
      origins: []

  - name: contoso-portal-api                   # the other back-end service
    project: Commerce
    repo_id: 3f7e2b18-1d64-4bb1-9c07-8a5e6f4d2a90
    clone_url: https://dev.azure.com/contoso/Commerce/_git/contoso-portal-api
    default_base_branch: develop               # ← different from contoso-checkout. This is why
    default_pr_target: develop                 #   the picker always asks (§8).
    stacks: [dotnet]
    language: csharp
    owners: [platform@contoso.com]
    build:
      definition_names: ["Portal-API-CI", "Portal-API-Release"]
      build_cmd: "dotnet build Contoso.Portal.sln -c Debug"
      verify: compile_only
      requires_infra: true                     # needs SQL + Redis → static reasoning only
    weight: 1.1                                # this repo is the usual suspect; nudge it
    match:
      assemblies: ["Contoso.Portal", "Contoso.Portal.*", "Contoso.Identity.Client"]
      namespaces: ["Contoso.Portal", "Contoso.Identity.Client"]
      cloud_role_names: ["portal-api", "portal_eus_api"]
      path_hints: ["src/Portal.Api", "src/Portal.Domain"]
      package_ids: []
      url_patterns: ["api\\.portal\\.contoso\\.com"]
      tags: [portal, api, dotnet, b2c]
      bundles: []
      routes: []
      components: []
      npm_packages: []
      origins: ["api.portal.contoso.com"]

  # ── front end (SEPARATE repo — this is the pairing that makes runs multi-repo) ──
  - name: contoso-portal-web
    project: Commerce
    repo_id: 91b0a5c7-7e11-42f6-8a2d-0c9ab7e5d311
    clone_url: https://dev.azure.com/contoso/Commerce/_git/contoso-portal-web
    default_base_branch: develop
    default_pr_target: develop
    stacks: [js]
    language: typescript
    owners: [web-guild@contoso.com]
    build:
      definition_names: ["Portal-Web-CI"]
      build_cmd: "pnpm build"                  # also the fixer's ONLY verification step
      verify: compile_only
      requires_infra: false
      bundler: vite                            # vite | webpack | rollup | esbuild | unknown
      output_dir: dist                         # where bundles and .js.map land
      source_maps: true                        # observed in build config; drives Phase 3 remap
    weight: 1.0
    # Pairing hint: when a full-stack bug resolves this repo, these are its usual
    # back-end counterparts. Purely advisory — it nudges §4.6's grouping, never scores.
    pairs_with: [contoso-portal-api, contoso-checkout]
    match:
      assemblies: []
      namespaces: []
      cloud_role_names: ["portal-web", "portal_eus_web"]
      path_hints: ["src/app", "src/components", "src/pages"]
      package_ids: ["@contoso/portal-web", "@contoso/portal-ui"]
      url_patterns: ["portal\\.contoso\\.com"]
      tags: [portal, frontend, react]
      # ── JS-specific match keys (§4.2 M11–M13) ──
      bundles: ["main", "portal", "checkout-page"]   # bundle base names, hashes stripped
      routes:                                        # router config, parameterised
        - "/orders/:id/checkout"
        - "/orders/:id"
        - "/cart"
        - "/account/*"
      components: ["CheckoutSummary", "CartPage", "OrderDetail"]   # exported component names
      npm_packages: ["@contoso/portal-web", "@contoso/portal-ui"]
      origins: ["portal.contoso.com", "app.contoso.com"]

  - name: contoso-admin-web                    # second front end — makes bundle names ambiguous
    project: Commerce
    repo_id: b81c40a2-9d33-4e57-8f10-77aa3e0c9d14
    clone_url: https://dev.azure.com/contoso/Commerce/_git/contoso-admin-web
    default_base_branch: main
    default_pr_target: main
    stacks: [js]
    language: typescript
    owners: [web-guild@contoso.com]
    build:
      definition_names: ["Admin-Web-CI"]
      build_cmd: "pnpm build"
      verify: compile_only
      requires_infra: false
      bundler: webpack
      output_dir: build
      source_maps: false                       # ← production strips maps: R13 territory
    weight: 1.0
    match:
      assemblies: []
      namespaces: []
      cloud_role_names: ["admin-web"]
      path_hints: ["src/"]
      package_ids: ["@contoso/admin-web"]
      url_patterns: ["admin\\.contoso\\.com"]
      tags: [admin, frontend, react]
      bundles: ["main", "admin"]               # "main" collides with portal-web → conflict-divided
      routes: ["/admin/:section", "/admin/users/:id"]
      components: ["UserTable", "AuditLog"]
      npm_packages: ["@contoso/admin-web"]
      origins: ["admin.contoso.com"]

  - name: contoso-platform-monorepo
    project: Commerce
    repo_id: c40e8d92-55af-4f3c-b7d1-2e6a91c04f78
    clone_url: https://dev.azure.com/contoso/Commerce/_git/contoso-platform-monorepo
    default_base_branch: main
    default_pr_target: main
    stacks: [dotnet, js]                       # genuinely both — the list earns its keep here
    language: csharp
    owners: [platform@contoso.com]
    build:
      definition_names: ["Platform-CI"]
      build_cmd: "dotnet build"
      verify: compile_only
      requires_infra: true
    sparse_paths:                              # monorepo → sparse-checkout these cones only
      - "services/notifications"
      - "services/scheduler"
      - "libs/common"
      - "web/shared-ui"
      - "Directory.Build.props"
    weight: 0.9                                # big and generic → slight penalty
    match:
      assemblies: ["Contoso.Notifications.*", "Contoso.Scheduler.*", "Contoso.Common"]
      namespaces: ["Contoso.Notifications", "Contoso.Scheduler", "Contoso.Common"]
      cloud_role_names: ["notifications-worker", "scheduler-worker"]
      path_hints: ["services/notifications", "services/scheduler", "libs/common"]
      package_ids: ["Contoso.Common"]
      url_patterns: []
      tags: [platform, workers, monorepo]
      bundles: []
      routes: []
      components: []
      npm_packages: ["@contoso/shared-ui"]
      origins: []
```

**What changed from a scalar `stack` to `stacks[]`.** A repo is not one thing: the monorepo above ships a .NET worker and a shared React package, and even a "pure" front-end repo often carries a small .NET BFF. `stacks[]` is read by three consumers — the resolver (which matcher family to run and how to weight it, §4.2), `GitWorkspace` (nothing yet, but it is where a future toolchain-aware prefetch would hook), and Phase 7 (which build command proves a change compiles). `stack: mixed` was a lossy way of saying "don't know"; `stacks: [dotnet, js]` says which two.

### 3.3 Seeding — `lazyboy catalog scan`

```
lazyboy catalog scan [--project Commerce ...] [--depth shallow|full] [--out ~/.lazyboy/repos.yaml.draft]
```

Pipeline, per repo:

1. `GET /{org}/_apis/projects` → `GET /{org}/{project}/_apis/git/repositories` → name, id, `defaultBranch`, `webUrl`, `size`.
2. **Manifest discovery without cloning** — use the Items API to list the tree at the default branch and pull only manifest blobs:
   `GET /{org}/{project}/_apis/git/repositories/{id}/items?recursionLevel=full&versionDescriptor.version={branch}&api-version=7.1`, filtered client-side to `*.csproj`, `*.fsproj`, `Directory.Build.props`, `package.json`, `pyproject.toml`, `setup.cfg`, `*.sln`. `--depth full` fetches each blob; `--depth shallow` only uses filenames (fast, ~90% as good for .NET, where the project filename *is* the assembly name by default).
3. **Inference rules:**

| Source | Rule | Produces |
|---|---|---|
| `Foo.Bar.csproj` | filename stem | `assemblies: [Foo.Bar]`, `namespaces: [Foo.Bar]` |
| `<AssemblyName>X</AssemblyName>` | explicit override wins over filename | `assemblies: [X]` |
| `<RootNamespace>Y</RootNamespace>` | | `namespaces: [Y]` |
| `<PackageId>`/`<IsPackable>true` | | `package_ids: [...]` |
| `package.json` `name` | | `package_ids: [name]`; if scoped `@org/x` also `namespaces: [@org/x]` |
| `pyproject.toml` `[project].name` + `tool.setuptools.packages` / `src/<pkg>` | | `package_ids`, `namespaces` |
| directory of the manifest | dirname relative to repo root | `path_hints: [...]` |
| `GET /{org}/{project}/_apis/build/definitions` where `repository.id == repo.id` | | `build.definition_names` |
| repo `size` > 1 GB **or** ≥ 8 top-level manifest dirs | | `sparse_paths` pre-populated with the manifest dirs, `weight: 0.9` |

4. **Common-prefix collapsing** — if a repo yields `Contoso.Checkout`, `Contoso.Checkout.Api`, `Contoso.Checkout.Domain`, emit the glob `Contoso.Checkout.*` plus the exact base, not 3 literals. Keeps the file readable and the matcher cheap.
5. **Collision report** — any assembly/namespace claimed by >1 repo is printed at the end with a `# CONFLICT:` comment inline in the draft. The scan never silently picks a winner; you do. (A conflicting key that survives into the live catalog is handled at match time — see §4.4.)
6. Output is a `.draft` file plus a unified diff against the current catalog. `lazyboy catalog apply` merges it, **preserving hand edits**: merge is per-repo, per-key, union-with-user-priority; keys under a `# pinned` comment block are never touched.

`cloud_role_names` cannot be inferred from the repo — it's a deploy-time value. `catalog scan --from-telemetry` complements the repo scan by running one KQL over the last 30 days:

```kusto
union requests, exceptions, dependencies
| where timestamp > ago(30d)
| summarize n=count(), assemblies=make_set(column_ifexists("assembly",""), 50) by cloud_RoleName
| order by n desc
```

and proposes `cloud_RoleName → repo` edges wherever an observed assembly already matches exactly one repo. This one query typically fills 80% of `cloud_role_names[]`.

### 3.4 Refresh

- `repos.yaml` is watched (`watchfiles`) and hot-reloaded; a parse error keeps the previous good catalog in memory and raises a UI banner with the line number.
- ADO-side facts (repo id, default branch, ref list, build definitions) are re-fetched on a 24 h TTL into SQLite table `catalog_repo_cache`. A renamed repo is detected by `repo_id` and the UI offers a one-click catalog fix.
- `lazyboy catalog scan --diff-only` is a good weekly cron: it reports *new* repos and *new* assemblies in existing repos without touching the file.

### 3.5 The learning loop (writeback)

Every manual intervention is a labelled training example, and the catalog is the model. Three triggers:

| Trigger | Writeback |
|---|---|
| User manually attaches repo R to a run whose top unmatched evidence was assembly `A` | append `A` (or its `Prefix.*` glob if ≥2 siblings seen) to `R.match.assemblies` |
| User rejects auto-accepted repo R and picks R′, where R matched on `cloud_RoleName = C` | move `C` from `R.match.cloud_role_names` to `R′` |
| Resolver used a Code Search hit and the user accepted it | append the winning file's top-level dir to `R.match.path_hints` and the distinctive symbol's namespace to `R.match.namespaces` |

Implementation: `CatalogWriter.propose(edits) -> CatalogDiff`, rendered in the UI as a YAML diff with a checkbox per hunk, applied with `ruamel.yaml` (round-trip loader) so **comments and ordering survive**. Nothing is written without the click. Every applied edit is recorded in `catalog_edits` (run id, matcher, token, timestamp) so `lazyboy catalog explain <assembly>` can tell you *why* a mapping exists.

---

## 4. The resolver

### 4.1 Inputs

From Phase 3's `BugContext.candidate_symbols`:

```python
class CandidateSymbols(BaseModel):
    assemblies: list[StackAssembly]     # {name, depth, is_framework, method, file_name, line}
    exception_types: list[str]          # "Contoso.Checkout.PaymentDeclinedException"
    method_signatures: list[str]        # "Contoso.Checkout.Services.CardService.Charge(...)"
    cloud_role_names: list[str]
    file_paths: list[str]               # raw fileName values from parsedStack
    package_ids: list[str]              # from dependency telemetry / SystemInfo
    links: list[str]                    # every href found in the work item
    ado_artifacts: list[AdoArtifact]    # {kind: commit|pr|build, repo_id, project, id}
    text_tokens: list[str]              # title + description + repro, tokenized
    application_version: str | None
    area_path: str | None
```

### 4.2 Matchers, in order, with weights

Each matcher emits zero or more `Evidence(repo, matcher, token, base_weight, factor)`. Scores accumulate per repo; **within a matcher only the best hit counts** (no stacking three assemblies from the same repo into a runaway score).

| # | Matcher | Base weight | Modulation | Notes |
|---|---|---|---|---|
| M1 | `exact_assembly` | **0.55** | × `depth_factor` | From `parsedStack[].assembly`. Deepest non-framework frame wins. Denylist applied first. Glob match via `fnmatch` on the catalog side. |
| M2 | `ado_artifact_link` | **0.50** | ×1.0 | `relations[]` `ArtifactLink` to a commit/PR/build gives a `repo_id` — this is a *fact*, not a heuristic. Only reason it isn't 1.0: linked artifacts are often the *previous* fix, not this bug. |
| M3 | `cloud_role_name` | **0.35** | ×1.0 | Exact, case-insensitive. Strong for services, useless for libraries. |
| M4 | `namespace_prefix` | **0.30** | × `prefix_specificity` | From exception type + method signatures. Longest matching prefix across the whole catalog wins; specificity = `len(matched_prefix.split('.')) / len(symbol.split('.'))`. |
| M5 | `path_hint` | **0.25** | × `path_factor` | Normalized stack `fileName` → repo-relative path (§4.3). |
| M6 | `url_pattern` | **0.20** | ×1.0 | Regex over `links[]`. A direct `_git/<repo>` link is upgraded to M2-strength (0.50) since it names the repo outright. |
| M7 | `package_id` | **0.18** | ×1.0 | Dependency telemetry / `SystemInfo` / `packages.lock.json` mentions. |
| M8 | `repo_name_mention` | **0.12** | × `mention_factor` | Catalog repo name (or a ≥8-char distinctive token from it) appearing in title/description/repro. Word-boundary regex, case-insensitive. |
| M9 | `tag_area_match` | **0.08** | ×1.0 | WI tags / area path ∩ `match.tags`. Tiebreaker only. |
| M10 | `code_search` | **0.30** | × `hit_quality` | Only runs as fallback (§5). Never combined with an already-auto-accepted result. |

**Depth factor (M1)** — the deepest frame is where the bug *is*; shallower frames are callers, often in a different repo, and still worth something:

```
depth_factor(i) = 1.0 for the deepest non-framework frame
                  0.75 for the next one
                  0.55, 0.40, then 0.30 floor
```

**Path factor (M5)** — an exact hint prefix match is 1.0; a match on only the first path segment is 0.5; a fuzzy basename-only match (same filename, different dir) is 0.25.

**Mention factor (M8)** — 1.0 in the title, 0.7 in description/repro, 0.4 if the mention is inside a code block or URL already scored by M6 (avoid double-counting).

### 4.3 Build-path normalization (M5)

`parsedStack[].fileName` carries the *build agent's* absolute path. Azure Pipelines hosted agents use `_work/<n>/s` as the sources root; on-prem and older agents vary. Strip everything up to and including the sources root:

```python
BUILD_ROOTS = [
    re.compile(r"^[A-Za-z]:[\\/]a[\\/]\d+[\\/]s[\\/]", re.I),            # D:\a\1\s\
    re.compile(r"^[A-Za-z]:[\\/]_work[\\/]\d+[\\/]s[\\/]", re.I),        # C:\_work\12\s\
    re.compile(r"^/home/vsts/work/\d+/s/", re.I),                        # Linux hosted
    re.compile(r"^/__w/\d+/\w+/", re.I),                                 # containerized
    re.compile(r"^[A-Za-z]:[\\/]agent[\\/]_work[\\/]\d+[\\/]s[\\/]", re.I),
    re.compile(r"^/builds/[^/]+/\d+/", re.I),
]

def normalize_build_path(p: str) -> str | None:
    """D:\\a\\1\\s\\src\\Checkout\\Service.cs -> src/Checkout/Service.cs"""
    if not p:
        return None
    for rx in BUILD_ROOTS:
        m = rx.match(p)
        if m:
            return p[m.end():].replace("\\", "/")
    # No known root: keep the tail after the last plausible repo-ish segment.
    parts = p.replace("\\", "/").split("/")
    for anchor in ("src", "source", "lib", "services", "apps", "packages"):
        if anchor in parts:
            return "/".join(parts[parts.index(anchor):])
    return None            # unusable — do not guess
```

Returning `None` is a first-class outcome; M5 simply contributes nothing rather than manufacturing a bad hint.

### 4.4 Scoring formula

```
raw(R)   = Σ over matchers m of  ( base_weight(m) × best_factor(m, R) )
score(R) = 1 - Π over matchers m of ( 1 - base_weight(m) × best_factor(m, R) )     # noisy-OR
conf(R)  = clamp( score(R) × repo_weight(R) × corroboration(R), 0, 1 )
```

**Noisy-OR, not a sum.** Independent weak signals should reinforce without any single one or any three of them saturating past 1.0. `cloud_RoleName` (0.35) + `namespace` (0.30) + `path_hint` (0.25) → `1 - 0.65·0.70·0.75 = 0.659` — meaningful but below auto-accept, which is right: three medium signals warrant a glance. A deep exact assembly hit alone (0.55) plus a namespace hit (0.30) → `1 - 0.45·0.70 = 0.685`; add the role name → `0.795` → auto-accept.

**Corroboration bonus** rewards *independent kinds* of evidence:

```
corroboration(R) = 1.0 + 0.05 × max(0, distinct_matcher_families(R) - 1)   # capped at 1.15
families: {M1,M4} = symbol · {M2,M6} = link · {M3} = telemetry · {M5} = path · {M7} = package · {M8,M9} = text
```

**Denylist gate**, applied before scoring — a frame whose assembly matches `denylist.assemblies` is marked `is_framework` and skipped by M1/M4 entirely. Consequence: a stack that is *only* `System.*` + `Microsoft.*` produces no M1 evidence at all and heads for the fallback, which is the correct behaviour for e.g. an unhandled `System.NullReferenceException` thrown inside an ASP.NET pipeline with a trimmed stack.

**Catalog conflicts** — when one token matches N repos in the same matcher, that matcher's contribution to each is divided by `N`. Ambiguous keys are self-penalizing, which is what pushes conflicts toward confirmation instead of a coin flip.

### 4.5 Thresholds & the decision table

| Condition | Outcome |
|---|---|
| `conf(top) ≥ 0.72` **and** `conf(top) − conf(2nd) ≥ 0.15` | `auto_accept` — resolver proceeds, UI shows the card as accepted with an undo |
| `conf(top) ≥ 0.72` but margin `< 0.15` | `needs_confirmation` — both shown, user picks one or both |
| `0.35 ≤ conf(top) < 0.72` | `needs_confirmation` |
| `conf(top) < 0.35` **and** Code Search available | run fallback (§5), then re-evaluate this table once |
| `conf(top) < 0.35` after fallback | `blocked_no_repo` (§6) |

Multi-repo is a first-class result, not an exception: **every** candidate with `conf ≥ 0.35` is returned and rendered. A bug that spans `contoso-portal-web` (the 500 the user saw) and `contoso-portal-api` (where it was thrown) should surface both, and the user checks both. Auto-accept applies to the top candidate only; siblings above 0.55 are pre-checked but require the click.

### 4.6 Worked example

Bug 48123 · "Checkout fails with 500 on card payment" · App Insights transaction attached.

```
Evidence extracted (Phase 3):
  parsedStack: [ {assembly: "Contoso.Checkout.Domain", file: "D:\a\1\s\src\Checkout\CardService.cs", level 0},
                 {assembly: "Contoso.Checkout",        level 1},
                 {assembly: "Microsoft.AspNetCore.Mvc.Core", level 2} ]
  exception:   Contoso.Checkout.Domain.PaymentDeclinedException
  cloud_RoleName: "checkout-api"
  links: [portal.contoso.com/orders/9912]
```

| Matcher | Repo | Token | Contribution |
|---|---|---|---|
| M1 | contoso-checkout | `Contoso.Checkout.Domain` → glob `Contoso.Checkout.*`, depth 0 | 0.55 × 1.00 = 0.550 |
| M4 | contoso-checkout | prefix `Contoso.Checkout` of exception type (2/4 segs → 0.5… longest-prefix rule gives specificity 0.60) | 0.30 × 0.60 = 0.180 |
| M3 | contoso-checkout | `checkout-api` | 0.35 |
| M5 | contoso-checkout | `src/Checkout/CardService.cs` → hint `src/Checkout` | 0.25 × 1.00 = 0.250 |
| M6 | contoso-portal-web | `portal.contoso.com` | 0.20 |

```
checkout: 1 − (0.45)(0.82)(0.65)(0.75) = 0.820 ; families {symbol, telemetry, path} = 3 → ×1.10 → 0.902 → AUTO-ACCEPT
portal-web: 1 − (0.80) = 0.200 → below 0.35, dropped (shown collapsed under "weak signals")
```

Margin 0.902 − 0.20 = 0.70 ≫ 0.15. One repo, auto-accepted, four evidence chips.

### 4.7 Resolver code sketch

```python
@dataclass(frozen=True)
class Evidence:
    matcher: str            # "exact_assembly"
    token: str              # "Contoso.Checkout.Domain"
    weight: float           # base × factor, already modulated
    detail: str             # human string for the UI chip tooltip

@dataclass
class RepoCandidate:
    repo: CatalogRepo
    confidence: float
    evidence: list[Evidence]
    resolved_by: Literal["catalog", "heuristic", "code-search", "manual"]

FAMILIES = {"exact_assembly": "symbol", "namespace_prefix": "symbol",
            "ado_artifact_link": "link", "url_pattern": "link",
            "cloud_role_name": "telemetry", "path_hint": "path",
            "package_id": "package", "repo_name_mention": "text",
            "tag_area_match": "text", "code_search": "search"}

class Resolver:
    def __init__(self, catalog: Catalog): self.catalog = catalog

    def resolve(self, sym: CandidateSymbols) -> list[RepoCandidate]:
        buckets: dict[str, dict[str, Evidence]] = defaultdict(dict)   # repo -> matcher -> best
        for ev_repo, ev in chain(
            self._m1_assembly(sym), self._m2_artifact(sym), self._m3_role(sym),
            self._m4_namespace(sym), self._m5_path(sym), self._m6_url(sym),
            self._m7_package(sym), self._m8_mention(sym), self._m9_tags(sym),
        ):
            cur = buckets[ev_repo].get(ev.matcher)
            if cur is None or ev.weight > cur.weight:
                buckets[ev_repo][ev.matcher] = ev

        out = []
        for repo_name, by_matcher in buckets.items():
            repo = self.catalog.repos[repo_name]
            prod = 1.0
            for ev in by_matcher.values():
                prod *= (1.0 - min(ev.weight, 0.95))
            score = 1.0 - prod
            fams = {FAMILIES[m] for m in by_matcher}
            corr = min(1.15, 1.0 + 0.05 * max(0, len(fams) - 1))
            conf = max(0.0, min(1.0, score * repo.weight * corr))
            out.append(RepoCandidate(
                repo=repo, confidence=round(conf, 4),
                evidence=sorted(by_matcher.values(), key=lambda e: -e.weight),
                resolved_by="catalog" if "exact_assembly" in by_matcher else "heuristic",
            ))
        return sorted(out, key=lambda c: -c.confidence)

    def _m1_assembly(self, sym):
        depth_factors = [1.0, 0.75, 0.55, 0.40]
        rank = 0
        for frame in sorted(sym.assemblies, key=lambda f: f.depth):
            if self.catalog.is_denylisted_assembly(frame.name):
                continue
            f = depth_factors[rank] if rank < len(depth_factors) else 0.30
            rank += 1
            hits = self.catalog.repos_matching_assembly(frame.name)   # fnmatch over globs
            share = 1.0 / max(1, len(hits))
            for repo_name, pattern in hits:
                yield repo_name, Evidence(
                    "exact_assembly", frame.name, 0.55 * f * share,
                    f"stack frame #{frame.depth} assembly {frame.name} matches `{pattern}`")
```

Matchers M2–M9 follow the same generator shape; each is independently unit-tested against fixture symbols.

---

## 5. Fallback — ADO Code Search + `repo-scout`

### 5.1 When

Only when `conf(top) < 0.35`, or when `needs_confirmation` fires and the user clicks **"Search the org"**. Never as a first move: it is slow (1 req/s), noisy, and the catalog is right most of the time.

### 5.2 Choosing distinctive queries

Ranked, at most **6 queries per run** (≈6 s at the rate limit):

| Rank | Source | Query form | Why distinctive |
|---|---|---|---|
| 1 | Custom exception type from the stack | `class PaymentDeclinedException` | Declared exactly once in the org |
| 2 | Deepest non-framework method name + type | `CardService Charge` | Two co-occurring identifiers |
| 3 | Verbatim error-message literal ≥ 20 chars, with format placeholders stripped | `"Card was declined by issuer"` | String literals live in exactly the repo that throws them |
| 4 | Normalized stack file path basename | `file:CardService.cs` | Code Search supports `file:` filter |
| 5 | Package id | `PackageReference Contoso.Checkout.Client` | Finds consumers *and* the producer |
| 6 | Distinctive identifier from the WI text | `checkoutSessionToken` | Last resort |

Distinctiveness filter before issuing: token must be ≥ 8 chars, not in a stop list (`Service`, `Manager`, `Handler`, `Controller`, `Helper`, `Utils`, `Client`, `Base`, `Impl`, `Test`), and must not appear in >3 catalog repos' known symbols. Message literals are stripped of interpolation (`{0}`, `{userId}`, `$"..."`) and split at the first placeholder — search the longest static run.

```python
PLACEHOLDER = re.compile(r"\{[^}]*\}|%[sd]|\$\{[^}]*\}")
def literal_fragments(msg: str) -> list[str]:
    return sorted((f.strip(' .,:;"\'') for f in PLACEHOLDER.split(msg)),
                  key=len, reverse=True)[:2]
```

### 5.3 Issuing and scoring

```http
POST https://almsearch.dev.azure.com/{org}/_apis/search/codesearchresults?api-version=7.1
{ "searchText": "class PaymentDeclinedException", "$top": 25,
  "$skip": 0, "includeFacets": false, "filters": {} }
```

Serialized through a shared `AsyncLimiter(1, 1.0)`; results cached per `(query, org)` for the life of the run. 404 → the extension isn't installed; set `code_search_available=false` in the health record and skip permanently (surfaced once in the UI, not per run).

Hit scoring, per repository in `results[].repository.name`:

```
hit_quality(R) = min(1.0,
      0.5 · distinct_query_coverage(R)         # fraction of the ≤6 queries that hit R at all
    + 0.3 · path_plausibility(R)               # best file: 1.0 under src/ ; 0.5 under test/ ; 0.15 under samples|docs|generated
    + 0.2 · concentration(R) )                 # hits_in_R / hits_total, i.e. penalize repos that match everything
```

`code_search` evidence enters the same noisy-OR at base weight 0.30. Because `hit_quality` is at most 1.0, Code Search alone maxes out at `conf = 0.30` — **below the 0.35 confirmation floor by design**. A pure Code Search result therefore *always* asks you. It only auto-accepts when it corroborates catalog evidence, which is exactly the trust level it deserves. A repo returned by Code Search that is absent from the catalog is still rendered (as an "unknown repo" card, `resolved_by="code-search"`), and accepting it triggers catalog writeback.

### 5.4 The `repo-scout` subagent

Deterministic search answers "which repos contain these strings". It does not answer "which of these three repos *owns* the failing behaviour". When ≥3 repos land in `[0.30, 0.55]` — genuine ambiguity, e.g. a shared library plus two consumers — LazyBoy spawns `repo-scout` (declared in the investigator profile, per §5.1 of the master doc):

- **Tools:** `ado_search_code`, `catalog_lookup`, `repo_info`, `ado_get_work_item`, `Read`/`Grep` restricted to already-materialized worktrees. No write tools, no Bash.
- **Budget:** `max_turns=12`, `task_budget` capped, hard 90 s wall clock.
- **Prompt shape:** the ranked candidates with their evidence, the exception + stack, and one instruction — *decide which repo(s) contain the code that must change; cite a file path per repo; if you cannot, say so.*
- **Output:** JSON `{repos: [{name, rationale, cited_paths[], confidence}], insufficient_evidence: bool}` via `output_format`, mapped to `RepoCandidate(resolved_by="code-search")` with `confidence = min(agent_confidence, 0.65)` — a ceiling, because an agent's self-reported confidence is not evidence. Its rationale becomes an evidence chip labelled *scout*.
- `insufficient_evidence: true` short-circuits to `blocked_no_repo` with the agent's reasoning included in the panel — often the most useful sentence in the whole flow ("the stack is entirely framework frames; the only actionable signal is `cloud_RoleName=checkout-api`, which no catalog repo claims").

---

## 6. The blocked path — `blocked_no_repo`

### 6.1 What happens locally

1. `Run.state = blocked_no_repo`; `Run.blocked_reason` = a structured `NoRepoReason{ evidence_found[], evidence_missing[], searched_queries[], code_search_available }`.
2. `RunEvent(type="repo.blocked", payload=NoRepoReason)` → SSE → UI panel.
3. **No ADO write happens automatically.** Per P4, the ADO side is a proposal: a pending `Gate(kind="ado_needs_repo")`.
4. The run stays resumable forever. Attaching a repo later transitions straight to `ready_to_investigate` (the state-machine edge in §4.1 of the master doc) with no re-harvest.

### 6.2 The UI panel

Two columns, deliberately: **what we found** and **what we'd need**.

```
┌─ Cannot identify a repository ─────────────────────────────────────────┐
│ Evidence found                    │ What would resolve this            │
│ • cloud_RoleName  "billing-svc"   │ • A catalog entry claiming         │
│   → no catalog repo claims it     │   cloud_RoleName "billing-svc"     │
│ • 7 stack frames, all framework   │ • A non-framework assembly in the  │
│   (System.*, Microsoft.AspNetCore)│   stack, or a build/commit link    │
│ • No ADO artifact links           │ • Attach the repo manually below   │
│ • Code Search: 4 queries, 0 hits  │                                    │
│   above threshold                 │                                    │
├────────────────────────────────────────────────────────────────────────┤
│ [ Attach repositories manually ▾ ]   [ Mark bug as needs-repo in ADO ]  │
└────────────────────────────────────────────────────────────────────────┘
```

### 6.3 The approval-gated ADO action

Clicking **Mark bug as needs-repo in ADO** opens a preview of the exact payload; approving executes two calls (a JSON-Patch for the tag, a comment for the body — §2.3/§2.5 of the reference doc):

```http
PATCH /{org}/_apis/wit/workitems/48123?api-version=7.1
Content-Type: application/json-patch+json
[ {"op":"test","path":"/rev","value": 14},
  {"op":"add","path":"/fields/System.Tags","value":"triage; lazyboy:needs-repo"} ]
```

The `test` op on `/rev` makes it a compare-and-swap: if someone edited the item since harvest, the patch 409s and LazyBoy re-reads rather than clobbering. Existing tags are read, deduped, and re-sent whole (ADO tags are a `;`-joined scalar, not a list — you must not send only the new tag).

```http
POST /{org}/{project}/_apis/wit/workItems/48123/comments?api-version=7.1-preview.4
{ "text": "<div><p><strong>LazyBoy: repository not identified.</strong></p>
  <p>Evidence found:</p><ul>
    <li>cloud_RoleName <code>billing-svc</code> — not present in the repo catalog</li>
    <li>Stack contains 7 frames, all framework assemblies (System.*, Microsoft.AspNetCore.*)</li>
    <li>No linked commits, PRs or builds on this work item</li>
    <li>Code Search returned no result above threshold for 4 queries</li>
  </ul>
  <p>Please add the implicated repository (a link to a commit, PR, or the repo itself is enough)
     and remove the <code>lazyboy:needs-repo</code> tag.</p></div>" }
```

The comment body is generated from the same `NoRepoReason` the UI renders — one source of truth, no divergence between screen and ADO. Idempotency: if the tag is already present and a `lazyboy:needs-repo` comment exists from the last 24 h, the gate offers "already posted — post again?" instead of duplicating.

### 6.4 Manual attach

The picker is a searchable list over **live org repos** (not just the catalog): `GET /{org}/{project}/_apis/git/repositories` across all projects, cached 24 h, fuzzy-filtered client-side (`fuse.js`), showing project · repo · default branch · catalog-known badge. Multi-select. Selecting one:

1. Creates `RepoCandidate(resolved_by="manual", confidence=1.0, evidence=[Evidence("manual", user_email, 1.0, "attached by you")])`.
2. Opens the base-branch picker (§8) for that repo.
3. Queues a `CatalogDiff` proposal (§3.5) mapping whatever unmatched evidence existed — the assembly, the `cloud_RoleName` — onto that repo, shown as a "Teach LazyBoy this mapping?" card with the YAML diff inline.
4. Transitions the run to `ready_to_investigate`.

Step 3 is the whole point of the phase: **the second time this bug's neighbours show up, they resolve automatically.**

---

## 7. GitWorkspace

### 7.1 Model

```
~/.lazyboy/workspace/<repo-name>/          shared partial clone, one per repo, reused forever
   .git/                                    blobs fetched lazily (--filter=blob:none)
~/.lazyboy/runs/<run-id>/worktrees/<repo>/ per-run worktree, pinned to the run's base branch
```

`git worktree` gives each run an isolated checkout and index while sharing one object store. Second run on a 4 GB monorepo: seconds. The alternative (clone per run) is a non-starter on disk and on time.

**Partial clone (`--filter=blob:none`)** downloads commits and trees but defers blobs until touched. On a large .NET repo this cuts initial clone from minutes to ~15 s; the agent then pays a small on-demand fetch per file it reads. That's the right trade for a tool whose agent reads 30 files out of 8,000. For `stack: node` repos with huge lockfile churn it's an even bigger win.

**Sparse-checkout** applies when `sparse_paths` is non-empty (monorepos). Cone mode, set on the worktree, so the agent's `Grep`/`Glob` never even see `services/unrelated-thing`.

### 7.2 Auth

Per the reference doc §1.3: the token goes in an `http.extraheader` **passed as environment-form git config**, never in the remote URL, never in `git config`, never on the command line (which would leak via `ps`):

```python
def _auth_env(token: str) -> dict[str, str]:
    return {
        "GIT_CONFIG_COUNT": "1",
        "GIT_CONFIG_KEY_0": "http.extraheader",
        "GIT_CONFIG_VALUE_0": f"Authorization: Bearer {token}",
        "GIT_TERMINAL_PROMPT": "0",     # never block on a credential prompt
        "GIT_ASKPASS": "true",
        "GCM_INTERACTIVE": "never",
    }
```

Tokens are re-acquired per command from `CredentialVault` (they expire in ~1h; a long clone can outlive one). The subprocess env is built fresh each call and never logged — the log filter redacts `Authorization:` and `Bearer\s+\S+` shapes.

### 7.3 Lifecycle

| Step | Command | Notes |
|---|---|---|
| ensure clone | `git clone --filter=blob:none --no-checkout <url> <dir>` | `--no-checkout`: the shared clone never has a working tree; only worktrees do |
| refresh | `git fetch --prune --filter=blob:none origin` | skipped if `last_fetch` within `staleness_ttl_minutes` (default 60) |
| list branches | `git for-each-ref --format=%(refname:short) refs/remotes/origin` | feeds the base-branch picker |
| add worktree | `git worktree add --detach <path> origin/<base>` | detached; the fix branch is created in Phase 7 |
| sparse | `git -C <wt> sparse-checkout init --cone` + `set <paths>` | only when `sparse_paths` non-empty |
| checkout | `git -C <wt> checkout` | after sparse config, so only the cone materializes |
| release | `git worktree remove --force <path>` + `git worktree prune` | on run completion/cancel |

### 7.4 Failure handling

| Failure | Detection | Recovery |
|---|---|---|
| Interrupted clone | dir exists, no `.git/HEAD`, or `git rev-parse --git-dir` fails | `rm -rf` the dir and re-clone once; second failure surfaces to the user |
| Stale `index.lock` / `shallow.lock` | git exits with "Unable to create ... File exists" and lock mtime > 10 min | delete the lock, retry once; younger lock → another LazyBoy op is running → serialize on the per-repo `asyncio.Lock` |
| Dirty worktree at release | `git status --porcelain` non-empty | if the run committed nothing, save `git diff` to `runs/<id>/orphaned.patch` before `--force` removal; never silently discard work |
| Worktree dir deleted out from under git | `git worktree list` shows it as `prunable` | `git worktree prune` before every add |
| Auth 401 mid-fetch | stderr contains `Authentication failed` / `TF400813` | re-acquire token, retry once, then raise `CredentialExpired` → UI re-auth prompt (per §7.3 of the master doc, a 401 must not fail the run) |
| Branch vanished server-side | `origin/<base>` missing after fetch | re-open the base-branch picker with a warning |
| Disk full | `ENOSPC` in stderr | run LRU eviction (§7.5) and retry once; if still failing, block with a clear message |

### 7.5 Disk accounting & LRU eviction

`workspace_repo` table: `name, path, bytes, last_used_at, clone_started_at, pinned`. `bytes` is refreshed after each fetch (`du`-equivalent via `os.scandir` walk, cached). Config: `workspace.max_bytes` (default 40 GiB), `workspace.min_free_bytes` (default 10 GiB).

Eviction runs before any clone and after any run completes: while `total > max_bytes` or `free < min_free`, remove the least-recently-used repo that is **not pinned** and has **no live worktrees**. Pinning is a UI toggle (your monorepo stays warm). Eviction is `git worktree prune` then `rm -rf` of the shared clone; it's always safe because the clone is a pure cache.

### 7.6 Code

```python
# src/lazyboy/connectors/git.py
from __future__ import annotations
import asyncio, os, shutil, time
from dataclasses import dataclass
from pathlib import Path

class GitError(RuntimeError):
    def __init__(self, cmd: list[str], code: int, stderr: str):
        super().__init__(f"git {' '.join(cmd)} failed ({code}): {stderr.strip()[:500]}")
        self.cmd, self.code, self.stderr = cmd, code, stderr

class CredentialExpired(GitError): ...

@dataclass
class WorktreeHandle:
    repo: str
    path: Path
    base_branch: str
    base_sha: str

class GitWorkspace:
    """Shared partial clones + per-run worktrees. All calls are async; each repo is
    serialized by its own lock because git's index/refs are not concurrency-safe."""

    def __init__(self, root: Path, vault, events, ttl_minutes: int = 60):
        self.root = root
        self.clones = root / "workspace"
        self.vault = vault                    # CredentialVault -> ado_git_token()
        self.events = events                  # EventBus
        self.ttl = ttl_minutes * 60
        self._locks: dict[str, asyncio.Lock] = {}

    def _lock(self, repo: str) -> asyncio.Lock:
        return self._locks.setdefault(repo, asyncio.Lock())

    # ---------- process plumbing ----------

    async def _git(self, args: list[str], cwd: Path | None = None,
                   auth: bool = False, timeout: float = 900) -> str:
        env = {**os.environ, "GIT_TERMINAL_PROMPT": "0", "GIT_ASKPASS": "true",
               "LC_ALL": "C"}
        if auth:
            token = await self.vault.ado_git_token()
            env |= {"GIT_CONFIG_COUNT": "1",
                    "GIT_CONFIG_KEY_0": "http.extraheader",
                    "GIT_CONFIG_VALUE_0": f"Authorization: Bearer {token}"}
        proc = await asyncio.create_subprocess_exec(
            "git", *args, cwd=str(cwd) if cwd else None, env=env,
            stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
        try:
            out, err = await asyncio.wait_for(proc.communicate(), timeout)
        except asyncio.TimeoutError:
            proc.kill(); await proc.wait()
            raise GitError(args, -1, f"timed out after {timeout}s")
        if proc.returncode != 0:
            e = err.decode(errors="replace")
            if "Authentication failed" in e or "TF400813" in e or "403" in e:
                raise CredentialExpired(args, proc.returncode, e)
            raise GitError(args, proc.returncode, e)
        return out.decode(errors="replace")

    # ---------- shared clone ----------

    async def ensure_clone(self, repo: CatalogRepo) -> Path:
        dest = self.clones / repo.name
        async with self._lock(repo.name):
            if dest.exists() and not await self._is_valid_repo(dest):
                await self.events.emit("git.repair", repo=repo.name,
                                       detail="incomplete clone; re-cloning")
                shutil.rmtree(dest, ignore_errors=True)
            if not dest.exists():
                await self._evict_if_needed(headroom_hint=repo.approx_bytes or 2 << 30)
                dest.parent.mkdir(parents=True, exist_ok=True)
                await self.events.emit("git.clone.start", repo=repo.name)
                await self._git(["clone", "--filter=blob:none", "--no-checkout",
                                 repo.clone_url, str(dest)], auth=True, timeout=1800)
                await self.events.emit("git.clone.done", repo=repo.name)
            else:
                await self._fetch_if_stale(repo, dest)
            self._touch(repo.name)
            return dest

    async def _is_valid_repo(self, path: Path) -> bool:
        try:
            await self._git(["rev-parse", "--git-dir"], cwd=path, timeout=30)
            return True
        except GitError:
            return False

    async def _fetch_if_stale(self, repo: CatalogRepo, dest: Path) -> None:
        stamp = dest / ".git" / "FETCH_HEAD"
        if stamp.exists() and (time.time() - stamp.stat().st_mtime) < self.ttl:
            return
        await self._clear_stale_locks(dest)
        await self.events.emit("git.fetch.start", repo=repo.name)
        await self._git(["fetch", "--prune", "--filter=blob:none", "origin"],
                        cwd=dest, auth=True, timeout=900)
        await self.events.emit("git.fetch.done", repo=repo.name)

    async def _clear_stale_locks(self, dest: Path) -> None:
        for name in ("index.lock", "shallow.lock", "config.lock", "HEAD.lock"):
            p = dest / ".git" / name
            if p.exists() and (time.time() - p.stat().st_mtime) > 600:
                p.unlink(missing_ok=True)
                await self.events.emit("git.lock.cleared", path=str(p))

    # ---------- branches ----------

    async def list_branches(self, repo: CatalogRepo) -> list[str]:
        dest = await self.ensure_clone(repo)
        out = await self._git(
            ["for-each-ref", "--sort=-committerdate",
             "--format=%(refname:strip=3)", "refs/remotes/origin"], cwd=dest)
        return [b for b in out.split() if b and b != "HEAD"]

    async def resolve_sha(self, repo: CatalogRepo, ref: str) -> str:
        dest = self.clones / repo.name
        return (await self._git(["rev-parse", f"origin/{ref}"], cwd=dest)).strip()

    # ---------- worktrees ----------

    async def add_worktree(self, repo: CatalogRepo, run_dir: Path,
                           base_branch: str) -> WorktreeHandle:
        clone = await self.ensure_clone(repo)
        wt = run_dir / "worktrees" / repo.name
        async with self._lock(repo.name):
            await self._git(["worktree", "prune"], cwd=clone)
            if wt.exists():
                await self.remove_worktree(repo, wt, save_dirty_to=run_dir)
            wt.parent.mkdir(parents=True, exist_ok=True)
            args = ["worktree", "add", "--detach"]
            if repo.sparse_paths:
                args.append("--no-checkout")
            args += [str(wt), f"origin/{base_branch}"]
            await self._git(args, cwd=clone, timeout=600)
            if repo.sparse_paths:
                await self._git(["sparse-checkout", "init", "--cone"], cwd=wt)
                await self._git(["sparse-checkout", "set", *repo.sparse_paths], cwd=wt)
                await self._git(["checkout"], cwd=wt, auth=True, timeout=900)
            sha = (await self._git(["rev-parse", "HEAD"], cwd=wt)).strip()
        self._touch(repo.name)
        await self.events.emit("git.worktree.ready", repo=repo.name,
                               path=str(wt), base=base_branch, sha=sha)
        return WorktreeHandle(repo.name, wt, base_branch, sha)

    async def remove_worktree(self, repo: CatalogRepo, wt: Path,
                              save_dirty_to: Path | None = None) -> None:
        clone = self.clones / repo.name
        if wt.exists():
            status = await self._git(["status", "--porcelain"], cwd=wt)
            if status.strip() and save_dirty_to:
                patch = await self._git(["diff", "HEAD"], cwd=wt)
                (save_dirty_to / f"orphaned-{repo.name}.patch").write_text(patch)
                await self.events.emit("git.worktree.dirty_saved", repo=repo.name)
            try:
                await self._git(["worktree", "remove", "--force", str(wt)], cwd=clone)
            except GitError:
                shutil.rmtree(wt, ignore_errors=True)
        await self._git(["worktree", "prune"], cwd=clone)

    # ---------- disk ----------

    def _touch(self, repo: str) -> None:
        self.db.upsert_workspace_use(repo, at=time.time())

    async def _evict_if_needed(self, headroom_hint: int) -> None:
        usage = shutil.disk_usage(self.clones.parent)
        total = self.db.workspace_total_bytes()
        while (total + headroom_hint > self.cfg.max_bytes
               or usage.free - headroom_hint < self.cfg.min_free_bytes):
            victim = self.db.lru_evictable_repo()      # not pinned, no live worktrees
            if victim is None:
                break
            await self.events.emit("git.evict", repo=victim.name, bytes=victim.bytes)
            shutil.rmtree(self.clones / victim.name, ignore_errors=True)
            self.db.drop_workspace_repo(victim.name)
            total -= victim.bytes
            usage = shutil.disk_usage(self.clones.parent)
```

---

## 8. Base branch selection

**Requirement:** LazyBoy never guesses which branch a fix comes from. It *suggests*, you *choose*, per repo.

### 8.1 Suggestion ranking

| Rank | Source | Confidence label | How |
|---|---|---|---|
| 1 | Branch of the most recent related PR/commit on the work item | "used by a linked PR" | `ado_artifacts` → `GET /_apis/git/repositories/{id}/pullrequests/{prId}` → `targetRefName` |
| 2 | Release branch matching the deployed `application_Version` | "matches deployed build 4.7.2" | `release_branch_pattern` regex from the catalog vs. `application_Version` from telemetry, with a `major.minor` fallback when patch doesn't match |
| 3 | Branch of the build that produced the deployed version | "produced build #4.7.2-1934" | `GET /_apis/build/builds?definitions=<id>&buildNumber=<ver>` → `sourceBranch` |
| 4 | Repo default branch | "repo default" | catalog / ADO `defaultBranch` |

The picker preselects rank 1's answer if present, else rank 2, else the default, and **always shows the reason**. All remote branches are listed, sorted by most-recent commit, with a type-ahead filter. A "recently used for this repo" section surfaces the last 5 choices from previous runs — in practice that's what you pick.

### 8.2 Storage

```python
class RunRepo(SQLModel, table=True):          # one row per (run, repo)
    run_id: str = Field(foreign_key="run.id", primary_key=True)
    repo_name: str = Field(primary_key=True)
    repo_id: str                              # ADO GUID, authoritative
    confidence: float
    resolved_by: str                          # catalog|heuristic|code-search|manual
    evidence_json: str
    accepted: bool = False
    base_branch: str | None = None
    base_sha: str | None = None               # pinned at worktree creation — reproducibility
    base_source: str | None = None            # "linked-pr" | "release-match" | "build" | "default" | "manual"
    worktree_path: str | None = None
```

`base_sha` is pinned at worktree creation and never re-resolved: the investigation report cites line numbers that must still mean something when you read it tomorrow. Phase 7 branches `bug/<id>-<slug>` from exactly this sha, and Phase 8 targets `base_branch` for the PR.

---

## 9. REST API

All under the localhost-only server, `Origin`-checked, session-token authenticated.

| Method | Path | Body / Query | Returns |
|---|---|---|---|
| `POST` | `/api/runs/{run_id}/resolve` | `{force_code_search?: bool}` | `202` — starts resolution; progress via SSE |
| `GET` | `/api/runs/{run_id}/candidates` | — | `RepoCandidate[]` with evidence, plus `decision: auto_accept\|needs_confirmation\|blocked` |
| `POST` | `/api/runs/{run_id}/candidates/{repo}/accept` | `{base_branch, base_source}` | `RunRepo`; queues worktree creation |
| `POST` | `/api/runs/{run_id}/candidates/{repo}/reject` | `{reason?}` | `204`; may queue a catalog diff |
| `POST` | `/api/runs/{run_id}/repos/attach` | `{repo_ids: [...], base_branches: {...}}` | manual attach; `RunRepo[]` |
| `GET` | `/api/repos/search` | `?q=&project=&top=50` | live org repos (24 h cache) for the picker |
| `GET` | `/api/repos/{repo}/branches` | `?refresh=false` | branches sorted by recency + `suggested[]` with reasons |
| `POST` | `/api/runs/{run_id}/block` | `{post_to_ado: bool}` | sets `blocked_no_repo`; if `post_to_ado`, opens the gate |
| `GET` | `/api/catalog` | — | parsed catalog + validation status |
| `POST` | `/api/catalog/scan` | `{projects?, depth}` | `202`; streams progress, returns a `CatalogDiff` |
| `GET` | `/api/catalog/diff/{diff_id}` | — | proposed YAML diff, per-hunk |
| `POST` | `/api/catalog/diff/{diff_id}/apply` | `{hunks: [ids]}` | writes `repos.yaml`, hot-reloads |
| `GET` | `/api/workspace` | — | repos on disk: bytes, last used, pinned, live worktrees |
| `POST` | `/api/workspace/{repo}/pin` | `{pinned: bool}` | `204` |
| `DELETE` | `/api/workspace/{repo}` | — | evict now (refused if a worktree is live) |

New `RunEvent` types: `repo.resolve.start`, `repo.evidence` (one per matcher hit, so the UI fills in live), `repo.candidates`, `repo.code_search.query`, `repo.scout.thinking`, `repo.blocked`, `git.clone.start/done`, `git.fetch.start/done`, `git.worktree.ready`, `git.evict`, `catalog.diff.proposed`.

---

## 10. UI — the Repo Resolution panel

Sits between Harvest and Investigate in the Bug Workspace, collapsing to a one-line summary once resolved.

**Candidate card** (one per repo, ranked):

```
┌────────────────────────────────────────────────────────────────┐
│ contoso-checkout          Commerce · main            90%  ✅   │
│ ████████████████████████████████████░░░░                        │
│ [assembly Contoso.Checkout.Domain ·0.55] [role checkout-api ·0.35]
│ [path src/Checkout ·0.25] [namespace Contoso.Checkout ·0.18]    │
│                                                                 │
│ Base branch  [ main ▾ ]   repo default · 12 branches            │
│                            ↳ suggested: release/4.7.0 (matches  │
│                              deployed build 4.7.2)              │
│                                            [Reject]  [Accept]   │
└────────────────────────────────────────────────────────────────┘
```

- **Confidence bar** — color-banded at the thresholds (≥0.72 green, ≥0.35 amber, below grey). The number is always shown; a progress bar without a number is a lie.
- **Evidence chips** — matcher · token · weight contribution. Hover shows `Evidence.detail`; clicking an assembly/path chip scrolls the Phase-3 stack viewer to that frame. This is the trust surface: a user who can see *why* will accept a 0.7 and reject a wrong 0.9.
- **Weak signals** (< 0.35) collapse behind "3 more repos considered" — visible, not hidden, because "why didn't it pick X" is a real question.
- **Base-branch picker** per repo — type-ahead over remote branches, suggestion reasons inline, "recently used" section.
- **Workspace status** — inline progress rows driven by SSE: `cloning… 340 MB`, `fetching…`, `worktree ready (a3f91c2)`. Never a spinner without a byte count.
- **Manual add** — always available, even on a successful auto-accept ("also include…").
- **Catalog-teach card** — appears after any manual correction; shows the YAML diff with per-hunk checkboxes and an "always do this" toggle.
- **Blocked panel** — §6.2.

---

## 11. Tests

### 11.1 Fixture catalog + synthetic stacks

`tests/fixtures/catalog/repos.yaml` — an 8-repo catalog mirroring the shapes in §3.2 (a monorepo, a node app, a python repo, two .NET services sharing a namespace prefix, one repo with a deliberately conflicting assembly claim).

`tests/fixtures/bugs/*.json` — ≥25 `CandidateSymbols` snapshots, each with an `expected` block:

```json
{
  "name": "framework-only-stack",
  "symbols": { "assemblies": [{"name":"System.Private.CoreLib","depth":0}], "cloud_role_names": ["billing-svc"] },
  "expected": { "decision": "blocked", "top": null, "max_confidence_below": 0.35 }
}
```

Cases that must be represented: deep exact assembly hit · framework-only stack · monorepo sparse hit · two repos claiming the same namespace prefix · `cloud_RoleName`-only · ADO artifact link present · path hint with an unusual build root · node stack trace (no assemblies at all) · python traceback · a bug legitimately spanning two repos · a repo name mentioned only in prose · a Code Search-only resolution.

### 11.2 Scoring-regression suite

`pytest tests/test_resolver_regression.py` runs every fixture and asserts, in aggregate:

| Metric | Gate |
|---|---|
| top-1 accuracy (expected repo ranked first) | ≥ 0.90 |
| **false auto-accept** (auto-accepted a repo not in `expected.repos`) | **== 0** |
| blocked-when-should-block recall | ≥ 0.95 |
| mean confidence on correct top-1 | ≥ 0.70 |

Confidences are snapshotted to `tests/fixtures/scores.lock.json`. Any weight change that shifts a score by >0.02 fails until the lock is regenerated *with the diff in the PR* — so weight tuning is a visible, reviewed act rather than a quiet drift. This is the single most valuable test in the codebase.

### 11.3 Other

- **Matcher units** — each of M1–M9 in isolation, including denylist, glob expansion, prefix specificity, and the conflict-division rule.
- **Path normalization** — table-driven over the six build-root regexes plus 10 real-world weird paths, asserting `None` for unusable input.
- **Catalog** — schema validation errors, hot-reload with a broken file (previous catalog retained), round-trip writeback preserving comments and key order (`ruamel.yaml`).
- **Code Search** — `respx` cassettes; rate-limiter asserted at ≥1 s spacing; 404 → graceful degradation; hit scoring table-driven.
- **GitWorkspace** — against a locally-created bare repo over `file://` (no network, no auth): clone, fetch TTL skip, worktree add/remove, sparse-checkout cone contents, dirty-worktree patch rescue, stale-lock clearing (touch a lock with an old mtime), interrupted-clone repair (delete `.git/HEAD`), LRU eviction with a fake size ledger. Auth-env construction is unit-tested for *absence* of the token in `argv`.
- **API** — accept/reject/attach flows, `blocked_no_repo` transition, gate creation, the `/rev` compare-and-swap 409 path.

---

## 12. Risks & mitigations

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | **Catalog rots.** New repos and renamed assemblies silently stop matching. | Resolution quality degrades invisibly. | `catalog scan --diff-only` weekly; the writeback loop; a UI badge when >30 days since last scan; `catalog_edits` telemetry showing which repos need manual help most often. |
| R2 | **Cold-start** — an empty catalog makes Phase 4 useless on day one. | Phase 4 blocks adoption. | `catalog scan` is a prerequisite in the Phase 4 setup flow, not optional. `--from-telemetry` fills `cloud_role_names`. Code Search covers the residual. |
| R3 | **Over-confident wrong repo.** Auto-accept sends the agent into the wrong codebase; it burns budget and produces a confident, wrong report. | Worst possible failure — erodes trust permanently. | The 0.15 margin rule; conflict-division; Code Search capped below auto-accept; false-auto-accept == 0 as a hard CI gate; every auto-accept is undoable in one click with the evidence visible. |
| R4 | **Framework-only stacks** (trimmed release stacks, async state machines, minimal APIs). | Frequent `blocked_no_repo`. | `cloud_RoleName` carries these; the telemetry scan makes that mapping cheap; the blocked panel names the missing signal precisely so the fix is one catalog line. |
| R5 | **Code Search extension not installed.** | Fallback unavailable. | Detected at connect time (404), surfaced once in health, resolution degrades to catalog-only + manual attach. Documented, not a crash. |
| R6 | **Monorepo disk & clone time.** | Slow first run, full disk. | Partial clone + sparse-checkout + shared clone + LRU eviction + pinning; byte-level progress in the UI so a 3-minute first clone doesn't read as a hang. |
| R7 | **Token expiry mid-clone** (~1 h token, 30 min clone). | Failed clone at 90%. | Token acquired per git invocation; `CredentialExpired` classified from stderr and retried once after refresh; long clones aren't restarted from zero because git resumes from the partial pack. |
| R8 | **Concurrent runs on one repo** corrupt the shared clone. | Nasty, hard-to-debug git errors. | Per-repo `asyncio.Lock` around every mutating op; `worktree prune` before add; stale-lock policy keyed on mtime. |
| R9 | **Ambiguity is real** — the bug genuinely spans a library and two consumers. | Forcing a single answer is wrong. | Multi-repo is first-class; `repo-scout` reasons over ambiguity; the fixer operates across multiple worktrees from Phase 7. |
| R10 | **Writeback poisons the catalog** with a one-off correction. | Future mis-resolutions. | Writeback is always a reviewed diff, never automatic; edits are attributed and reversible via `catalog_edits`; globs are only proposed at ≥2 sibling observations. |
| R11 | Prompt injection via work item text steering `repo-scout` to an attacker-chosen repo. | Agent reads the wrong code. | `repo-scout` has read-only tools, no writes, and its output is a *proposal* rendered with evidence; work item text stays inside `<untrusted>` fencing (master doc §7.1). |

---

## 13. Effort

| Item | Estimate |
|---|---|
| Catalog schema, loader, validation, hot-reload | 0.25 d |
| `catalog scan` (repo enumeration + manifest inference + telemetry scan) | 0.5 d |
| Resolver: matchers M1–M9, scoring, thresholds | 0.5 d |
| Code Search fallback + query builder + `repo-scout` wiring | 0.25 d |
| `GitWorkspace` (clone/fetch/worktree/sparse/eviction/failure paths) | 0.5 d |
| Base-branch suggestion + persistence | 0.25 d |
| API + SSE events | 0.25 d |
| UI: candidate cards, evidence chips, branch pickers, blocked panel, catalog-teach | 0.5 d |
| Fixtures + scoring-regression suite + git tests | 0.5 d |
| **Total** | **~2–2.5 days** (master doc budgets 2 d; the extra half-day is the regression suite, and it is worth it) |

**Ship order inside the phase:** catalog + resolver + fixtures first (testable with zero network), then `GitWorkspace` (testable with zero network via `file://`), then the API/UI, then Code Search and `repo-scout` last — the fallback is only meaningful once you can measure how often the deterministic path misses.
