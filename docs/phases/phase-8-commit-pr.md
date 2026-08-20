# Phase 8 — Commit & Pull Request

Part of [LazyBoy Master Design](../LazyBoy-Design.md).

---

## Goal

Take the diff you approved in Phase 7 and turn it into a **commit**, a **pushed branch**, and a **pull request into a target branch you explicitly confirmed** — with the work item linked both ways (mandatory), the target branch's policies read and shown *before* the confirm click, and a rollback that actually undoes things.

Three org conventions are settled (master doc §8, answers 1, 8, 9) and this phase implements them literally:

- **One ADO org, one project.** Every call is project-scoped. There is no project picker and no cross-project link format to handle.
- **The work item must be linked.** `workItemRefs` on the PR *and* `AB#<id>` in the description and commit message. Not a setting.
- **Branch policies must be respected.** The policy configurations API is read for the target ref as a **required pre-flight**, and required reviewers, build validation, minimum approvers and merge strategy appear in the Publish checklist before anything is created.
- **The PR target varies per repo.** The catalog's `default_pr_target` pre-selects it; the human always confirms.

And one thing this phase must not hide: the change is **compile-only, unverified**. Phase 7 ran no tests (master doc §8, answer 4). The commit message, the PR description and the ADO comment all say so explicitly, in their own words, so no reviewer mistakes a green build for a green suite.

This phase is deliberately *not* agentic. Every step here is a deterministic API call or a `git` invocation (P6). Claude contributes exactly two things: the PR title and the PR description, both generated from the already-approved Change Report, both editable before send. Nothing in this phase is a tool the agent can call (P4) — the agent's SDK client is closed before Phase 8 begins.

`changes_ready → committed → pr_created → done`.

---

## Definition of Done

- [ ] `git add` stages **only** the files that survived Diff Review — never `git add -A`, never `git add .`.
- [ ] The commit message follows the configured convention, contains `AB#<work-item-id>` on its own line, and contains a verification line stating the change was **compiled but not tested**.
- [ ] GPG/SSH signing is used when `commit.gpgsign` / `gpg.format` is configured; absence of a signing key is a warning, not a failure.
- [ ] Push uses `-c http.extraheader` auth injection with `--set-upstream`; **`--force` and `--force-with-lease` are never issued** by LazyBoy.
- [ ] A remote branch that already exists is detected before push and handled by explicit user choice (fast-forward, new suffixed branch, or abort).
- [ ] The **target branch is a required user confirmation** per repo, pre-selected from the catalog's `default_pr_target`, validated to exist and to differ from the source.
- [ ] PR created via `POST .../pullrequests?api-version=7.1` with `workItemRefs` **and** `AB#<id>` in the description. Both are **mandatory** — a PR body without `AB#<id>`, or a request body without `workItemRefs`, fails an assertion before the call is made.
- [ ] **Branch policy read is a required pre-flight**, not an optional enrichment: the Publish screen does not render its confirm button until the policy configurations call for every repo's selected target has returned or definitively failed. Surfaced: required reviewers, minimum approver count, build validation definitions, work-item linking, comment resolution, merge strategy.
- [ ] A policy that would block the PR is shown as a **warning**, LazyBoy proceeds anyway, and the warning is recorded in `publish.json` and `audit.jsonl` with what the policy required and what was actually sent.
- [ ] **Not draft.** PRs are created ready for review (`isDraft: false`) in every case; there is no draft toggle.
- [ ] Multi-repo run (the normal case for a full-stack bug) → one PR per repo, one confirm step for all of them, each body cross-linking the siblings, all linked to the same work item, with per-repo partial-failure recovery.
- [ ] The PR link is posted back to the work item as a comment and added as an `ArtifactLink` relation.
- [ ] Every step is idempotent: re-running after a crash detects the existing commit / remote branch / PR and resumes rather than duplicating.
- [ ] Publish UI shows a single checklist screen with everything that will happen, and one confirm button.
- [ ] Rollback: abandon PR, delete remote branch, reset local branch — each available and each logged.
- [ ] A secret scan runs over the staged diff and **blocks** commit on a high-confidence hit.

---

## Design

### 8.1 What happens, in order

```
approve(changes)                     ← Phase 7 gate
  ├─ 1. re-verify working tree matches the reviewed diff (hash of `git diff` == recorded hash)
  ├─ 2. secret scan on the staged content                       [blocking]
  ├─ 3. compose commit message (template + Change Report)       [editable]
  ├─ 4. gate: COMMIT  ────────────────────────────────────► you click
  ├─ 5. git add <reviewed files> ; git commit
  ├─ 6. detect remote branch collision; fetch target; verify target exists
  ├─ 6b. **REQUIRED** read branch policies for <repo, target>            [blocking pre-flight]
  ├─ 7. gate: PUBLISH (target branch, reviewers, policies, what gets posted) ► you click
  ├─ 8. git push --set-upstream origin <branch>
  ├─ 9. POST pullrequests  (per repo)  — workItemRefs mandatory
  ├─ 10. PATCH work item: ArtifactLink relation  +  POST comment with the PR link(s)
  └─ 11. state → done ; run summary + policy warnings written to audit.jsonl
```

Step 6b is **not optional and not lazy**. It runs for every included repo against that repo's selected target, and the Publish screen's confirm button stays disabled until each has resolved to a policy set or to an explicit `unknown`. Moving it after PR creation — or making it a background enrichment that may or may not have arrived — would defeat its only purpose, which is to tell you what will happen *before* you cause it.

Steps 5, 8, 9, 10 are the irreversible ones. 5 and 8–10 sit behind the two gates. There is no third gate between push and PR: once you've said "publish into `release/2026.08` with these reviewers", pushing a branch and opening the PR is one intent, and splitting it just produces orphaned branches when someone walks away mid-flow.

### 8.2 Commit composition

**Message template** (configurable, `config.yaml`):

```yaml
commit:
  template: |
    {type}({scope}): {subject}

    {body}

    {verification_line}

    Fixes AB#{work_item_id}
    {trailers}
  type_from_change_kind: {fix: fix, test: test, refactor: refactor, config: chore}
  scope_from: repo            # repo | top-level-dir | none
  subject_max: 72
  body_wrap: 72
  co_author: ask              # never | always | ask
  signoff: false
  gpgsign: inherit            # inherit git config | true | false
```

Rendered example:

```
fix(portal-api): guard null Address in CheckoutService.Complete

Order.Address is null for carts created before the address-capture change
in #4412. Complete() dereferenced it unconditionally, producing the
NullReferenceException seen in operation 8f2c… (portal_eus_appinsight).

Added a guard that fails fast with a domain exception.

UNVERIFIED: compiled only, tests were not run locally.
  dotnet build src/Portal.sln — exit 0
  dotnet format --verify-no-changes — exit 0
  Tests not run: the suite requires infrastructure unavailable on the
  author's machine. CI (Portal-CI) is the verification gate.

Fixes AB#12345
```

- **Subject** comes from `ChangeSummary.summary`, hard-truncated at 72 chars at a word boundary, imperative mood enforced by the prompt and by a lint (`^[a-z][a-z0-9-]*(\(.+\))?: [a-z]` and no trailing period).
- **Body** is the root cause + what changed, wrapped at 72.
- **The verification line is mandatory and it says the change is unverified.** It is rendered deterministically from `ChangeSummary.verification` (Phase 7 §7.7), never from model prose, and it always begins with the literal token `UNVERIFIED:`. It lists the commands that *did* run with their exit codes, then states plainly that no test executed and why, and names the CI build that will be the first real signal.

  > **Why this belongs in the commit message and not just the PR.** The PR description is read once, during review, and then it is effectively archived. The commit message travels with the change forever — into `git log`, `git blame`, the release notes, and the bisect session eighteen months from now where someone is trying to work out how a regression got in. A reviewer who approves on the strength of a build badge has been misled by omission; a maintainer who later finds `UNVERIFIED: compiled only` in the blame output has been told the truth by the only artefact that outlives the review. A pre-commit assertion fails the commit if the verification line is missing or if it has been edited to drop the `UNVERIFIED:` token. The line is editable — you may add detail — but not removable.

- **`AB#<id>`** must be present; a pre-commit assertion fails the commit if the template was edited to drop it. ADO's autolink turns it into a work-item link and, on completion, can transition the item.

**Co-author trailer — decision.** Default `ask`, and the ask is a single checkbox in the Publish screen, remembered per-org. The argument for including `Co-authored-by: Claude <noreply@anthropic.com>`: honest provenance, and it makes "which of our fixes were AI-assisted?" answerable later with `git log --grep`. The argument against: some orgs' commit linters reject unknown identities, and it can read as diffusion of responsibility when *you* reviewed and are accountable for the diff. LazyBoy's position: **offer it, default it on, let config turn it off org-wide** — but never hide it, because a silent trailer is worse than either choice. If enabled, always also emit `LazyBoy-Run: <run-id>` so a commit can be traced back to its audit trail.

**Single vs multiple commits.** One commit per repo, by default. Rationale: the unit of review here is the PR, ADO squash-merges are common, and a synthetic multi-commit history invented by a tool is noise. Config offers `split: none | by_change_kind` — the latter produces at most two commits per repo (`fix` + `test`), which some teams' policies want. `by_file` is deliberately not offered.

**Scoped staging.** The staged set is exactly `ChangeSummary.files[]` minus anything the human reverted in the Diff Review UI, recomputed at commit time from `git diff --name-only lazyboy/start/<run-id>`:

```bash
git -C <wt> add -- <path1> <path2> …          # explicit pathspecs, never -A / .
git -C <wt> diff --cached --name-only         # assert == expected set, else abort
```

If the working tree contains files outside the expected set (something wrote after review), the commit **aborts** with a diff of the discrepancy. Do not silently include, do not silently drop.

**Signing.** If `git config --get commit.gpgsign` is true, or `gpg.format=ssh` with `user.signingkey` set, LazyBoy passes `-S`. If signing fails (no agent, locked key), it surfaces the gpg stderr and offers *retry after unlocking* / *commit unsigned* / *abort* — never auto-downgrades to unsigned, because a repo with a signing policy will reject the push anyway and the confusing failure would land later.

### 8.3 Pushing

```bash
git -C <wt> \
  -c http.extraheader="Authorization: Bearer $TOKEN" \
  push --set-upstream origin bug/12345-null-ref-in-checkout
```

In practice the header goes through `GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=http.extraheader GIT_CONFIG_VALUE_0=…` env form so the token never appears in `ps` (external-apis §1.3). PAT mode uses `Authorization: Basic <b64(":"+pat)>` in the same header slot.

**Force-push policy: never.** Not `--force`, not `--force-with-lease`, not configurable. LazyBoy only ever creates branches it named itself; if the push is rejected, the correct answer is a new branch, not a rewrite of someone else's history. The UI says so when it offers the alternatives.

**Branch already exists remotely** — detected before push via `git ls-remote --heads origin <branch>`:

| Situation | Offered |
|---|---|
| Remote branch == our base SHA ancestor, ours fast-forwards | *Push (fast-forward)* — default |
| Remote branch has commits we don't have | *Create `bug/<id>-<slug>-2`* / *Abort*. Never force. |
| Remote branch exists **and** an active PR targets it | *Push to it and update the existing PR* (see idempotency) / *New branch* / *Abort* |

**Push rejections** are classified from stderr: `non-fast-forward` → the table above; `TF401027`/`403` → permission (re-auth prompt, check PAT `Code (Write)` scope); `pre-receive hook declined` → the hook text is surfaced verbatim (branch policies can block direct pushes to protected names); `RPC failed; HTTP 413` → suggest splitting the change.

### 8.4 The target-branch prompt

**Per repo, pre-selected from the catalog, always confirmed** — the same discipline as the base-branch prompt in Phase 7, but a *different* decision and worth its own click. The base branch is where the fix was written; the target is where it should land. They're usually the same, and "usually" is exactly why silently assuming it is dangerous.

The PR target **varies per repo** (master doc §8, answer 9): the .NET service may ship from `release/2026.08` while the SPA merges to `main`. `repos.yaml` stores `default_pr_target` per repo; the dropdown opens on that value, marked *unconfirmed* (amber outline), and the confirm button stays disabled until every included repo's target has been explicitly confirmed. Pre-select, never auto-apply — and in a multi-repo run, one confirmation per repo, never one shared dropdown.

Candidates, each labelled:

| Rank | Candidate | Label | Source |
|---|---|---|---|
| 0 | The catalog's `default_pr_target` for this repo | `catalog default` — **pre-selected** | `repos.yaml` |
| 1 | The base branch used in Phase 7 | `you branched from this` | `Run.repos[repo].base` |
| 2 | Repo default branch | `default` | `GET /repositories/{repoId}` → `defaultBranch` |
| 3 | Branches matching `release_conventions` | `release` | `GET /refs?filter=heads/release/` etc., sorted by committer date desc |
| 4 | Branches recently merged into (targets of PRs completed in the last 30 days, top 5 by count) | `team ships here` | `GET /pullrequests?searchCriteria.status=completed&$top=100` |
| 5 | Free text | — | validated on entry |

Validation before the confirm button enables:

- `GET /refs?filter=heads/<target>` returns exactly one ref (else "target branch not found").
- `source != target` (obvious, and cheap to get wrong when someone edits both fields).
- Target is not the source's ancestor-of-nothing case: if `git merge-base --is-ancestor <branch> origin/<target>` is true, the branch is already contained — warn "no changes to merge; did you already publish this?".
- If the target differs from the Phase 7 base, show a divergence preview: `git rev-list --left-right --count origin/<target>...<branch>` and a conflict pre-check via `git merge-tree` (no working-tree mutation). A predicted conflict does not block — it warns and offers *rebase onto target* (interactive-free: `git rebase origin/<target>`, abort on conflict, never auto-resolve).

### 8.5 PR creation

```http
POST https://dev.azure.com/{org}/{project}/_apis/git/repositories/{repoId}/pullrequests?api-version=7.1
Content-Type: application/json

{
  "sourceRefName": "refs/heads/bug/12345-null-ref-in-checkout",
  "targetRefName": "refs/heads/release/2026.08",
  "title": "Bug 12345: guard null Address in CheckoutService.Complete",
  "description": "…generated from the Change Report…\n\nAB#12345",
  "isDraft": false,
  "reviewers": [{"id": "6f1b…-guid", "isRequired": true}, {"id": "a20c…-guid"}],
  "workItemRefs": [{"id": "12345"}],
  "labels": [{"name": "lazyboy"}, {"name": "bugfix"}]
}
```

Response gives `pullRequestId` and `_links.web.href` — the human URL is
`https://dev.azure.com/{org}/{project}/_git/{repo}/pullrequest/{prId}`.

**Title**: `Bug {id}: {ChangeSummary.summary truncated to 100}`. Configurable template; ADO shows ~120 chars before truncating in lists.

**Description**, generated deterministically from the Change Report (Claude wrote the report; the renderer is a template, not a second model call):

```markdown
## What
{ChangeSummary.summary}

## Why
{root_cause_addressed}

## Changes
| File | Kind | +/- | Why |
|---|---|---|---|
| src/Checkout/CheckoutService.cs | fix | +18 −4 | Guard clause … |

## ⚠ UNVERIFIED — compiled only, tests not run
This change has **not been tested**. LazyBoy does not execute test suites (they need
infrastructure that isn't available locally); the compiler is the only objective evidence
that exists.

Evidence that exists:
- `dotnet restore src/Portal.sln` — exit 0 (private ADO Artifacts feed)
- `dotnet build src/Portal.sln` — exit 0
- `dotnet format --verify-no-changes` — exit 0

Evidence that does not exist: no test executed, no behaviour observed, no runtime path
exercised. **`Portal-CI` is the first real signal.**

How to verify by hand: place an order on a cart created before #4412 and confirm the
domain exception surfaces as a 400 rather than a 500 in `portal_eus_appinsight`.

## Risk
low — change is confined to a single method; no public contract change.

## Follow-ups
- Address-capture backfill for carts created before #4412 (not in scope).

---
Related PRs: portal-web !8832
🛠 Generated by LazyBoy run `r_01J…` from AB#12345. Investigation: see the work item discussion.

AB#12345
```

The unverified section is rendered deterministically from `ChangeSummary.verification` (Phase 7 §7.7) — never from model prose — and it is **always present**, because in v1 every change is compile-only. Its content degrades honestly: if `restored: false`, the "evidence that exists" list reads *none — package restore failed against the private ADO Artifacts feed, so nothing was compiled*, and the section heading becomes **⚠ NOT COMPILED, NOT TESTED**. This is the single most important field in the PR for a reviewer's calibration, and it sits above the fold, above Risk.

**Draft: no. Considered and rejected.** PRs are created with `isDraft: false` in every case; there is no draft toggle, no `pr.draft_default` config key, and no auto-forcing to draft on `converged == false` or on a failed build. The tempting design was "unverified work shouldn't page reviewers, so mark it draft" — but the org does not use draft PRs as a convention (master doc §8, answer 8), a draft PR does not trigger build validation policies in ADO, and hiding an unverified change behind draft status makes it *less* visible when visibility is exactly what an unverified change needs. The honest answer is a PR that is plainly labelled unverified in its title-adjacent first section, not one that is quietly parked.

**PR description template: none.** There is no house PR template to conform to (master doc §8, answer 8), so LazyBoy renders its own structure above and does not attempt to detect, fetch or fill a `.azuredevops/pull_request_template.md`. If a repo grows one later, the renderer gains a template-merge step; designing for a template that does not exist would be guessing at its fields.

**Reviewers picker**, sourced in priority order and merged/deduped:

1. Identities required by branch policy on the target (§8.6) — pre-checked, marked `required by policy`, non-removable (removing them does nothing server-side anyway).
2. Code owners: if the repo has a `CODEOWNERS`-equivalent or `repos.yaml` declares `owners:`, matched against the changed paths.
3. Recent contributors to the changed files: `git log -n 50 --format=%ae -- <paths>` in the worktree, top 3 by count, excluding the current user and bot accounts.
4. Free search against `GET https://vssps.dev.azure.com/{org}/_apis/identities?searchFilter=General&filterValue=<q>` (or the Graph API `GET /_apis/graph/users`).

Email → identity GUID resolution is cached in SQLite (`identity_cache`), because it's the slowest lookup in this phase.

**Labels/tags**: ADO PR "labels" are created via the PR create body (`labels`) or `POST .../pullrequests/{id}/labels`. Defaults `lazyboy` + the `type` from the commit convention; configurable, and skipped silently if the org restricts label creation (403 → warn, don't fail the run).

### 8.6 Branch policy awareness

**A required, blocking pre-flight** — not an enrichment (master doc §8, answer 8). Fetched for every included repo against *that repo's confirmed target* before the Publish screen renders its confirm button, so nothing is a surprise:

```http
GET https://dev.azure.com/{org}/{project}/_apis/policy/configurations?api-version=7.1
GET https://dev.azure.com/{org}/{project}/_apis/git/policy/configurations
      ?repositoryId={repoId}&refName=refs/heads/release/2026.08&api-version=7.1-preview.1
```

Interpreted policy types (`type.id` GUIDs are stable and worth hard-coding with names):

| Policy | GUID | Surfaced as |
|---|---|---|
| Minimum number of reviewers | `fa4e907d-c16b-4a4c-9dfe-4d4bd6e0a4b0` | "needs N approvals; self-approval {allowed/blocked}" |
| Required reviewers | `fd2167ab-b0be-447a-8ec8-39368250530e` | pre-checked reviewer chips (path-scoped) |
| Build validation | `0609b952-1397-4640-95ec-e00a01b2c241` | "build `<name>` must pass; {auto-queued/manual}" |
| Work item linking | `40e92b44-2dec-4a11-bc72-cc39c4f2c02e` | ✅ satisfied by `workItemRefs` |
| Comment requirements | `c6a1889d-b943-4856-b76f-9e46bb6b0df2` | "all comments must be resolved" |
| Merge strategy | `fa4e907d-c16b-4a4c-9dfe-4d4bd6e0a4f6` | "squash only" → mentioned in the checklist |
| Status policy | `cbdc66da-9728-4af8-aada-9a5a32e4a226` | external status gate name |

Rendered as a "what will block this PR" list in the Publish screen. Because every change in v1 is compile-only, a **build validation** policy is good news and is framed that way: *"`Portal-CI` must pass — that build is the first real verification this change will get; expect a signal within ~N minutes."* If the repo could not be built or restored locally at all, the line is stronger: *"nothing was verified on your machine; `Portal-CI` is the only check that will run."*

**When a policy would block the PR**, the behaviour is defined and uniform:

| Situation | LazyBoy does |
|---|---|
| Policy requires N approvers, or a required reviewer LazyBoy cannot resolve | Renders an amber **warning** row naming the policy and what is missing. **Proceeds anyway** on confirm. |
| Build validation required | Warning row framed as above. Proceeds. |
| Work item linking required | ✅ satisfied — `workItemRefs` + `AB#<id>` are mandatory, so this policy is never a blocker. |
| Comment resolution / status policy | Warning row: "the PR will open but cannot complete until …". Proceeds. |
| Merge strategy (squash-only, no rebase) | Informational row; affects nothing LazyBoy does. |
| Policy endpoint 403 or unreachable | Rendered as **policies unknown** — stated, never assumed absent. Proceeds, with the unknown recorded. |

LazyBoy never refuses to create a PR because of a policy, and never tries to satisfy one on your behalf. A blocked-but-open PR is a normal, recoverable state; a PR that was silently not created is a surprise you find out about tomorrow. Every warning — the policy name, what it required, and what was actually sent — is written to `publish.json` and `audit.jsonl` and emitted as `publish.policy_warning`, so the record of "we knew and proceeded" survives the run.

A 403 on the policy endpoint (common with narrow PAT scopes) degrades to "policies unknown" rather than blocking — but the Publish screen still waits for that definitive failure before enabling confirm, rather than rendering as if the policies were absent.

### 8.7 Multi-repo runs

**The normal case, not an edge case** (master doc §8, answer 3): the .NET back end and the JS/TS front end live in separate repos, so a full-stack bug produces N branches and N PRs. One branch and one PR per repo. Coordination rules:

- **One confirm step for all of them.** The Publish screen lists every repo with its own target, policies and reviewers, and a single button — `Commit, push & open N PRs`. There is no per-repo approval click; approving the set is one intent, and splitting it strands half a fix.
- All PRs share the same work item, the same `AB#` autolink, and the same `LazyBoy-Run` trailer. **Every PR is linked to the one work item** — `workItemRefs` on each, no exceptions, so the work item's Development section shows the whole change.
- Each PR's target comes from *its own* repo's `default_pr_target`, confirmed separately; the targets are frequently different branch names and must never be coupled.
- **Two-pass description**: PRs are created in a deterministic order (repos sorted by name); after all succeed, each description is PATCHed with the `Related PRs:` line listing the siblings as `repo !prId` (ADO renders `!123` as a PR link within the same project; cross-project uses full URLs).
- **Partial failure**: if repo 2's push fails after repo 1's PR was created, the run enters `publish_partial`. The UI shows per-repo status chips (`committed`, `pushed`, `pr_created`, `failed`) and a **Resume publish** button that retries only the failed repos. Nothing is rolled back automatically — half a fix in review is recoverable; a surprise branch deletion is not.
- The work-item comment is posted **once**, after the last repo, listing every PR. If the run stays partial for more than one attempt, the user can post the comment for what exists.
- `git push` calls are serialized, not parallel: ADO rate limits and the failure story is much clearer.

### 8.8 Linking back to the work item

Two writes, both after all PRs exist:

**ArtifactLink relation** (the structured link that makes the PR appear in the work item's Development section):

```http
PATCH https://dev.azure.com/{org}/_apis/wit/workitems/12345?api-version=7.1
Content-Type: application/json-patch+json

[{
  "op": "add",
  "path": "/relations/-",
  "value": {
    "rel": "ArtifactLink",
    "url": "vstfs:///Git/PullRequestId/{projectId}%2F{repositoryId}%2F{pullRequestId}",
    "attributes": {"name": "Pull Request"}
  }
}]
```

The `vstfs://` URL format matters: `projectId`, `repositoryId` are **GUIDs**, and the separators are URL-encoded `%2F`. Getting this wrong yields a 200 with a link that renders as raw text.

> Note: `workItemRefs` on PR creation usually creates this relation for you. LazyBoy attempts the PATCH anyway and treats "relation already exists" (400 `VS403371` / duplicate) as success — cheaper than a conditional read, and idempotent.

**Comment** (HTML, per external-apis §2.3), posted to the work item:

```html
<div>
  <p>🛠 <b>LazyBoy</b> opened a pull request for this bug.</p>
  <ul>
    <li><a href="https://dev.azure.com/org/proj/_git/portal-api/pullrequest/8831">portal-api !8831</a> → <code>release/2026.08</code></li>
    <li><a href="…/portal-web/pullrequest/8832">portal-web !8832</a> → <code>release/2026.08</code></li>
  </ul>
  <p><b>Summary:</b> guard null Address in CheckoutService.Complete</p>
  <p><b>⚠ Verification:</b> <b>UNVERIFIED — compiled only.</b> <code>dotnet restore</code>,
     <code>dotnet build</code> and <code>dotnet format --verify-no-changes</code> all exit 0.
     <b>No tests were run</b> — LazyBoy does not execute test suites. <code>Portal-CI</code>
     is the verification gate.</p>
</div>
```

Reuses the Phase 6 publisher (markdown → constrained HTML subset, image upload, redaction) — same code path, different template. The redactor runs here too: nothing posted to ADO may contain a token, connection string, or absolute local path.

### 8.9 Idempotency & recovery

Every step records its result in `Run.publish_state` before and after, so a crash mid-flight is recoverable by *detection*, not by assumption:

| Step | Idempotency check | On re-entry |
|---|---|---|
| commit | `git log -1 --format=%B` contains `LazyBoy-Run: <run-id>` | skip; reuse SHA |
| push | `git ls-remote --heads origin <branch>` SHA == local HEAD | skip |
| PR create | `GET /pullrequests?searchCriteria.sourceRefName=refs/heads/<branch>&status=active` | reuse `pullRequestId`; PATCH title/description instead of creating |
| ArtifactLink | duplicate-relation error treated as success | skip |
| comment | search last 20 comments for the `LazyBoy-Run: <run-id>` HTML marker (an invisible `<!-- -->` comment) | skip |

An HTTP call that times out without a response is the dangerous case (did the PR get created?). LazyBoy always follows a timeout with the detection query above before retrying — never a blind retry on a POST that creates something.

### 8.10 Rollback

Available from the run page for as long as the run exists:

| Action | Command / call | Guardrails |
|---|---|---|
| Abandon PR | `PATCH .../pullrequests/{id}` `{"status":"abandoned"}` | Confirm dialog naming the PR; blocked if it already has approvals or comments from someone else ("someone is looking at this — abandon manually") |
| Delete remote branch | `git push origin --delete <branch>` — the one push variant that is *not* a force, and is allowed here because LazyBoy created the branch | Blocked unless the branch tip is the SHA LazyBoy pushed **and** no PR references it (abandon first) |
| Reset local branch | `git reset --hard lazyboy/start/<run-id>` in the worktree | Always available |
| Remove work-item comment | `PATCH .../comments/{commentId}` (soft delete) | Offered; the ArtifactLink relation is removed by index in the same pass |

Rollback is a sequence, and the UI runs it in the safe order: abandon PR → delete branch → reset local → clean up work item. Each step's success/failure is shown independently; a failure stops the sequence rather than continuing blind.

---

## Code

### `src/lazyboy/stages/publish.py` (abridged)

```python
async def commit(run, repo, cfg, ado, git) -> str:
    wt = run.worktree(repo)
    expected = set(run.reviewed_files(repo))

    actual = set(await git.diff_name_only(wt, base=f"lazyboy/start/{run.id}"))
    if actual != expected:
        raise PublishAbort(
            "Working tree no longer matches the reviewed diff",
            unexpected=sorted(actual - expected), missing=sorted(expected - actual),
        )

    hits = scan_secrets(await git.diff(wt, base=f"lazyboy/start/{run.id}"))
    if any(h.confidence == "high" for h in hits):
        raise PublishAbort("Secret detected in diff", hits=hits)   # blocking, no override

    await git.run(wt, ["add", "--"] , *sorted(expected))
    staged = set(await git.run_lines(wt, ["diff", "--cached", "--name-only"]))
    assert staged == expected, (staged, expected)

    msg = render_commit_message(run, repo, cfg)
    assert f"AB#{run.work_item_id}" in msg, "work item link missing from commit message"

    args = ["commit", "-F", "-"]
    if cfg.commit.gpgsign is True or await git.config_bool(wt, "commit.gpgsign"):
        args.append("-S")
    if cfg.commit.signoff:
        args.append("--signoff")
    await git.run(wt, args, stdin=msg)

    sha = await git.rev_parse(wt, "HEAD")
    run.publish_state[repo].commit_sha = sha
    run.emit("publish.committed", repo=repo, sha=sha, files=len(expected))
    return sha


async def push(run, repo, cfg, git) -> None:
    wt, branch = run.worktree(repo), run.branch_name
    remote = await git.ls_remote_head(wt, branch)                     # None | sha
    if remote:
        decision = run.publish_state[repo].collision_decision         # set by the gate
        if decision == "abort":
            raise PublishAbort(f"remote branch {branch} already exists")
        if decision == "new_branch":
            branch = next_available_branch_name(branch)
            await git.run(wt, ["branch", "-m", branch])
            run.branch_name = branch
        elif not await git.is_ancestor(wt, remote, "HEAD"):
            raise PublishAbort("remote branch has commits we don't have; refusing to force-push")

    await git.run_authed(wt, ["push", "--set-upstream", "origin", branch])   # never --force
    run.publish_state[repo].pushed_sha = await git.rev_parse(wt, "HEAD")
    run.emit("publish.pushed", repo=repo, branch=branch)


async def create_pr(run, repo, cfg, ado) -> PullRequest:
    existing = await ado.find_active_pr(repo, source=f"refs/heads/{run.branch_name}")
    if existing:                                        # idempotent re-entry
        await ado.update_pr(repo, existing.id,
                            title=render_pr_title(run, repo),
                            description=render_pr_description(run, repo))
        return existing

    body = {
        "sourceRefName": f"refs/heads/{run.branch_name}",
        "targetRefName": f"refs/heads/{run.publish_state[repo].target_branch}",
        "title": render_pr_title(run, repo),
        "description": render_pr_description(run, repo),
        "isDraft": False,          # never draft — see §8.5; not configurable
        "reviewers": [{"id": g, "isRequired": r} for g, r in run.selected_reviewers(repo)],
        "workItemRefs": [{"id": str(run.work_item_id)}],
        "labels": [{"name": n} for n in cfg.pr.labels],
    }
    # Work-item linking is mandatory (master doc §8, answer 8) — assert before the POST,
    # because a PR created without it has to be fixed by hand afterwards.
    assert body["workItemRefs"] == [{"id": str(run.work_item_id)}], "workItemRefs missing"
    assert f"AB#{run.work_item_id}" in body["description"], "AB# autolink missing from PR body"
    assert "UNVERIFIED" in body["description"], "PR body must state the change is unverified"

    pr = await ado.create_pull_request(repo, body)
    run.publish_state[repo].pr_id = pr.id
    run.emit("publish.pr_created", repo=repo, pr_id=pr.id, url=pr.web_url)
    return pr
```

### Secret scanner (also used in Phase 9 §security)

```python
PATTERNS = [
    ("ado_pat",        r"\b[a-z2-7]{52}\b",                                  "high"),
    ("azure_key",      r"[A-Za-z0-9+/]{86}==",                                "high"),
    ("connstr",        r"(?i)(AccountKey|Password|Pwd)\s*=\s*[^;\s\"']{8,}",  "high"),
    ("jwt",            r"eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}", "high"),
    ("aws",            r"AKIA[0-9A-Z]{16}",                                   "high"),
    ("private_key",    r"-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----",    "high"),
    ("bearer",         r"(?i)bearer\s+[A-Za-z0-9._~+/-]{20,}",                "medium"),
    ("generic_secret", r"(?i)(api[_-]?key|secret|token)\s*[:=]\s*[\"'][^\"']{12,}[\"']", "medium"),
]
# Only added lines (`^\+`, excluding `+++`) are scanned. `medium` warns; `high` blocks.
# Allowlist by regex in config for known-safe test fixtures, matched on the *line*, logged when used.
```

---

## API

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/runs/{id}/publish/preflight` | Everything the Publish screen needs, per repo: reviewed file list, rendered commit message, branch collision status, target-branch candidates with labels, policies for the currently selected target, reviewer suggestions, secret-scan result, divergence/conflict pre-check. |
| `POST` | `/api/runs/{id}/publish/commit-message/preview` | `{overrides}` → re-rendered message + lint result. |
| `PUT` | `/api/runs/{id}/publish/commit-plan` | `{repos:[{repo, message, co_author, sign}]}` → opens/supersedes the `COMMIT` gate with this payload (Phase 6 `GateKeeper`). |
| `POST` | `/api/runs/{id}/commit/execute` | `{gate_id}` → 409 unless the `COMMIT` gate is approved and fresh; performs step 5. |
| `GET` | `/api/runs/{id}/publish/target-options?repo=` | Candidate targets with labels/reasons (§8.4). |
| `GET` | `/api/runs/{id}/publish/policies?repo=&target=` | Interpreted branch policies (§8.6); `{"known": false}` on 403. |
| `GET` | `/api/identities/search?q=` | Reviewer picker (cached). |
| `PUT` | `/api/runs/{id}/publish/pr-plan` | `{repos:[{repo, target, target_confirmed: true, reviewers, labels, collision_decision}], post_comment: true, add_artifact_link: true}` → opens/supersedes the **single** `CREATE_PR` gate covering all repos. 422 unless every repo has `target_confirmed` **and** a resolved (or explicitly `unknown`) policy set. No `draft` field — PRs are never drafts. |
| `POST` | `/api/gates/{gate_id}/decide` | Shared Phase 6 endpoint: `{approve, reason?, payload_sha}`. Editing the commit message or the target changes `payload_sha` and invalidates a stale approval. |
| `POST` | `/api/runs/{id}/pr/execute` | `{gate_id}` → steps 8–10. Idempotent by the gate's `idempotency_key` **and** by the detect-before-create checks in §8.9. |
| `POST` | `/api/runs/{id}/publish/resume` | Retry only the repos in a failed state. |
| `POST` | `/api/runs/{id}/publish/rollback` | `{steps:["abandon_pr","delete_branch","reset_local","remove_comment"], repos:[…]}` |
| `GET` | `/api/runs/{id}/publish/status` | Per-repo chips: commit/push/pr/link state + URLs. |

Events: `publish.preflight_ready`, `publish.committed`, `publish.pushed`, `publish.pr_created`, `publish.linked`, `publish.commented`, `publish.partial`, `publish.rolled_back`, `publish.blocked_secret`, `publish.policy_warning`.

---

## UI

### Publish checklist (one screen, one button)

```
┌ Publish — AB#12345 "Checkout fails for legacy carts" ───────────────────────┐
│                                                                             │
│ ▸ portal-api                                          [ ✎ edit ]            │
│   Files (2)   src/Checkout/CheckoutService.cs  +18 −4                        │
│                src/Checkout/OrderAddress.cs    +6  −0                        │
│   Commit      fix(portal-api): guard null Address in CheckoutService…        │
│               ☑ Co-authored-by: Claude    ☑ Sign (ssh key ~/.ssh/id_ed…)     │
│   Branch      bug/12345-null-ref-in-checkout   (new on origin ✓)             │
│   Target      ┌ release/2026.08  catalog default ────────▾ ┐ ⚠ confirm       │
│               │ release/2026.08   catalog default         │                  │
│               │ release/2026.08   you branched from this  │                  │
│               │ main              default                 │                  │
│               │ develop           team ships here (14 PRs)│                  │
│               └───────────────────────────────────────────┘                  │
│   Policies    ⚠ 2 approvals required · ⚠ build "Portal-CI" must pass         │
│               ✓ work item linking satisfied (AB#12345)                      │
│               → proceeding anyway; warnings recorded in the audit log        │
│   Reviewers   [M. Rossi ✕ required by policy]  [A. Bianchi ✕]  + add…        │
│                                                                             │
│ ▸ portal-web                                          [ ✎ edit ]            │
│   Target      main  catalog default  ⚠ confirm                              │
│   …                                                                          │
│                                                                             │
│ ── Will also post ───────────────────────────────────────────────────────── │
│   ☑ Comment on AB#12345 with both PR links                                  │
│   ☑ Add PR as a linked artifact on the work item                            │
│                                                                             │
│ ✓ Secret scan: clean (312 added lines scanned)                              │
│ ⚠ UNVERIFIED — compiled only, no tests run. Both PR bodies say so.           │
│                                                                             │
│                    [ Cancel ]        [ Commit, push & open 2 PRs ]          │
└─────────────────────────────────────────────────────────────────────────────┘
```

Details that matter:

- The confirm button's label states the exact count and action. Never "Submit".
- The commit message is inline-editable with the lint shown live (subject length, imperative mood, `AB#` present).
- Target dropdown opens **pre-selected on the catalog's `default_pr_target`** for that repo, outlined amber until confirmed; the button stays disabled until every included repo's target has been confirmed *and* its policies have resolved (or definitively failed). A spinner row reads `reading branch policies…` — the button is never enabled while it is showing.
- Policy warnings are informational, never blocking — LazyBoy tells you the PR will need 2 approvals; it doesn't pretend to satisfy them, and it doesn't refuse to open the PR because of them.
- There is no Draft row. PRs are never drafts (§8.5).
- The unverified banner is non-dismissible and states which artefacts will carry it: both PR bodies, both commit messages, and the work-item comment.
- After confirm: a per-repo progress list (`committing… pushed ✓ opening PR…`) driven by SSE, ending in a card with the PR links, a **Copy links** button, and **Open both in browser**.
- On partial failure the same card shows the failed repo in red with the raw error, a **Retry this repo** button, and a **Roll back everything** link.

---

## Tests

| Layer | Test |
|---|---|
| unit | Commit message rendering: templates, 72-char subject truncation at word boundary, body wrap, `AB#` assertion, co-author toggle, and the mandatory `UNVERIFIED:` verification line — including the `restored: false` variant that says nothing was compiled |
| unit | **Mandatory-link assertions**: a PR body without `AB#<id>`, a request body without `workItemRefs`, or a description without `UNVERIFIED` each raise before any HTTP call is made (assert zero requests issued) |
| unit | `isDraft` is `False` in every generated PR body; no code path and no config key can produce `true` (assert over the whole body log) |
| unit | Target pre-selection: `default_pr_target` from the catalog is the initial value; an unconfirmed target makes `pr-plan` 422 |
| unit | Policy interpretation: each GUID maps to its warning row; a blocking policy produces a warning + `publish.policy_warning` event + `publish.json` record, and does **not** prevent PR creation |
| unit | Commit-message lint: rejects trailing period, capitalized subject, missing type, over-length |
| unit | Secret scanner: each pattern hits; only `+` lines scanned; `+++` header ignored; allowlist honoured and logged; a 300 KB diff scans in < 200 ms |
| unit | `vstfs:///Git/PullRequestId/...` URL construction with `%2F` encoding, from GUIDs |
| unit | Target validation: nonexistent ref, source==target, already-merged detection |
| unit | Branch-collision decision matrix → correct action for each of the three cases; force-push flags never appear in any generated argv (assert on the full command log) |
| unit | Push-error classifier: non-fast-forward, TF401027, pre-receive declined, HTTP 413 |
| integration (git) | Scoped staging: an extra untracked file in the tree → abort with discrepancy; reverted file excluded from the commit |
| integration (git) | Signing path with a throwaway GPG key; unsigned fallback requires explicit choice |
| integration (git) | Delete-remote-branch rollback against a local bare "remote" |
| integration (respx) | PR create → 201; idempotent re-entry finds the active PR and PATCHes instead of creating a second |
| integration (respx) | Policy endpoint 403 → `{"known": false}`, screen still renders, confirm enables only after the definitive failure |
| integration (respx) | ArtifactLink duplicate error treated as success |
| integration (respx) | Timeout on PR POST → detection query runs before any retry (assert call order) |
| integration | Multi-repo (the normal case): 2 repos with **different** targets, one confirm step, both PRs linked to the same work item, cross-link PATCH after both succeed; second push fails → `publish_partial`, first PR untouched, resume retries only repo 2 |
| e2e | Golden bug: Phase 7 diff → commit → push to a local bare remote → mocked ADO PR → comment + relation payloads snapshot-asserted |
| manual | One real PR into a scratch repo, verifying `AB#` autolink and the Development section link render correctly in the ADO UI |

---

## Risks & mitigations

| Risk | Mitigation |
|---|---|
| **Wrong target branch** — fix lands on `main` instead of the release train | Per-repo `default_pr_target` from the catalog is pre-selected but **never auto-applied**: confirmation is required per repo, candidates are labelled with *why*, and a divergence + conflict preview is shown. In a multi-repo run the two repos' targets are independent fields, so a shared dropdown can't get one of them silently wrong |
| **Committing files the human reverted** | Staged set recomputed from git at commit time and asserted equal to the reviewed set; abort on any discrepancy; never `git add -A` |
| **Token leaking into git config, remote URL, or `ps`** | `-c http.extraheader` via `GIT_CONFIG_*` env form only; never written to `~/.git-credentials`; log redaction filter; the token never appears in an argv |
| **Secrets in the diff reaching the remote** | Blocking scanner pre-commit on added lines; allowlist requires explicit config and is logged |
| **Force-push destroying work** | Not implemented. No code path constructs `--force`/`--force-with-lease`; a unit test asserts this over the whole command log |
| **Duplicate PRs after a crash or a timeout** | Detect-before-create on every irreversible POST; `LazyBoy-Run` markers in the commit body and an HTML marker in the comment make every artefact self-identifying |
| **Branch policy surprises** (required reviewer, build validation, squash-only) | Policies fetched and rendered before the confirm click; unknown policies stated as unknown rather than assumed absent |
| **Unverified fix looks verified** | Every change is compile-only, so the ⚠ UNVERIFIED section is always present, rendered from `ChangeSummary.verification` rather than model prose, and sits above the fold in the PR body; the `UNVERIFIED:` line is mandatory in the commit message (which outlives the review) and asserted pre-commit; the ADO comment repeats it. Draft status is deliberately *not* used for this — see §8.5 |
| **Branch policy read skipped or raced** | It is a blocking pre-flight: the confirm button does not render until every repo's policy call has returned or definitively failed. A policy that would block is a recorded warning, not a silent proceed and not a refusal |
| **Multi-repo half-publish** | Explicit `publish_partial` state, per-repo chips, resume-only-failed, no automatic rollback |
| **PR spam on a work item** | Comment posted once, after the last repo, with a marker checked on re-entry |
| **Stale branch** — target moved during review | `merge-tree` conflict pre-check at preflight; offer a clean rebase (abort on conflict) rather than pushing something that won't merge |
| **Reviewer picker suggests people who left** | Identity lookups validate the GUID is active; contributors filtered by last-90-days activity; failures to resolve are dropped with a note rather than sending an invalid GUID (ADO 400s the whole PR create otherwise) |

---

## Effort

| Work | Estimate |
|---|---|
| Commit composition: templates, lint, scoped staging, signing, secret scanner | 0.3 d |
| Push: auth injection, collision detection & decisions, error classifier | 0.2 d |
| Target-branch prompt: candidates, validation, divergence/conflict preview | 0.2 d |
| PR creation, description renderer, reviewers picker + identity cache, labels | 0.3 d |
| Branch policy pre-flight: fetch, interpretation, blocking-warning surface & recording | 0.2 d |
| Work-item linking (ArtifactLink + comment, reusing Phase 6 publisher) | 0.1 d |
| Multi-repo orchestration, idempotency, partial-recovery, rollback | 0.25 d |
| Publish UI checklist + progress + result card | 0.3 d |
| Tests | 0.2 d |
| **Total** | **≈ 2 days** (master doc says 1–1.5; the blocking policy pre-flight, idempotency and multi-repo-as-normal are what push it) |
