# Phase 9 — Hardening, Cost & Packaging

Part of [LazyBoy Master Design](../LazyBoy-Design.md).

---

## Goal

Turn nine phases of working code into something you can install with one command, trust with your credentials, hand to a colleague, and debug at 18:00 on a Friday. Phase 9 is continuous — it starts the day Phase 0 lands — but this doc defines what "v1 done" means.

Five deliverables: a **complete audit trail** you can export, **usage that can't run away** (turns and tokens against the rate-limit window — LazyBoy runs on a Claude subscription, so there is no dollar meter, see [master §5.4](../LazyBoy-Design.md#54-budget-control)), **reliability** that survives a laptop lid closing mid-run, a **security posture** mapped threat-by-control, and a **`uvx lazyboy` one-liner** that works on Windows, macOS and Linux.

---

## Definition of Done

- [ ] Every run produces a complete, schema-versioned `audit.jsonl`; `lazyboy export <run-id>` produces a single shareable zip with secrets redacted.
- [ ] `max_turns` and `task_budget` enforced per run; a run that hits its turn ceiling degrades gracefully (keeps artefacts, marks `budget_exhausted`) instead of erroring. No dollar budget exists on a subscription credential.
- [ ] Usage dashboard: turns, tokens (input / output / cache-read) and rate-limit events per run, per stage, per model, per work item, with a 30-day trend and a configurable alert threshold.
- [ ] Credential detection at startup (`claude` CLI session vs `ANTHROPIC_API_KEY`) drives `usage.mode: turns|dollars`; a 429 pauses and resumes a run rather than failing it.
- [ ] Every external call has a defined retry policy; a circuit breaker opens after repeated failures and the UI says which dependency is down.
- [ ] Every stage resumes after a crash from the last good state, including mid-agent (via `resume=<session_id>`).
- [ ] Orphaned worktrees and abandoned runs are detected and cleaned on startup and on demand.
- [ ] SQLite integrity is checked at startup; a corrupt DB is quarantined, not silently truncated.
- [ ] Security checklist complete: every threat in the table has a named control with a test.
- [ ] Secret scanner runs on the diff before commit **and** on anything posted to ADO **and** on log output — and is treated as a blocking control, because with no test execution nothing else in the pipeline would ever catch a committed credential.
- [ ] Retention: run TTL, `lazyboy prune`, workspace LRU eviction, all with a dry-run mode.
- [ ] `uvx lazyboy` from a clean machine → first-run wizard → connected → inbox, in under 3 minutes.
- [ ] `/api/diagnostics` + a "copy diagnostics" button producing a paste-ready, redacted report.
- [ ] Golden-bug e2e green in CI — a **multi-repo (.NET + JS/TS), compile-only** scenario; repo-resolver regression suite green; manual test script executed once per release.
- [ ] README, one-page quickstart, and the catalog authoring guide written.

---

## Design

### 9.1 The audit trail

`runs/<run-id>/audit.jsonl` — append-only, one JSON object per line, `fsync`ed on gate decisions and irreversible actions (everything else is buffered; a lost line about a `Read` is acceptable, a lost line about a push is not).

```jsonc
{"v":1,"seq":412,"ts":"2026-08-19T14:22:31.884Z","run_id":"r_01J…","stage":"fix",
 "kind":"tool.pre","tool":"Edit","session_id":"sess_…","agent":"main",
 "input_digest":"sha256:9f2c…","input":{"file_path":"portal-api/src/…/CheckoutService.cs"},
 "decision":"allow","policy":"path_jail:ok,denylist:ok"}

{"v":1,"seq":413,"ts":"…","kind":"tool.post","tool":"Edit","duration_ms":38,
 "result":"ok","bytes_written":412,"file_sha_before":"…","file_sha_after":"…"}

{"v":1,"seq":501,"kind":"permission.denied","tool":"Bash","input":{"command":"git push"},
 "reason":"'git push' is reserved for LazyBoy"}

{"v":1,"seq":640,"kind":"gate","gate":"publish","state":"approved","actor":"local-user",
 "payload_digest":"sha256:…","payload":{"repos":[{"repo":"portal-api","target":"release/2026.08"}]}}

{"v":1,"seq":641,"kind":"usage","stage":"fix","model":"claude-sonnet-…","turns":37,
 "input_tokens":184201,"output_tokens":9822,"cache_read_tokens":151002,
 "usd":null,"limit_hit":false}

{"v":1,"seq":700,"kind":"external","service":"ado","op":"POST /pullrequests",
 "status":201,"duration_ms":712,"attempt":1,"correlation_id":"…"}
```

Event kinds: `run.*`, `stage.*`, `tool.pre`, `tool.post`, `permission.denied`, `gate`, `agent.message`, `usage`, `external`, `git`, `error`. (`usage` is the SSE `cost.updated` payload written verbatim; `usd` is `null` under a subscription credential.) `input` values are redacted through the same filter as logs; anything over 4 KB is replaced by a digest plus a pointer to a blob in `runs/<id>/blobs/<sha>`.

**A full run record** (what the export contains):

```
lazyboy-run-r_01J….zip
├── manifest.json          # lazyboy version, python version, OS, org, run id, timestamps, redaction map version
├── audit.jsonl
├── events.jsonl           # the RunEvent stream (SSE replay source)
├── run.json               # Run row + state transitions with timestamps + turn/token usage
├── context.json           # BugContext (attachments listed, optionally included)
├── transaction.json       # App Insights trace
├── findings.json / investigation.md
├── change-report.md / changes.patch / diffstat.json
├── build-probe.log        # restore + compile output per repo; no test logs — tests are never run
├── publish.json           # commit shas, branches, PR ids/urls, policies seen
└── config-snapshot.yaml   # effective config with every secret field elided
```

`lazyboy export <run-id> [--with-attachments] [--redact-paths]` builds it. `--redact-paths` rewrites the home directory and repo absolute paths to `<HOME>` / `<REPO>` so the zip can go into a ticket. Redaction is applied on **write**, not on read, so a leaked zip is a leaked *redacted* zip.

### 9.2 Usage & budget

LazyBoy authenticates to Claude with a **subscription** (`claude login`), not an API key, so **there is no per-run dollar figure and `max_budget_usd` is not set, stored, or displayed anywhere**. The scarce resource is the rate-limit window. Usage per run comes from the SDK's result messages (turn counts, token counts including cache reads) plus per-subagent attribution from `task_budget` accounting. Persisted to a `usage_entry` table: `(run_id, stage, agent, model, turns, input, output, cache_read, cache_write, usd_nullable, limit_hits, ts)`.

**Credential detection.** At startup `CredentialVault` reports which Claude credential is live: a `claude` CLI session (subscription) or `ANTHROPIC_API_KEY` (metered). That derives one flag, `usage.mode: turns|dollars`, read by exactly two places — the meter component and profile construction. Under `turns` (the default and the shipped configuration), any `total_cost_usd` the SDK returns is recorded on the row but never rendered; under `dollars` the same meter grows a cost chip and a per-run ceiling becomes settable. Switching to an API key later is therefore a config change, not a redesign. `lazyboy doctor` prints the detected mode, because a meter showing the wrong unit is worse than no meter.

**Enforcement, layered:**

| Control | Where | Default |
|---|---|---|
| `max_turns` | `ClaudeAgentOptions` per stage client | investigate 60, fix 80 — the primary runaway guard |
| `task_budget` | subagent fan-out cap | `{"total": 12}`; fan-out is where turns multiply silently |
| `effort` routing | per stage | `high` investigate, `medium` fix — the cheap lever, not a limit |
| Per-run turn ceiling | LazyBoy sums stages; blocks a new stage that would exceed | 180 turns |
| Daily turn/token soft ceiling | across runs, from `usage_entry` | 60% of the observed rate-limit window → warn, 85% → block new runs (override in the UI, logged) |

Hitting a turn ceiling is a **normal outcome**, not an error: the SDK ends the turn, LazyBoy captures whatever artefacts exist, marks the run `budget_exhausted`, and the UI offers *add 20 turns and resume* (uses `resume=<session_id>`, so the context is not repaid) or *accept partial*.

**Rate limits (429) pause, they do not fail.** On a 429 or an SDK capacity error the runtime keeps the client and the session id, emits `cost.updated` with `limit_hit: true` and `limit_resets_at` parsed from `Retry-After` / rate-limit headers, moves the run to `paused_rate_limited`, and retries with full jitter until the window reopens — then emits `limit_hit: false` and the transcript continues in place. Only a wait exceeding `limits.rate_limit_max_wait` (default 30 min) surfaces as a failure, and even then the run lands resumable rather than `failed`. `fallback_model` covers a capacity error on one model; it does not help when the window itself is spent, which is why the meter carries cumulative per-work-item history — the point is to see an expensive bug before the day is gone, not after.

**Model routing by stage** — opinionated defaults, all overridable in `config.yaml`:

| Stage | Model | Effort | Why |
|---|---|---|---|
| Harvest / Resolve heuristics | none (deterministic) | — | P6 |
| `repo-scout` subagent | haiku | low | search + shortlist, high volume |
| Investigate (main) | opus | high | judgement-heavy, one shot, the whole value proposition |
| `log-analyst` subagent | sonnet | medium | KQL + trace reading |
| Fix (main) | sonnet | medium | edit loops; escalate to opus on `revise` ×2 |
| `build-runner` | haiku | low | runs restore + compile, parses output. Never runs tests |
| `reviewer` | sonnet | high | adversarial reading, worth the effort tier |
| PR text | template (no model) | — | P6 |

Escalation rule: if the fix loop's reviewer returns `revise` twice on the same file set, the next iteration re-runs with `model="opus"` and `effort="high"` and a note in the event stream — one deliberate escalation, not an open-ended climb.

**Usage dashboard** (`/usage`): a stacked bar of **turns per day by stage**; a token-volume chart beside it (input / output / cache-read, cache-read called out because it is most of the volume); a table of the 20 most turn-expensive runs with work-item title, stage split, and outcome (`done` / `abandoned` / `budget_exhausted`); turns and tokens per work item including re-runs; cache-hit ratio (cache_read / total input) as a health metric — a falling ratio usually means prompt churn; and a **rate-limit timeline** marking every `limit_hit` with how long the run waited, which is the single best predictor of a bad afternoon. Alerts: a toast when a run passes 75% of its turn ceiling, a banner when the daily usage estimate is within 20% of the rate-limit window. A dollar figure appears in exactly one case: `usage.mode == "dollars"`.

### 9.3 Performance

| Metric | Target | How |
|---|---|---|
| Cold start (`uvx lazyboy` → inbox rendered) | < 4 s warm wheel cache, < 25 s first-ever | lazy imports of `azure.*` and the SDK; SPA served from disk; DB migrations are cheap and versioned |
| Worktree creation (repo already cloned) | < 3 s p50 | shared clone + `git worktree add`; `--no-checkout` then checkout so sparse settings apply |
| Clone cache hit rate | > 90% after week 1 | one clone per repo in `workspace/`, refreshed with `git fetch --prune` not re-cloned; measured and shown in diagnostics |
| App Insights query cache | > 60% hit within a run | keyed on `(resource_id, kql_hash, timespan_bucket)`, stored in SQLite with a 24 h TTL; the transaction query is fetched once and reused by `appi_get_transaction` |
| ADO work-item cache | 60 s freshness on the inbox | ETag / `If-None-Match` where ADO supplies it, else timestamp-based |
| SPA bundle | < 350 KB gzipped initial | route-level code splitting; Shiki languages loaded on demand (only the languages present in the diff); no icon-font, no moment.js; `vite build --report` checked in CI with a size budget that fails the build |
| SSE reconnect | < 1 s, zero missed events | `?after=<seq>` replay from SQLite (Phase 0) |
| Diff render (2 000-line file) | < 150 ms | virtualized hunk list; highlight only the visible window |

A `bench/` directory holds three reproducible benchmarks (worktree creation on a 4 GB repo fixture, 1 000-event SSE replay, 5 000-line diff render) run manually per release, not in CI.

### 9.4 Reliability

**Retry matrix** — all via `tenacity`, all with full jitter, all logging attempt number to `audit.jsonl`:

| Call | Retry on | Attempts | Backoff | Notes |
|---|---|---|---|---|
| ADO REST GET | 429, 500, 502, 503, 504, connect/read timeout | 5 | respect `Retry-After`, else 1 s × 2ⁿ, cap 30 s | idempotent |
| ADO REST POST/PATCH (mutating) | 429, 503, connect timeout **only** | 3 | as above | never retry on read timeout without a detection query first (Phase 8 §8.9) |
| ADO auth (401) | — | 0 | — | triggers re-auth prompt, run pauses, not fails |
| Log Analytics query | 429, 503, throttling | 4 | 2 s × 2ⁿ, cap 60 s | shrink timespan on `PartialError`, then retry once |
| Code Search | 429 | 3 | 1 s fixed + 200 ms pacing | degrade to catalog-only on repeated failure |
| `git fetch/clone/push` | exit 128 with transient network text | 3 | 2 s × 2ⁿ | never retry a push that may have partially succeeded — `ls-remote` first |
| Claude Agent SDK | process exit ≠ 0, broken stdio | 2 | 3 s | resume with `resume=<session_id>`, never re-seed from scratch |
| Keyring | backend locked | 1 | — | prompt the user, don't loop |

**Circuit breaker** per external service: 5 failures in 60 s opens it for 60 s; while open, calls fail fast with a typed `DependencyDown` and the UI shows a persistent banner naming the service ("Azure DevOps is not responding — retrying at 14:31"). Half-open probes with a single cheap GET. Runs in a stage that needs the down dependency go to `paused_dependency`, not `failed`.

**Resume-after-crash, per stage:**

| Stage | Resume mechanism |
|---|---|
| Harvest | Idempotent re-fetch; attachments checked by size+sha before re-download |
| Resolve | Cached candidates in SQLite; re-run only if the catalog changed |
| Investigate / Fix | `AgentSession.session_id` → `ClaudeAgentOptions(resume=…)`; SDK replays its own context. `fork_session=True` when the user wants a variant rather than a continuation. Checkpoints from `enable_file_checkpointing` survive the crash and are re-listed. |
| Publish | `publish_state` per repo + detect-before-create (Phase 8 §8.9) |

On startup, any run in a non-terminal, non-paused state is moved to `interrupted` with a banner offering *resume* or *abandon*. Nothing auto-resumes without a click — an unattended agent restarting on boot is not a behaviour anyone asked for.

**Orphaned worktree cleanup.** On startup and via `lazyboy gc`: `git worktree list --porcelain` per clone, cross-referenced against runs in the DB. Worktrees with no run, or belonging to a run older than the TTL, are pruned (`git worktree remove --force` then `git worktree prune`). Local `lazyboy/start/<run-id>` tags for dead runs are deleted. Dry-run by default in `gc`; automatic only for the unambiguous "no such run" case.

**DB integrity on startup.** `PRAGMA quick_check` (fast) always; `PRAGMA integrity_check` when the previous shutdown was unclean. On failure: rename to `lazyboy.db.corrupt-<ts>`, start fresh, and tell the user where the old file is and that runs on disk are still readable. WAL mode on, `synchronous=NORMAL`, `busy_timeout=5000`, `foreign_keys=ON`. A single writer (one process) makes this comfortable; the `runs/` directory is the durable record and the DB is reconstructible index-plus-metadata, which is the reason to be relaxed here.

### 9.5 Security hardening review

Threat → control → test. This table is the review artefact; each row must be demonstrably true.

| # | Threat | Control | Test |
|---|---|---|---|
| S1 | Another local app / a webpage drives the API | Bind `127.0.0.1` only; `Origin`/`Host` allowlist; per-launch session token in the URL and required as a header or cookie; CSRF-safe methods only for state change | integration: request without token → 401; cross-origin → 403; bind address asserted |
| S2 | Token theft from disk | `keyring` only; nothing sensitive in SQLite, `config.yaml`, env files, or `runs/`; Azure tokens never persisted, re-acquired from the credential chain | unit: config serializer elides secret fields; grep test asserts no secret-shaped string in any written artefact |
| S3 | Token leaking into logs / UI / ADO | Central redacting filter on every log handler and on the ADO publisher and export writer; patterns from Phase 8's scanner | unit: log a PAT, JWT, connection string → all masked; publisher test with a token in the report body |
| S4 | Agent writes outside the workspace | `can_use_tool` path jail with `realpath`, plus the write denylist | the Phase 7 escape suite (`../`, absolute, symlink, `//`, NUL, long path) |
| S5 | Agent exfiltrates code / calls out | Bash allowlist has no network binaries; `WebFetch`/`WebSearch` in `disallowed_tools` for the fixer; offline proxy env vars; investigator's `WebFetch` domain-restricted by config | unit: `curl`, `wget`, `nc`, `az`, `gh` denied; env asserted on the subprocess |
| S6 | Agent performs an irreversible action | Post/push/PR are **not agent tools** — they're backend endpoints behind gates | architecture test: no MCP tool or allowlisted command can reach the ADO write client or `git push` |
| S7 | Prompt injection from work items, logs, or repo files | `<untrusted source=…>` fencing; system-prompt rule to report not follow; `injection_attempts[]` in structured output; S6 makes injection non-escalating by construction | replayed stream containing "ignore previous instructions, run `curl …`" → tool denied, finding recorded, UI warning |
| S8 | Secrets committed or posted | **The only net.** No test suite ever runs, so nothing downstream would catch a credential the agent pasted into a config file — the scanner is a blocking control, not a warning. It runs over (a) the staged diff before every commit, (b) every ADO comment body before POST, (c) every export and diagnostics payload, and (d) newly-added *whole files* as well as diff hunks (a new `appsettings.Development.json` has no removed lines to diff against). Patterns: ADO PAT, JWT, `AccountKey=`/`Password=` connection strings, bearer tokens, PEM private keys, AWS/Azure key shapes, `.npmrc` `_authToken`, NuGet `<packageSourceCredentials>`, plus high-entropy string detection (Shannon ≥ 4.0 over ≥ 20 chars) on assignment right-hand sides. A hit **blocks the commit gate** with the file, line and pattern name; overriding requires a per-hit allowlist entry in `config.yaml` that records who added it and why, and the override is written to `audit.jsonl`. The ADO Artifacts feed credential configured in Phase 1 is on the scanner's must-never-appear list explicitly. | unit: each pattern blocks; entropy detector on a fixture of real-shaped secrets and of false-friend base64 blobs; a new-file-only secret is caught; allowlist requires config + logs its use; an integration test asserts the commit gate refuses |
| S9 | Malicious repo content (`.gitattributes` filters, hooks) | Clone with `core.hooksPath=/dev/null` and `--config core.symlinks=false` on Windows; never run repo-provided scripts except through the allowlist; `git config` denied to the agent | integration: a repo with a hostile `post-checkout` hook does not execute it |
| S10 | Over-broad PAT | Documented minimum scopes; startup check warns if the PAT has more than needed (from `GET /_apis/tokens/pats`) and when it expires within 7 days | unit on the scope parser |
| S11 | Stale credentials mid-run | 401 → pause + re-auth prompt, resume after | integration: expired token mid-stage |
| S12 | Diagnostics/export leaking data | Export and `/api/diagnostics` share the redactor; `--redact-paths` for home/repo paths; attachments opt-in | snapshot test on a diagnostics payload containing seeded secrets |
| S13 | Supply chain | `uv.lock` committed; `uv pip compile --generate-hashes`; `pip-audit` in CI; the SPA has zero runtime CDN dependencies | CI job |

Also: `lazyboy doctor --security` prints this table with a live pass/fail per row (bind address, token presence, keyring backend, denylist loaded, offline env, scanner patterns compiled).

### 9.6 Disk & data retention

The org is **one ADO project holding 30–100 repositories** ([master §8](../LazyBoy-Design.md#8-environment-constraints-answered) constraints 1 and 7), and full-stack bugs check out two or more of them per run. Cloning everything is not viable and the working set is what matters, so eviction policy is a first-class setting rather than a footnote.

```yaml
retention:
  run_ttl_days: 30              # runs/<id> pruned after this, unless pinned
  keep_terminal_artifacts: true # keep report/patch/audit even when pruning worktrees
  workspace_max_gb: 60          # LRU eviction of workspace/<repo> clones (30-100 repos, one project)
  workspace_min_keep: 8         # never evict below this many repos — the hot set is bigger than it looks
  workspace_protect_pairs: true # never evict half of a repo pair that ran together (front end + back end)
  attachments_max_mb_per_run: 200
```

- Worktrees are pruned aggressively (they're cheap to recreate); reports, patches and `audit.jsonl` are kept for the full TTL and can be **pinned** per run from the UI (a pin survives all pruning).
- `lazyboy prune [--dry-run] [--older-than 30d] [--include-workspace]` prints exactly what it would delete with sizes, then does it. `--dry-run` is the default when run interactively without flags.
- Workspace LRU uses `last_used_at` from the DB, not filesystem mtime (which git operations churn). Because a run touches N repos, `last_used_at` is stamped on **every** repo in the run, not just the primary — otherwise the front-end clone is evicted between full-stack bugs and re-cloned on every one.
- `workspace_protect_pairs` keeps co-occurring repos together: eviction scores a repo down if it has appeared in a run alongside a repo that is still hot. Evicting one half of a .NET/JS pair guarantees a slow next run.
- Clones are `--filter=blob:none` partial clones, so 30–100 repos is tens of GB, not hundreds; the guard is still needed because a couple of large monorepos dominate the total. Diagnostics shows per-repo size and last use so an eviction is explicable.
- A disk-space guard checks free space before clone/worktree; below 5 GB it refuses to start a run and points at `lazyboy prune`.

### 9.7 Packaging & distribution

**Build** (`pyproject.toml`, hatchling, `uv`-managed):

```toml
[project]
name = "lazyboy"
requires-python = ">=3.12"
dependencies = [
  "fastapi", "uvicorn[standard]", "sqlmodel", "httpx[http2]", "pyyaml",
  "claude-agent-sdk", "azure-identity", "azure-monitor-query",
  "keyring", "tenacity", "typer", "rich", "jsonschema",
]
[project.scripts]
lazyboy = "lazyboy.main:app"
[tool.hatch.build.targets.wheel]
packages = ["src/lazyboy"]
artifacts = ["src/lazyboy/static/**"]      # the built SPA ships inside the wheel
```

```bash
cd web && npm ci && npm run build          # → ../src/lazyboy/static
uv build                                   # → dist/lazyboy-0.9.0-py3-none-any.whl
uv publish --index internal                # or: gh release upload v0.9.0 dist/*
```

The SPA is built into the wheel by a `hatch` build hook so there is no Node dependency at install time. Wheel target: < 8 MB.

**Install / run:**

```bash
uvx lazyboy                                       # public/internal index
uvx --from git+https://dev.azure.com/... lazyboy  # direct from ADO
uvx --index-url https://pkgs.dev.azure.com/{org}/_packaging/{feed}/pypi/simple/ lazyboy
uv tool install lazyboy && lazyboy                # persistent install
```

`lazyboy` (no subcommand) = start the server, pick a free port (default 7717, probe upward), open the browser at `http://127.0.0.1:<port>/?t=<session-token>`. Subcommands: `doctor`, `export`, `prune`, `gc`, `catalog validate`, `config path`, `version`.

**First-run wizard** (in the SPA, not the terminal — the terminal is for the one command):
1. ADO org URL + connectivity probe.
2. Auth: *Use my `az login`* (probes `AzureCliCredential` immediately and shows the resolved identity) / *Sign in in a browser* / *Paste a PAT* (with the scope table and an expiry field).
3. App Insights resource(s) — discovered via ARM if the identity allows, else pasted portal URL, parsed by the §3.1 decoder.
4. Repo catalog — *start empty* / *import a YAML* / *auto-seed from the org's repo list* (names + default branches only).
5. Model defaults and turn ceilings (no dollar budget — the wizard says why, and which Claude credential it detected).
6. A health-check screen: five green ticks, and a **Run the golden bug** button for a no-cost smoke test.

Config lands in `~/.lazyboy/config.yaml` (`%USERPROFILE%\.lazyboy` on Windows); secrets in the keyring.

**Self-update check**: once per day, a HEAD/GET against the index's JSON for the latest version; a subtle "0.9.1 available — `uv tool upgrade lazyboy`" chip in the footer. No auto-update, no background download. Disable with `updates.check: false`.

**Platform notes:**

| Platform | Issue | Handling |
|---|---|---|
| Windows | **MAX_PATH 260** — deep monorepos + `runs/<id>/worktrees/<repo>/` blow past it | Short run ids (`r_` + 12 chars); worktrees under `%LOCALAPPDATA%\lazyboy\w\<short>` with a junction from the run dir; startup check for `git config core.longpaths=true` and for the registry `LongPathsEnabled`, with a fix-it button that prints the exact commands |
| Windows | Line endings — CRLF churn producing enormous diffs | Never set `core.autocrlf` globally; per-clone `core.autocrlf=false` and rely on the repo's `.gitattributes`; the diff generator asserts that a change with only EOL differences is dropped, and warns if > 50% of hunks are EOL-only |
| Windows | Keyring backend = Windows Credential Manager (fine), but 2 560-byte value limit | PATs fit; long tokens are chunked with a documented `key/1`, `key/2` scheme |
| Windows | Symlinks off by default | `core.symlinks=false`; the path jail resolves junctions too (`realpath` handles them) |
| macOS | Keychain prompts on every access from a new binary path | Store with a stable service name `lazyboy`; document "Always Allow"; `uv tool install` (stable path) recommended over `uvx` for daily use |
| macOS | Gatekeeper — not applicable (pure Python, no binary) | — |
| Linux | `keyring` may fall back to a plaintext backend if no Secret Service | Detect at startup; refuse the plaintext backend by default with a clear message and an explicit `--allow-insecure-keyring` opt-in; suggest `keyring.alt` / `gnome-keyring` / `kwallet` |
| All | `git` not installed / < 2.35 (needed for `worktree` ergonomics and `merge-tree --write-tree`) | Startup version check with a clear message |
| All | Python 3.12 requirement | `uv` handles it; `requires-python` fails loudly rather than half-installing |

### 9.8 Observability

- **Structured logs**: `structlog` JSON to `~/.lazyboy/logs/lazyboy-<date>.jsonl`, rotated at 20 MB × 5, human-readable Rich output to the console. Every line carries `run_id`, `stage`, `correlation_id`. The redacting processor is the **first** processor in the chain, not the last.
- **OpenTelemetry (optional)**: `observability.otel: file | otlp | off` (default `off`). `file` writes spans to `~/.lazyboy/logs/traces.jsonl` — no collector, no network. Spans: stage, each external call, each tool call, each git command. This exists for post-mortems on "why did that run take 11 minutes", and the file backend means it costs nothing to leave on.
- **`GET /api/diagnostics`** returns a single redacted document: versions (lazyboy, python, git, uv, OS), config with secrets elided, keyring backend, auth mode + token expiry (date only), connectivity probe results for ADO/Log Analytics/Claude, DB size + row counts + integrity result, workspace size + repo count + cache hit rates, last 20 errors, active circuit breakers, current run states, disk free.
- **"Copy diagnostics"** button in the UI footer → the same document as markdown on the clipboard, ready to paste into a ticket or a chat. This is the single highest-leverage support feature in the product; it is worth building on day one, not day ninety.

### 9.9 QA plan

**The golden bug** — `tests/fixtures/golden/`:

- `workitem.json`: a realistic ADO work item with an HTML description, a repro-steps field containing an App Insights portal URL (in the awkward doubly-encoded form), and an image attachment.
- `appinsights/`: canned responses for the `itemId` lookup and the transaction query, including an `exceptions.details` payload with a `parsedStack` pointing at `Portal.Checkout.CheckoutService.Complete`.
- `repos/`: **two** seeded repositories, because a full-stack bug spanning separate .NET and JS/TS repos is the normal case, not an edge case — `golden-api/`, a tiny .NET service with a genuine null-dereference on the checkout path (plus a `packages.lock.json` and a `nuget.config` pointing at a **fake local Artifacts feed**, so the restore path is exercised rather than assumed), and `golden-web/`, a small Vite + TypeScript front end that calls it and surfaces the failure in browser telemetry (with its own `.npmrc` against the same fake feed). The seeded defect requires a change in **both** repos, so the fixture covers per-repo base branches, per-repo builds, and two PRs against one work item.
- `expected/`: snapshot of the resolver's candidates (both repos, with evidence), the per-repo commit messages, both PR payloads, and the ADO comment HTML.

**The fixture is compile-only.** It asserts `dotnet restore` + `dotnet build` in `golden-api` and `npm ci` + `npm run build` + `npx tsc --noEmit` in `golden-web`, and it asserts that **no test runner was invoked** — the audit trail must contain a `permission.denied` for any attempt, and `ChangeSummary.verification.tests_run` must be `false`. A second variant, `golden-restore-broken/`, points at an unreachable feed and asserts the honest degradation: `restored: false`, `compiled` not claimed, `unverified_reason: "package restore failed"`, and the Change Report banner carrying it.

Run modes: `--replay` (canned SDK message streams, zero API usage, runs in CI on every push) and `--live` (real Claude, run manually before a release; asserts on outcomes — both repos build, `verification.tests_run == false`, `unverified` stamped, ≤ 3 files changed per repo — not on exact text).

**Repo-resolver regression suite** — the resolver is the component most likely to silently degrade. A table-driven corpus of ≥ 40 cases: `(stack frames, cloud_RoleName, assembly names, work-item text) → expected repo(s)`. Metrics reported per run: top-1 accuracy, recall@3, `blocked_no_repo` rate. CI fails if top-1 drops more than 2 points below the committed baseline. New production misses get added as cases — that's the feedback loop that keeps Phase 4 honest.

**Manual test script** (`docs/qa/manual.md`, ~20 minutes, once per release): fresh install on a clean profile; wizard with `az login`; wizard with a PAT; inbox loads; harvest a real bug with an attachment; a bug with no App Insights link; a bug the resolver misses → `blocked_no_repo` + tag posted; investigation + post to ADO; fix on a chosen base; revert a hunk; ask-for-changes; publish to a scratch repo; verify PR links and the Development section; roll back; close the laptop mid-run and resume; `lazyboy export`; `lazyboy prune --dry-run`.

### 9.10 Documentation deliverables

| Doc | Audience | Contents |
|---|---|---|
| `README.md` | anyone who lands on the repo | what it is in 5 lines, a 20-second demo GIF, `uvx lazyboy`, the four approval gates, the security posture in a paragraph, links |
| `docs/quickstart.md` | you, on a new laptop | one page: install, auth, first bug, the four gates, where things live on disk, the three commands worth knowing (`doctor`, `export`, `prune`) |
| `docs/catalog.md` | whoever maintains `repos.yaml` | the schema, all resolution keys (`assemblies`, `cloud_role_names`, `namespaces`, `path_hints`), `restore_cmd`/`build_cmd`/`typecheck_cmd`/`lint_cmd` (no test command — LazyBoy never runs tests), `owners`, worked examples, `lazyboy catalog validate`, and how to add a case to the resolver regression suite when it gets something wrong |
| `docs/troubleshooting.md` | future you at 18:00 | the twelve failure modes seen in testing, each with the diagnostic line to grep for |
| `CHANGELOG.md` | everyone | kept from v0.1 |

---

## Code

### Redacting log processor (used by logs, export, diagnostics, ADO publisher)

```python
import re
from typing import Any

_PATTERNS = [
    (re.compile(r"\b[a-z2-7]{52}\b"), "«ado-pat»"),
    (re.compile(r"eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}"), "«jwt»"),
    (re.compile(r"(?i)(AccountKey|Password|Pwd)\s*=\s*[^;\s\"']{8,}"), r"\1=«redacted»"),
    (re.compile(r"(?i)bearer\s+[A-Za-z0-9._~+/-]{20,}"), "Bearer «redacted»"),
    (re.compile(r"-----BEGIN[^-]*PRIVATE KEY-----[\s\S]+?-----END[^-]*-----"), "«private-key»"),
]
_SECRET_KEYS = {"pat", "token", "password", "secret", "authorization", "client_secret", "extraheader"}


def redact(value: Any, *, home: str | None = None) -> Any:
    if isinstance(value, str):
        for pat, repl in _PATTERNS:
            value = pat.sub(repl, value)
        if home:
            value = value.replace(home, "<HOME>")
        return value
    if isinstance(value, dict):
        return {k: ("«redacted»" if k.lower() in _SECRET_KEYS else redact(v, home=home))
                for k, v in value.items()}
    if isinstance(value, (list, tuple)):
        return type(value)(redact(v, home=home) for v in value)
    return value


def structlog_redactor(logger, name, event_dict):     # FIRST in the processor chain
    return redact(event_dict)
```

### Circuit breaker

```python
class Breaker:
    def __init__(self, name, threshold=5, window=60.0, cooldown=60.0):
        self.name, self.threshold, self.window, self.cooldown = name, threshold, window, cooldown
        self.failures: deque[float] = deque()
        self.opened_at: float | None = None

    def check(self):
        now = time.monotonic()
        if self.opened_at:
            if now - self.opened_at < self.cooldown:
                raise DependencyDown(self.name, retry_at=self.opened_at + self.cooldown)
            self.opened_at = None            # half-open: let one through
        return True

    def record(self, ok: bool):
        now = time.monotonic()
        if ok:
            self.failures.clear(); self.opened_at = None; return
        self.failures.append(now)
        while self.failures and now - self.failures[0] > self.window:
            self.failures.popleft()
        if len(self.failures) >= self.threshold:
            self.opened_at = now
            EVENTS.publish("dependency.down", service=self.name, cooldown=self.cooldown)
```

### CLI surface (`typer`)

```
lazyboy                       start the server and open the browser
lazyboy doctor [--security]   environment + connectivity + security checklist
lazyboy export <run-id> [--with-attachments] [--redact-paths] [-o FILE]
lazyboy prune [--dry-run] [--older-than 30d] [--include-workspace]
lazyboy gc [--dry-run]        orphaned worktrees, dangling tags, stale caches
lazyboy catalog validate [path]
lazyboy config path | lazyboy version | lazyboy logs --tail
```

---

## API

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/diagnostics` | The redacted diagnostics document (§9.8). |
| `GET` | `/api/diagnostics/markdown` | Same, formatted for pasting. |
| `GET` | `/api/health` | Per-dependency status + breaker state. Used by the wizard's tick list. |
| `GET` | `/api/usage?from=&to=&group_by=stage\|model\|work_item` | Usage dashboard data: turns, tokens, rate-limit events. `usd` present only when `usage.mode == "dollars"`. |
| `GET` | `/api/usage/limits` / `PUT` | Read/update turn ceilings and the daily usage warning thresholds. No dollar field under a subscription. |
| `POST` | `/api/runs/{id}/export` | Build the zip; returns a download URL. |
| `POST` | `/api/maintenance/prune` | `{dry_run, older_than_days, include_workspace}` → the plan or the result. |
| `POST` | `/api/maintenance/gc` | Orphaned worktree cleanup. |
| `GET` | `/api/updates` | Latest available version. |
| `POST` | `/api/runs/{id}/resume` | Resume an `interrupted` / `paused_dependency` / `paused_rate_limited` / `budget_exhausted` run. |

---

## UI

- **Footer status bar** (always visible): connection ticks for ADO / Azure Monitor / Claude, the current run's **turn + token meter** (`turn 37 / 80` with a thin progress bar that turns amber at 75% and red at 90%, plus a token chip `184k in · 10k out · 151k cached`), a rate-limit chip that appears only when `limit_hit` is live and counts down to `limit_resets_at`, the version chip with the update hint, and **Copy diagnostics**. A dollar figure is shown only under `usage.mode == "dollars"`.
- **Usage page**: stacked daily bar of turns by stage, token-volume chart, turn-expensive-runs table, turns/tokens per work item, cache-hit-ratio sparkline, rate-limit timeline, turn-ceiling controls.
- **Maintenance page**: disk usage by directory (runs / workspace / attachments / logs) as a treemap, run list with pin toggles and sizes, `prune`/`gc` with a dry-run preview you approve.
- **Banners**: dependency down (with the retry countdown), turn budget exhausted (with *add 20 turns & resume*), rate-limited (with the reset countdown and a note that the run is paused, not lost), interrupted runs found on startup, insecure keyring backend, git too old, low disk, PAT expiring within 7 days.
- **Wizard**: five steps, each independently re-runnable later from Settings; ends on the health-check screen with the golden-bug smoke test button.

---

## Tests

| Layer | Test |
|---|---|
| unit | Redactor: every pattern, nested dicts/lists, secret key names, `<HOME>` rewriting, idempotence (redacting twice changes nothing) |
| unit | Audit writer: schema version, ordering by `seq`, oversized payload → blob + digest, `fsync` on gate/irreversible kinds |
| unit | Export: manifest completeness, every secret-seeded field elided, zip opens, deterministic ordering |
| unit | Usage accounting: turns and cache-read tokens counted separately; subagent turns attributed to the parent run; daily ceiling arithmetic across timezones; `usd` stays `None` under `usage.mode == "turns"` |
| unit | Credential detection: `claude` CLI session → `turns`; `ANTHROPIC_API_KEY` → `dollars`; both present → subscription wins and `doctor` says so |
| unit | Secret scanner: every pattern, the entropy detector, a secret in a **newly added file**, `.npmrc` `_authToken`, the Artifacts feed credential; an override requires config and is audited |
| unit | Breaker: opens at threshold, fails fast while open, half-open probe, closes on success |
| unit | Retry matrix: table-driven — each status code either retries the expected number of times or doesn't retry at all; mutating POSTs never retry a read timeout |
| unit | Slug/path length on Windows: synthesized 300-char paths → the junction strategy keeps every path < 260 |
| unit | Retention planner: TTL, pins, workspace LRU with `min_keep`, dry-run produces no filesystem calls |
| integration | Corrupt DB → quarantined with a timestamped name, fresh DB created, runs still listed from disk |
| integration | Kill the process mid-fix → restart → run shows `interrupted` → resume uses `resume=<session_id>` (asserted on the options passed to the SDK) and does not re-seed |
| integration | Orphaned worktree (run deleted from DB) → `gc --dry-run` lists it, `gc` removes it, clone untouched |
| integration | Turn budget exhausted mid-fix → artefacts present, state `budget_exhausted`, resume with a raised turn ceiling continues the same session |
| integration | 429 mid-investigation → run enters `paused_rate_limited`, `cost.updated{limit_hit:true}` emitted, retry after the window resumes the same session; a wait past `rate_limit_max_wait` leaves the run resumable, not `failed` |
| security | The S1–S13 table, each row with at least one automated test; `doctor --security` asserted to report all-pass on a clean install |
| perf | Bundle-size budget in CI (fails over 350 KB gz); worktree benchmark recorded per release |
| e2e | Golden bug `--replay` on every push (multi-repo .NET + JS, compile-only; asserts no test runner was invoked and `tests_run == false`); the broken-restore variant asserts `unverified_reason: "package restore failed"`; `--live` before each release |
| regression | Repo-resolver corpus with committed baseline metrics |
| manual | `docs/qa/manual.md`, once per release, on all three OSes if available |

---

## Risks & mitigations

| Risk | Mitigation |
|---|---|
| Hardening is never "done" and slips forever | This doc's DoD is the v1 cut line; everything else goes to the v2 backlog below |
| Redaction misses a novel token shape | Defence in depth: secrets never persisted in the first place (S2), keys elided by *name* as well as by pattern, and the export is opt-in per run |
| Windows path length bites a real monorepo late | Short ids + junction strategy + a startup check, implemented in Phase 0's directory layout, not retrofitted here |
| Rate-limit exhaustion mid-afternoon | Turns are the budget: per-stage `max_turns`, `task_budget` on fan-out, a per-run turn ceiling, a daily usage warning, and cumulative per-work-item history so an expensive bug is visible early. A 429 pauses and resumes rather than failing. If someone later switches to an API key, credential detection flips `usage.mode` to `dollars` and the same meter renders cost — the UI always says which mode it is in, so the number is never misleading |
| A credential is committed and nothing catches it | With no test execution the secret scanner is the only automated gate before a push (S8): it blocks the commit, covers whole new files as well as diff hunks, includes entropy detection, and its overrides are audited |
| Golden-bug fixture rots as prompts change | `--replay` asserts on *structure and outcomes*, not on model prose; the `--live` run is the one allowed to be judgement-based |
| Diagnostics become a data-exfiltration vector | Same redactor, path rewriting, attachments opt-in, and the payload is shown to the user before it's copied |
| SQLite corruption on a hard power loss | WAL + `synchronous=NORMAL` + startup integrity check + `runs/` on disk as the durable record |

---

## Effort

| Work | Estimate |
|---|---|
| Audit schema, blob spill, export command | 0.25 d |
| Usage accounting (turns/tokens), turn ceilings, credential detection, 429 pause/resume, dashboard | 0.35 d |
| Retry matrix, circuit breaker, resume-after-crash wiring, gc, DB integrity | 0.4 d |
| Security review pass + the S1–S13 test suite | 0.3 d |
| Retention, prune, LRU, disk guard | 0.15 d |
| Packaging, build hook, first-run wizard, platform fixes, self-update | 0.4 d |
| Observability: structlog, OTel-to-file, diagnostics endpoint + copy button | 0.2 d |
| QA: multi-repo compile-only golden bug fixture (incl. the fake Artifacts feed and the broken-restore variant), resolver corpus, manual script | 0.4 d |
| Docs: README, quickstart, catalog guide, troubleshooting | 0.2 d |
| **Total** | **≈ 2.5 days** for the v1 cut, then ongoing |

---

## v2 backlog

Deliberately out of scope for v1 (master doc §1 non-goals), ordered by expected value per day of work:

| # | Item | Sketch |
|---|---|---|
| 1 | **Batch triage of the whole inbox** | Run harvest + resolve + a cheap "triage" investigation (haiku, capped at ~4 turns) across every assigned bug overnight; the inbox gains a confidence/complexity/blast-radius column and a suggested order of attack. Highest leverage item on this list: it changes LazyBoy from a tool you point at a bug into a tool that tells you which bug to point it at. |
| 2 | **Watch App Insights for new exception clusters → open work items** | A scheduled KQL (`exceptions | summarize by problemId, bin(timestamp,1h)`) detecting a `problemId` unseen in 30 days above a rate threshold; propose a new ADO bug pre-filled with the transaction link, the parsed stack, the blast radius, and the resolved repo — behind an approval gate, obviously. Turns the tool from reactive to proactive. |
| 3 | **Learning from accepted/rejected fixes** | Every hunk revert, every "ask for changes" message, and every PR comment on a LazyBoy PR is a labelled signal. Store them; distil per-repo conventions into a `repos.yaml` `guidance:` block injected into the fixer's system prompt append. Explicitly *not* fine-tuning — a curated, human-visible, editable preferences file is more useful and far cheaper. |
| 4 | **Run tests once the infrastructure is solvable** | The v1 constraint is environmental, not philosophical: suites need databases and downstream services that do not exist on this laptop ([master §8](../LazyBoy-Design.md#8-environment-constraints-answered) constraint 4). The unlock is **Testcontainers** (or a per-run ephemeral environment) driving the dependencies a suite actually needs, discovered per repo and recorded in the catalog. Shape: a `test_cmd` + `test_infra` block in `repos.yaml`, the denylist relaxed to an allowlist for repos that declare working infra, `build-runner` regaining a test mode, `ChangeSummary.verification.tests_run` finally able to be `true`, and the ⚠ banner dropping off reports that genuinely ran tests. Highest-value correctness item on this list — it is the difference between "compiles and reads correctly" and "verified". |
| 5 | **Consume front-end source maps** | Browser exceptions arrive minified, so JS stack frames resolve to `main.4f2a.js:1:88214` and the resolver falls back to text heuristics for the front-end half of every full-stack bug. If the build publishes source maps (master §8 constraint 11 — verify during Phase 3), fetch or read them, map frame → original file + line, and feed real paths into `affected_files`, the catalog matcher and the fixer's context. Also lets the investigator cite front-end code the way it already cites C#. |
| 6 | **CI integration** | Poll the PR's build validation status; on failure, pull the CI log, resume the fix session with the failure, and propose a follow-up commit. This is what makes static-mode repos genuinely usable — and it is the fallback verification story for as long as item 4 is unbuilt. |
| 7 | **GitHub support** | The provider layer already abstracts issues + git host; GitHub needs an `IssueProvider` and a `GitHostProvider` (PRs via REST, `gh` not required), plus token handling. ~2 days. |
| 8 | **Jira support** | Harder than GitHub: ADF description format, different attachment model, smart-commit syntax instead of `AB#`. |
| 9 | **Multi-user / team mode** | Requires everything v1 deliberately avoided: a server, real auth, RBAC, Postgres, shared run history. A separate product decision, not an increment. |
| 10 | **Fix-quality metrics** | Track PR acceptance rate, review-comment count, revert rate, and time-to-merge for LazyBoy PRs vs. hand-written ones. The honest scoreboard; needed before anyone claims a number. |
| 11 | **Local model routing for cheap stages** | Triage and build-output parsing on a local model; only worth it if batch triage (#1) makes volume matter. |
| 12 | **VS Code extension** | Open the diff review in the editor instead of the browser. Nice, not necessary — the browser diff viewer already exists and works. |
