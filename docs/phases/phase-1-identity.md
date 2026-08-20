# Phase 1 — Identity & Credential Vault

Part of [LazyBoy Master Design](../LazyBoy-Design.md).

---

## Goal

Make LazyBoy *be you* against Azure DevOps and Azure Monitor (P2), with no service principal, no shared secret, and nothing sensitive on disk outside the OS keychain. At the end of this phase the Settings screen shows six green cards — ADO identity, ADO project access, Code Search, App Insights, **package feed (ADO Artifacts)**, Claude CLI — and every later phase can call `await tokens.bearer(ADO_SCOPE)` or `vault.ado_pat()` without thinking about auth again.

The sixth probe is not decoration. Packages come from a **private ADO Artifacts feed** (master doc §8, answer 10), and verification in Phase 7 is **compile-only** — so if `dotnet restore` can't authenticate to that feed, the compiler never runs and the one objective signal in the whole product silently disappears (master doc §8.2). Phase 1 therefore owns feed credentials *and* owns the probe that proves they work.

This is the phase where enterprise reality bites: Conditional Access, blocked device-code flows, guest accounts, and `az` not being installed. The design budget goes into *diagnosable failure*, not into the happy path — a wrong-tenant error must surface as "you're signed in to tenant X, LazyBoy needs Y — run `az login --tenant Y`", never as a 401.

## Definition of Done

| # | Criterion | Verified by |
|---|---|---|
| D1 | `GET /api/auth/health` returns six probes with `status ∈ {ok, warn, fail, skipped}`, each with a machine `code` and a human `remedy` | `test_health.py` |
| D2 | With `az login` already done, connecting takes zero interaction and `ado_identity` goes green in < 2 s | manual |
| D3 | Without `az`, the browser flow completes and the token cache persists across restarts (second launch is silent) | manual, two OSes |
| D4 | With interactive+device-code both blocked, the UI shows the exact AADSTS code and offers the PAT path inline | `test_entra_blockers.py` |
| D5 | A PAT is validated before storage (`GET /_apis/connectionData`), rejected with a scope-specific message if insufficient, and its expiry is stored and warned on ≤ 7 days | `test_pat.py` |
| D6 | Nothing sensitive is in `lazyboy.db`, `config.yaml`, logs, or any HTTP response — a grep of all four for the fixture secret finds nothing | `test_no_secret_leak.py` |
| D7 | `git clone` of a private ADO repo succeeds with no credential written to `~/.git-credentials`, `git config`, or the remote URL, and the token is not visible in `ps` | `test_git_creds.py` |
| D8 | A 401 from ADO triggers exactly one silent token refresh + retry; a second 401 raises `ReauthRequired` and emits a UI-visible auth event | `test_token_provider.py` |
| D9 | Disconnect removes every keychain entry under the `lazyboy` service and clears the MSAL cache; a subsequent health check is `fail`/`not_connected`, not a crash | `test_vault.py` |
| D10 | Code Search absent (404) degrades to `warn`, never blocks | `test_health.py` |
| D11 | Feed credentials are generated **per run into the run directory**, never into `~/.nuget/NuGet.config`, `~/.npmrc`, or any repo file; the files are `0600`, deleted on run teardown, and a crash-recovery sweep removes orphans on startup | `test_feed_auth.py` |
| D12 | The **restore probe** runs a real `dotnet restore` (and `npm ci --dry-run` where a JS repo uses the feed) against one catalog repo and reports pass/fail with the raw NuGet/npm error surfaced verbatim — no interpretation, no "probably fine" | `test_feed_probe.py`, manual |

---

## Design

### 1. What is stored, what is re-acquired

The rule: **persist the minimum that cannot be re-derived, and prefer platform-managed caches over our own.**

| Item | Where | Why |
|---|---|---|
| Entra access token | **nowhere** — held in process memory only, ≤ 1 h | re-acquirable from the credential; persisting it adds risk for no benefit |
| Entra refresh token | MSAL persistent cache (`TokenCachePersistenceOptions(name="lazyboy")`) — DPAPI on Windows, Keychain on macOS, libsecret on Linux | this is the thing that makes launch #2 silent; MSAL encrypts it with OS primitives, better than anything we'd write |
| ADO PAT | `keyring` service `lazyboy`, key `ado_pat` | must survive restarts; user cannot re-derive it |
| PAT metadata (display name, expiry, scopes, `sha256[:8]` fingerprint) | SQLite `credential_meta` | non-sensitive; drives the expiry warning without touching the keychain |
| Auth mode, org, tenant, project | `config.yaml` | user-editable, non-secret |
| Anthropic API key | **not managed by us** — the `claude` CLI owns it (`claude login` or `ANTHROPIC_API_KEY`) | avoids a second copy of a credential; we only *detect* which mode is active |

Keychain namespacing: service is always the literal `lazyboy`; keys are `ado_pat`, `ado_pat_meta` is *not* used (metadata is in SQLite). A future multi-org world uses `ado_pat@<org>`; `CredentialVault` already takes an optional account suffix so that's a config change, not a rewrite.

### 2. CredentialVault

```python
# src/lazyboy/connectors/credentials.py
from __future__ import annotations
import base64, hashlib, json, logging
from dataclasses import dataclass
from datetime import UTC, date, datetime
from enum import StrEnum

import keyring
from keyring.errors import KeyringError, NoKeyringError

log = logging.getLogger(__name__)
SERVICE = "lazyboy"

class AuthMode(StrEnum):
    ENTRA = "entra"
    PAT = "pat"
    NONE = "none"

class VaultUnavailable(LazyBoyError):
    """No usable keyring backend on this machine."""
    problem_type = "lazyboy/vault-unavailable"

@dataclass(frozen=True, slots=True)
class PatMeta:
    fingerprint: str            # sha256(pat)[:8] — safe to log and show
    display_name: str | None
    expires_on: date | None
    scopes: tuple[str, ...]
    stored_at: datetime

    @property
    def days_left(self) -> int | None:
        return (self.expires_on - date.today()).days if self.expires_on else None

class CredentialVault:
    """The only code in LazyBoy allowed to touch the OS keychain."""

    def __init__(self, engine, account_suffix: str = "") -> None:
        self._engine = engine
        self._suffix = f"@{account_suffix}" if account_suffix else ""
        self._backend_checked = False

    # ---- backend --------------------------------------------------------
    def backend_name(self) -> str:
        return type(keyring.get_keyring()).__name__

    def ensure_backend(self) -> None:
        """Fail loudly and early rather than silently writing to a plaintext file."""
        if self._backend_checked:
            return
        kr = keyring.get_keyring()
        name = type(kr).__name__
        if name in {"fail.Keyring", "Keyring"} or "fail" in kr.__module__:
            raise VaultUnavailable(
                "No OS keychain backend is available. On headless Linux install "
                "`gnome-keyring` + `libsecret` (and unlock it), or set "
                "PYTHON_KEYRING_BACKEND=keyrings.cryptfile.cryptfile.CryptFileKeyring.")
        if "chainer" in name.lower() and not getattr(kr, "backends", None):
            raise VaultUnavailable("Keyring chainer resolved to no usable backend.")
        self._backend_checked = True
        log.info("keyring backend: %s", name)

    # ---- ADO PAT --------------------------------------------------------
    def _key(self, base: str) -> str:
        return f"{base}{self._suffix}"

    def set_ado_pat(self, pat: str, meta: PatMeta) -> None:
        self.ensure_backend()
        keyring.set_password(SERVICE, self._key("ado_pat"), pat)
        self._save_meta("ado_pat", meta)
        log.info("stored ADO PAT fp=%s expires=%s", meta.fingerprint, meta.expires_on)

    def ado_pat(self) -> str | None:
        try:
            self.ensure_backend()
            return keyring.get_password(SERVICE, self._key("ado_pat"))
        except (KeyringError, NoKeyringError, VaultUnavailable) as exc:
            log.warning("keyring read failed: %s", type(exc).__name__)
            return None

    def ado_pat_meta(self) -> PatMeta | None:
        with self._engine.begin() as cx:
            row = cx.exec_driver_sql(
                "SELECT payload FROM credential_meta WHERE key = ?",
                (self._key("ado_pat"),)).fetchone()
        if not row:
            return None
        d = json.loads(row[0])
        return PatMeta(fingerprint=d["fingerprint"], display_name=d.get("display_name"),
                       expires_on=date.fromisoformat(d["expires_on"]) if d.get("expires_on") else None,
                       scopes=tuple(d.get("scopes", ())),
                       stored_at=datetime.fromisoformat(d["stored_at"]))

    def basic_header(self) -> dict[str, str] | None:
        pat = self.ado_pat()
        if not pat:
            return None
        b = base64.b64encode(f":{pat}".encode()).decode()
        return {"Authorization": f"Basic {b}"}

    # ---- lifecycle ------------------------------------------------------
    def delete_all(self) -> list[str]:
        """Full disconnect: keychain entries + metadata + MSAL cache."""
        removed = []
        for base in ("ado_pat",):
            try:
                keyring.delete_password(SERVICE, self._key(base))
                removed.append(base)
            except keyring.errors.PasswordDeleteError:
                pass
        with self._engine.begin() as cx:
            cx.exec_driver_sql("DELETE FROM credential_meta WHERE key LIKE ?", (f"%{self._suffix}",))
        if purge_msal_cache("lazyboy"):
            removed.append("msal_cache")
        return removed

    @staticmethod
    def fingerprint(secret: str) -> str:
        return hashlib.sha256(secret.encode()).hexdigest()[:8]
```

Backend behaviour per OS:

| OS | Backend | Prompt behaviour | Gotcha |
|---|---|---|---|
| macOS | `macOS.Keyring` (Keychain Services) | first read after a rebuild prompts "lazyboy wants to use your keychain"; "Always Allow" persists per-binary | a `uvx` upgrade changes the binary path → re-prompt. Expected; documented in the UI. |
| Windows | `Windows.WinVaultKeyring` (Credential Manager) | silent, per-user, DPAPI | 2 560-byte value cap — PATs and MSAL entries are far below it |
| Linux | `SecretService.Keyring` (libsecret/gnome-keyring) | needs an unlocked session keyring; headless/SSH fails | `ensure_backend()` raises `VaultUnavailable` with the `keyrings.cryptfile` remedy rather than silently using `PlaintextKeyring` |

We never install `keyrings.alt` — a plaintext fallback masquerading as a vault is worse than a hard failure.

### 3. Entra SSO flow

Chain order and rationale, straight out of [`reference/external-apis.md §1.1`](../reference/external-apis.md):

1. **`AzureCliCredential`** — shells out to `az account get-access-token`. Zero interaction, inherits the Conditional Access session already satisfied by `az login`, works behind corporate proxies. Try first, always.
2. **`InteractiveBrowserCredential`** — opens the system browser to a loopback redirect. Handles MFA and CA policies natively because it's a real auth code + PKCE flow. Requires a usable browser and a free loopback port.
3. **`DeviceCodeCredential`** — the "type this code on another device" flow. The only path that works over SSH/headless, and the one most likely to be blocked by tenant policy.

```python
# src/lazyboy/connectors/entra.py
from azure.identity import TokenCachePersistenceOptions
from azure.identity.aio import (
    AzureCliCredential, ChainedTokenCredential,
    DeviceCodeCredential, InteractiveBrowserCredential,
)

ADO_SCOPE   = "499b84ac-1321-427f-aa17-267ca6975798/.default"
LAW_SCOPE   = "https://api.loganalytics.io/.default"
ARM_SCOPE   = "https://management.azure.com/.default"

# Microsoft's well-known public client id for `az` — usable for delegated flows
# without registering an app in the tenant. If the tenant restricts it, config
# can point `entra.client_id` at a first-party app registration instead.
DEFAULT_CLIENT_ID = "04b07795-8ddb-461a-bbee-02f9e1bf7b46"

def build_credential(cfg: AdoSettings, *, device_code_callback=None) -> ChainedTokenCredential:
    cache = TokenCachePersistenceOptions(name="lazyboy", allow_unencrypted_storage=False)
    common = {"tenant_id": cfg.tenant_id or None,
              "client_id": cfg.entra_client_id or DEFAULT_CLIENT_ID,
              "cache_persistence_options": cache}
    return ChainedTokenCredential(
        AzureCliCredential(tenant_id=cfg.tenant_id or None, process_timeout=15),
        InteractiveBrowserCredential(**common, timeout=180),
        DeviceCodeCredential(**common, timeout=300,
                             prompt_callback=device_code_callback),
    )
```

`prompt_callback(verification_uri, user_code, expires_on)` does **not** print to stdout (there is no terminal in front of the user). It emits an `auth.device_code` event on the EventBus and stores it on `app.state.auth.pending_device_code` so the Settings screen can render the code, a copy button, and a "Open microsoft.com/devicelogin" link with a live countdown.

`allow_unencrypted_storage=False` is deliberate: if the platform can't encrypt the cache, we'd rather re-authenticate every launch than write refresh tokens in the clear.

**Token cache location** — MSAL extensions put it at `%LOCALAPPDATA%\.IdentityService\lazyboy` (Windows, DPAPI), the login Keychain (macOS), and `~/.IdentityService/lazyboy` sealed with libsecret (Linux). `purge_msal_cache("lazyboy")` deletes the file/entry on disconnect.

### 4. Enterprise blockers → exact UX fallbacks

The `ClientAuthenticationError` message from `azure-identity` contains the AADSTS code. `classify_auth_error()` maps it to a `code`, a plain-English `detail`, and a `remedy` with a copyable command. The UI never shows a raw stack trace.

| Signal | Meaning | LazyBoy code | UX / remedy |
|---|---|---|---|
| `AADSTS7000218` | client assertion required — tenant blocks the public client for this flow | `entra_public_client_blocked` | "Your tenant blocks the generic client. Ask IT for an app registration id and set `ado.entra_client_id`, or use a PAT." + PAT button |
| `AADSTS50076` / `AADSTS50079` | MFA required for this resource | `entra_mfa_required` | Skip CLI, force `InteractiveBrowserCredential` (it can satisfy MFA). Button: "Sign in again in the browser". |
| `AADSTS53003` | blocked by Conditional Access (device compliance / location) | `entra_conditional_access` | "CA policy requires a compliant/joined device. Sign in with Edge on your corporate device, or run `az login` there first." Offer PAT. |
| `AADSTS50020` | user from another tenant / guest not authorized | `entra_wrong_tenant` | Show the tenant we resolved vs `ado.tenant_id`. Remedy: `az login --tenant <configured>` or fix `tenant_id` in config. |
| `AADSTS700016` / `AADSTS500011` | app or resource not found in tenant | `entra_app_not_in_tenant` | Usually a wrong `tenant_id`. Same remedy panel as above. |
| `AADSTS65001` | user/admin has not consented | `entra_consent_required` | "Admin consent needed for `user_impersonation` on Azure DevOps." Offer the consent URL; offer PAT. |
| `AADSTS900023` | invalid tenant name | `entra_bad_tenant_value` | Config validation error, inline on the field |
| `AADSTS50158` | external security challenge not satisfied | `entra_conditional_access` | as above |
| `az` not on PATH / `CredentialUnavailableError` from CLI | | `az_cli_missing` | Non-fatal, `skipped` in the chain. Card shows "Azure CLI not found — install it for silent sign-in: `winget install Microsoft.AzureCLI` / `brew install azure-cli`" |
| `az` present but not logged in (`AADSTS`-free "Please run 'az login'") | | `az_cli_not_logged_in` | Copy button for `az login --tenant <tenant>` |
| Device code flow disabled by policy | | `entra_device_code_blocked` | Detected as `AADSTS7000218`/`AADSTS50199` during device flow → collapse the whole Entra path, promote the PAT card |
| Browser can't open (headless/WSL without wslu) | `InteractiveBrowserCredential` timeout | `entra_browser_unavailable` | Automatically fall through to device code and render the code in the UI |
| Guest (B2B) account with no ADO identity | ADO returns 401/203 even with a valid token | `ado_identity_not_provisioned` | "Your Entra account authenticates but has no Azure DevOps identity in org `<org>`. Ask an org admin to add you, or use a PAT issued from an account that has access." |

Non-obvious detail: ADO returns **HTTP 203 with an HTML sign-in page** rather than 401 when a token is valid for Entra but not accepted by the org. Every ADO call therefore checks `resp.status_code == 203 or "text/html" in content-type` and raises `AdoNotAuthorized` — this single check prevents a whole class of "why is my JSON a login page" confusion.

Circuit breaker: if all three Entra legs fail, the chain is not retried for 60 s (an in-memory cooldown), and the UI switches the primary CTA to "Use a Personal Access Token".

### 5. PAT path

Required scopes (mirrors [`external-apis.md §1.2`](../reference/external-apis.md), with the ADO scope strings):

| UI checkbox | Scope token | Needed for | Phase |
|---|---|---|---|
| Work Items — Read, write, & manage | `vso.work_write` | WIQL, batch read, comments, tags, JSON-Patch | 2, 3, 6 |
| Code — Read & write | `vso.code_write` | clone, push `bug/<id>-<slug>` | 4, 7, 8 |
| Code — Pull Request contribute *(included in `vso.code_write`)* | `vso.code_write` | create PR | 8 |
| **Packaging — Read** | `vso.packaging` | `dotnet restore` / `npm ci` against the private ADO Artifacts feed — **without this, nothing compiles and Phase 7 verifies nothing** (§9) | 1, 4, 7 |
| Build — Read *(optional)* | `vso.build` | build definition → repo in the catalog | 4 |
| Identity — Read *(optional)* | `vso.identity` | resolve reviewer GUIDs for PRs | 8 |
| Token administration *(optional)* | `vso.tokenadministration` | read the PAT's own expiry via the PATs API | 1 |

Validation, performed **before** anything is written to the keychain:

```python
async def validate_pat(http: httpx.AsyncClient, org: str, pat: str) -> PatMeta:
    headers = {"Authorization": "Basic " + b64(f":{pat}")}
    r = await http.get(f"https://dev.azure.com/{org}/_apis/connectionData",
                       params={"api-version": "7.1-preview"}, headers=headers)
    if r.status_code in (401, 203) or "text/html" in r.headers.get("content-type", ""):
        raise AuthError("pat_invalid", "The PAT was rejected by "
                        f"https://dev.azure.com/{org}. Check the token and the organization name.")
    r.raise_for_status()
    ident = r.json()["authenticatedUser"]
    if ident.get("providerDisplayName") in (None, "Anonymous"):
        raise AuthError("pat_no_identity", "The PAT authenticated but resolved to an anonymous "
                                           "identity — it was probably issued for a different org.")

    # Probe the scopes we actually need, cheaply, and report which one is missing.
    probes = {
        "vso.work_write": ("POST", f"https://dev.azure.com/{org}/_apis/wit/wiql",
                           {"query": "SELECT [System.Id] FROM WorkItems WHERE [System.Id] = 0"}),
        "vso.code_write": ("GET", f"https://dev.azure.com/{org}/_apis/git/repositories", None),
    }
    missing = [s for s, (m, u, body) in probes.items()
               if (await http.request(m, u, params={"api-version": "7.1"},
                                      json=body, headers=headers)).status_code == 403]
    if missing:
        raise AuthError("pat_insufficient_scope",
                        "The PAT is missing: " + ", ".join(SCOPE_LABELS[s] for s in missing))

    expiry = await _pat_expiry(http, org, headers, pat)     # PATs API; None if not permitted
    return PatMeta(fingerprint=CredentialVault.fingerprint(pat),
                   display_name=ident.get("providerDisplayName"),
                   expires_on=expiry, scopes=tuple(probes), stored_at=datetime.now(UTC))
```

Expiry tracking uses `GET https://vssps.dev.azure.com/{org}/_apis/tokens/pats?api-version=7.1-preview.1`, matching the entry whose `authorizationId`/`targetAccounts` fits — but that endpoint itself requires `vso.tokenadministration`, which most users won't grant. When it 401s we fall back to asking the user for the expiry date in the PAT form (a date picker, optional). A startup check emits `credential.expiring` when `days_left ≤ 7` and the UI shows an amber banner with a link to the PAT creation page pre-scoped.

**Never** does a PAT leave the vault→header path: it is not logged, not returned by `/api/config`, not echoed by the form (the input is write-only; the UI shows `fp:1a2b3c4d` afterwards).

### 6. Git credential injection

Requirements: no token in `~/.git-credentials`, no token in the repo's `git config`, no token in the remote URL (it would end up in `origin` forever), no token in `ps` output.

```python
# src/lazyboy/connectors/git.py  (Phase 1 slice)
async def git_env(tokens: TokenProvider, vault: CredentialVault) -> dict[str, str]:
    """Env that makes any `git` subprocess authenticate as the user, once."""
    if (pat := vault.ado_pat()):
        secret = "Authorization: Basic " + b64(f":{pat}")
    else:
        secret = "Authorization: Bearer " + await tokens.raw(ADO_SCOPE)
    return {
        # config-via-env: never touches any config file, never appears in argv/ps
        "GIT_CONFIG_COUNT": "2",
        "GIT_CONFIG_KEY_0": "http.https://dev.azure.com/.extraheader",
        "GIT_CONFIG_VALUE_0": secret,
        "GIT_CONFIG_KEY_1": "credential.helper",
        "GIT_CONFIG_VALUE_1": "",            # disable any inherited helper (osxkeychain/manager)
        "GIT_TERMINAL_PROMPT": "0",          # fail fast instead of hanging on a prompt
        "GIT_ASKPASS": "echo",               # belt and braces
        "GCM_INTERACTIVE": "never",
    }
```

Why `GIT_CONFIG_*` env rather than `git -c http.extraheader=...`: the `-c` form puts the token in `argv`, visible to `ps` for any user on the box. The env form is only visible via `/proc/<pid>/environ`, which is owner-restricted. The URL-scoped key (`http.https://dev.azure.com/.extraheader`) also means the header is never sent to a different host if a repo has an unexpected remote.

Token lifetime vs. clone duration: a 4 GB monorepo clone can outlive a 60-minute token. Mitigations: (a) `tokens.raw()` refreshes if `< 15 min` remain before starting a long git op; (b) on `fatal: Authentication failed` we re-acquire and retry once; (c) `--filter=blob:none` partial clone (Phase 4) keeps the initial fetch short.

### 7. TokenProvider

```python
# src/lazyboy/connectors/tokens.py
from __future__ import annotations
import asyncio, logging, time
from dataclasses import dataclass

log = logging.getLogger(__name__)
REFRESH_MARGIN_S = 300          # refresh when < 5 min remain (per external-apis §1.1)
LONG_OP_MARGIN_S = 900          # stricter margin before a long git/clone operation

class ReauthRequired(LazyBoyError):
    problem_type = "lazyboy/reauth-required"

@dataclass(slots=True)
class _Cached:
    token: str
    expires_on: int             # epoch seconds

class TokenProvider:
    """Per-scope token cache with proactive refresh and single-flight acquisition."""

    def __init__(self, credential, bus: EventBus | None = None) -> None:
        self._cred = credential
        self._bus = bus
        self._cache: dict[str, _Cached] = {}
        self._locks: dict[str, asyncio.Lock] = {}

    async def raw(self, scope: str, *, margin: int = REFRESH_MARGIN_S) -> str:
        now = int(time.time())
        hit = self._cache.get(scope)
        if hit and hit.expires_on - now > margin:
            return hit.token
        lock = self._locks.setdefault(scope, asyncio.Lock())
        async with lock:                                  # single-flight
            hit = self._cache.get(scope)
            if hit and hit.expires_on - int(time.time()) > margin:
                return hit.token
            try:
                tok = await self._cred.get_token(scope)
            except ClientAuthenticationError as exc:
                info = classify_auth_error(exc)
                if self._bus:
                    await self._bus.emit_global("auth.failed", info.as_dict())
                raise ReauthRequired(info.detail, code=info.code) from exc
            self._cache[scope] = _Cached(tok.token, tok.expires_on)
            log.info("acquired token scope=%s ttl=%ds", scope, tok.expires_on - int(time.time()))
            return tok.token

    async def bearer(self, scope: str) -> dict[str, str]:
        return {"Authorization": f"Bearer {await self.raw(scope)}"}

    def invalidate(self, scope: str | None = None) -> None:
        self._cache.pop(scope, None) if scope else self._cache.clear()

    async def aclose(self) -> None:
        self._cache.clear()
        await self._cred.close()
```

401 interception is an `httpx` auth-aware transport hook shared by every connector, so no call site handles it:

```python
class AdoAuth(httpx.Auth):
    requires_response_body = False

    def __init__(self, tokens: TokenProvider, vault: CredentialVault, scope: str) -> None:
        self._t, self._v, self._scope = tokens, vault, scope

    async def async_auth_flow(self, request):
        request.headers.update(await self._headers())
        response = yield request
        if response.status_code in (401, 203):
            if self._v.ado_pat():                         # PAT mode: no refresh possible
                raise ReauthRequired("The stored PAT was rejected — it may have expired or been revoked.",
                                     code="pat_rejected")
            self._t.invalidate(self._scope)               # Entra mode: one silent retry
            request.headers.update(await self._headers())
            response = yield request
            if response.status_code in (401, 203):
                raise ReauthRequired("Re-authentication did not restore access to Azure DevOps.",
                                     code="entra_reauth_failed")

    async def _headers(self) -> dict[str, str]:
        return self._v.basic_header() or await self._t.bearer(self._scope)
```

Exactly one retry — a loop here would burn through a Conditional Access lockout. `ReauthRequired` surfaces as an RFC 7807 problem with `status: 401` and `code`, which the frontend's fetch wrapper turns into a "Reconnect" toast that deep-links to Settings.

### 8. Connection health checks

Six independent probes, run concurrently, each with its own timeout, each degradable. Results are cached for 60 s (`?force=true` bypasses) because the Settings screen polls. Five probes are cheap HTTP calls with a 10 s timeout; the sixth — the **restore probe** — spawns a real build tool, so it gets its own longer timeout and is **not** run on every poll (see §9.4).

```python
# src/lazyboy/api/auth.py  (health)
@dataclass(slots=True)
class Probe:
    id: str
    label: str
    status: Literal["ok", "warn", "fail", "skipped"]
    detail: str
    code: str | None = None
    remedy: str | None = None
    data: dict | None = None
    latency_ms: int | None = None

async def check_health(ctx: AuthContext) -> list[Probe]:
    async def timed(fn, pid, label):
        t0 = time.perf_counter()
        try:
            p = await asyncio.wait_for(fn(), timeout=10)
        except TimeoutError:
            p = Probe(pid, label, "fail", "Timed out after 10 s.", code="timeout",
                      remedy="Check VPN / proxy connectivity to the service.")
        except ReauthRequired as e:
            p = Probe(pid, label, "fail", str(e), code=e.code, remedy="Reconnect in Settings.")
        except Exception as e:                                    # noqa: BLE001
            info = classify_auth_error(e)
            p = Probe(pid, label, "fail", info.detail, code=info.code, remedy=info.remedy)
        p.latency_ms = int((time.perf_counter() - t0) * 1000)
        return p

    return list(await asyncio.gather(
        timed(lambda: _ado_identity(ctx),  "ado_identity",  "Azure DevOps identity"),
        timed(lambda: _ado_project(ctx),   "ado_project",   "Project access"),
        timed(lambda: _code_search(ctx),   "code_search",   "Code Search extension"),
        timed(lambda: _app_insights(ctx),  "app_insights",  "Application Insights"),
        timed(lambda: _package_feed(ctx),  "package_feed",  "Package feed (ADO Artifacts)"),
        timed(lambda: _claude_cli(ctx),    "claude_cli",    "Claude CLI"),
    ))

async def _ado_identity(ctx) -> Probe:
    r = await ctx.http.get(f"https://dev.azure.com/{ctx.org}/_apis/connectionData",
                           params={"api-version": "7.1-preview"}, auth=ctx.ado_auth)
    if r.status_code == 203 or "text/html" in r.headers.get("content-type", ""):
        return Probe("ado_identity", "Azure DevOps identity", "fail",
                     f"Authenticated with Entra but org '{ctx.org}' did not accept the identity.",
                     code="ado_identity_not_provisioned",
                     remedy="Ask an org admin to add you to the organization, or use a PAT.")
    r.raise_for_status()
    u = r.json()["authenticatedUser"]
    return Probe("ado_identity", "Azure DevOps identity", "ok",
                 f"Signed in as {u.get('providerDisplayName')} ({u.get('properties', {}).get('Account', {}).get('$value', '')})",
                 data={"id": u["id"], "mode": ctx.mode})

async def _ado_project(ctx) -> Probe:
    r = await ctx.http.get(f"https://dev.azure.com/{ctx.org}/_apis/projects",
                           params={"api-version": "7.1", "$top": 200}, auth=ctx.ado_auth)
    r.raise_for_status()
    names = [p["name"] for p in r.json().get("value", [])]
    if ctx.project and ctx.project not in names:
        return Probe("ado_project", "Project access", "fail",
                     f"Project '{ctx.project}' is not visible to you.", code="project_not_visible",
                     remedy=f"Pick one of: {', '.join(names[:10])}", data={"projects": names})
    return Probe("ado_project", "Project access", "ok",
                 f"{len(names)} project(s) visible.", data={"projects": names})

async def _code_search(ctx) -> Probe:
    r = await ctx.http.post(f"https://almsearch.dev.azure.com/{ctx.org}/_apis/search/codesearchresults",
                            params={"api-version": "7.1"},
                            json={"searchText": "lazyboy-probe", "$top": 1, "includeFacets": False},
                            auth=ctx.ado_auth)
    if r.status_code == 404:
        return Probe("code_search", "Code Search extension", "warn",
                     "The Code Search extension is not installed in this organization.",
                     code="code_search_missing",
                     remedy="Repo resolution will use the catalog and heuristics only "
                            "(Phase 4). Install 'Code Search' from the Marketplace to enable "
                            "org-wide symbol search.")
    if r.status_code == 403:
        return Probe("code_search", "Code Search extension", "warn",
                     "Code Search is installed but this credential cannot query it.",
                     code="code_search_forbidden", remedy="PAT needs the 'Code (read)' scope.")
    r.raise_for_status()
    return Probe("code_search", "Code Search extension", "ok", "Available.")

async def _app_insights(ctx) -> Probe:
    if not ctx.ai_resource:
        return Probe("app_insights", "Application Insights", "skipped",
                     "No App Insights resource configured.",
                     remedy="Add one under app_insights.resources in config.yaml (Phase 3).")
    res = await ctx.logs.query_resource(ctx.ai_resource.resource_id,
                                        "union * | take 1 | project itemType",
                                        timespan=timedelta(hours=1))
    if res.status == LogsQueryStatus.FAILURE:
        return Probe("app_insights", "Application Insights", "fail", str(res.partial_error),
                     code="ai_query_failed",
                     remedy="You need at least 'Log Analytics Reader' (or Reader) on the resource.")
    return Probe("app_insights", "Application Insights", "ok",
                 f"Queried {ctx.ai_resource.name} successfully.",
                 data={"resource_id": ctx.ai_resource.resource_id})

async def _package_feed(ctx) -> Probe:
    """Two-stage: a cheap feed-index reachability check, then (on demand) a real restore.

    The cheap stage runs on every health poll. The expensive stage — an actual
    `dotnet restore` against a catalog repo — runs only when explicitly requested
    (`POST /api/auth/feed/restore-probe`) or once per app launch, because it can take
    a minute and it writes to the package cache. See §9.4.
    """
    if not ctx.feeds:
        return Probe("package_feed", "Package feed (ADO Artifacts)", "skipped",
                     "No Artifacts feed configured.",
                     remedy="Add feeds.nuget / feeds.npm in config.yaml. Without a feed, "
                            "restore will fail for any repo that uses internal packages, "
                            "and compile-only verification (Phase 7) will be worthless.")
    # Stage 1 — can this credential see the feed at all? (service index is a plain GET)
    r = await ctx.http.get(ctx.feeds.nuget.index_url, auth=ctx.ado_auth)
    if r.status_code in (401, 403):
        return Probe("package_feed", "Package feed (ADO Artifacts)", "fail",
                     f"{r.status_code} from {ctx.feeds.nuget.index_url}",
                     code="feed_unauthorized",
                     remedy="The credential cannot read the feed. In PAT mode the token needs "
                            "'Packaging (read)'. In SSO mode, ask for Feed Reader on the feed. "
                            "Until this is green, LazyBoy cannot compile anything and every "
                            "Change Report will be stamped 'restore failed'.")
    r.raise_for_status()

    last = ctx.feed_probe_cache.get("restore")       # result of the last real restore
    if last is None:
        return Probe("package_feed", "Package feed (ADO Artifacts)", "warn",
                     "Feed is reachable, but no restore has been proven yet.",
                     code="feed_restore_unproven",
                     remedy="Run the restore probe — reachability is not the same as restore.")
    if not last.ok:
        return Probe("package_feed", "Package feed (ADO Artifacts)", "fail",
                     last.stderr_tail,                       # verbatim NuGet/npm error
                     code="feed_restore_failed",
                     remedy="Compile-only verification is unusable until this passes. "
                            "Re-authenticate, then re-run the restore probe.",
                     data={"repo": last.repo, "command": last.command,
                           "duration_ms": last.duration_ms})
    return Probe("package_feed", "Package feed (ADO Artifacts)", "ok",
                 f"`{last.command}` succeeded against {last.repo} "
                 f"({last.duration_ms} ms, {last.package_count} packages).",
                 data={"repo": last.repo, "checked_at": last.checked_at})


async def _claude_cli(ctx) -> Probe:
    exe = shutil.which("claude")
    if not exe:
        return Probe("claude_cli", "Claude CLI", "fail", "`claude` was not found on PATH.",
                     code="claude_cli_missing",
                     remedy="Install it: `npm i -g @anthropic-ai/claude-code`, then `claude login`.")
    proc = await asyncio.create_subprocess_exec(exe, "--version",
                                                stdout=asyncio.subprocess.PIPE,
                                                stderr=asyncio.subprocess.STDOUT)
    out, _ = await proc.communicate()
    version = out.decode().strip()
    if os.environ.get("ANTHROPIC_API_KEY"):
        mode, note = "api_key", "using ANTHROPIC_API_KEY from the environment"
    elif (Path.home() / ".claude" / ".credentials.json").exists() or \
         (Path.home() / ".claude.json").exists():
        mode, note = "subscription", "signed in via `claude login`"
    else:
        return Probe("claude_cli", "Claude CLI", "warn", f"{version} found but not authenticated.",
                     code="claude_not_authenticated",
                     remedy="Run `claude login`, or set ANTHROPIC_API_KEY.", data={"version": version})
    return Probe("claude_cli", "Claude CLI", "ok", f"{version} — {note}.",
                 data={"version": version, "auth_mode": mode})
```

Overall status = worst probe, with `warn` never blocking: `fail` on `ado_identity` or `claude_cli` gates the rest of the app (the Inbox shows "Connect first"); everything else is advisory. `package_feed` is deliberately **advisory rather than gating** — you can still investigate a bug and read code with a broken feed; what you cannot do is *verify*, and Phase 7 says so on every artefact it produces rather than Phase 1 refusing to start.

### 9. Package feed authentication (ADO Artifacts)

Restore is not a build detail here — it is the load-bearing beam under compile-only verification (master doc §8.2). This section is how LazyBoy makes `dotnet restore` and `npm ci` authenticate to a **private ADO Artifacts feed** using the ADO credential it already holds, without leaving a token lying around on disk.

The credential situation is the good news: an Artifacts feed in the same org accepts exactly the same PAT or Entra access token used for work items and git. No second login, no second secret to manage. Everything below is about *delivery* — getting that token to two package managers that were designed around config files.

```yaml
# config.yaml
feeds:
  nuget:
    name: internal
    index_url: https://pkgs.dev.azure.com/{org}/{project}/_packaging/internal/nuget/v3/index.json
  npm:                                  # omit entirely if the front end uses public npm only
    name: internal
    registry: https://pkgs.dev.azure.com/{org}/{project}/_packaging/internal/npm/registry/
    scope: "@zeal"                      # scope-limited: only @zeal/* goes to the private feed
  probe_repo: portal-api                # catalog repo used by the restore probe
```

Note both URLs are **project-scoped**, matching the one-org/one-project topology (master doc §8, answer 1). There is no org-level feed picker.

#### 9.1 .NET — credential provider first, generated `nuget.config` second

Two mechanisms exist and LazyBoy prefers them in this order:

**(a) The Azure Artifacts Credential Provider (preferred).** If `~/.nuget/plugins/netcore/CredentialProvider.Microsoft/` is present — or `dotnet tool` can find it — NuGet will call it on a 401 and it reads the environment variable `VSS_NUGET_EXTERNAL_FEED_ENDPOINTS`:

```json
{"endpointCredentials":[{"endpoint":"https://pkgs.dev.azure.com/{org}/{project}/_packaging/internal/nuget/v3/index.json",
                         "username":"lazyboy","password":"<ADO PAT or Entra access token>"}]}
```

This is the mechanism to want, because **the secret lives in an environment variable of one child process and never touches the filesystem at all.** LazyBoy sets it in the `env` dict it passes to the restore subprocess (the same `env` mechanism Phase 7 already uses for the offline proxy vars), so it is scoped to that process tree, invisible in `ps` (argv is untouched), and gone when the process exits. The provider is detected once at connect time; a missing provider is a `warn` with a one-line install remedy, not a failure, because path (b) always works.

**(b) A generated per-run `nuget.config` with `packageSourceCredentials` (fallback).** When the credential provider is absent, NuGet's only other authenticated path is a config file:

```xml
<configuration>
  <packageSources>
    <clear />
    <add key="internal" value="https://pkgs.dev.azure.com/{org}/{project}/_packaging/internal/nuget/v3/index.json" />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <internal>
      <add key="Username" value="lazyboy" />
      <add key="ClearTextPassword" value="%LAZYBOY_FEED_TOKEN%" />
    </internal>
  </packageSourceCredentials>
</configuration>
```

> **Why writing the token to a file is undesirable, stated plainly.** A `nuget.config` with a `ClearTextPassword` is a plaintext bearer secret sitting on disk with no expiry enforcement of its own, and it fails in three specific ways that the environment-variable path does not. First, **persistence**: if the process crashes between writing and cleanup, the token outlives the run and nobody notices — it is not in the keychain, so no OS UI will ever show it to you for revocation. Second, **blast radius**: NuGet's config discovery walks *up* the directory tree, so a file written in the wrong place is silently picked up by unrelated builds, and a file written inside a repo worktree can be committed and pushed — the exact accident this whole product is trying not to have. Third, **backups and indexers** copy files; they do not copy environment variables. The keychain-only rule in §1 exists for these reasons, and a credentials file is a hole in it.

So when path (b) is used, it is scoped as hard as the format allows:

- **Per-run temp file in the run directory.** Written to `runs/<run-id>/nuget.config`, mode `0600`, never to `~/.nuget/NuGet.config`, never into a repo worktree. Passed explicitly with `dotnet restore --configfile <path>` so NuGet's upward config discovery is bypassed entirely and no other build can pick it up. (This is also why Phase 7's bash allowlist denies a `--configfile` override to the agent: the path is LazyBoy's to set.)
- **Env-var indirection, so the token is not the file's content.** NuGet expands `%VAR%` in config values, so the file stores the literal string `%LAZYBOY_FEED_TOKEN%` and the actual token is injected through the subprocess environment. The artefact on disk contains a variable name, not a secret. This is the same trick as (a), one level down, and it means the crash-leaves-a-file case leaks a *shape*, not a credential.
- **Deleted in a `finally`, and swept on startup.** Teardown unlinks the file; a startup sweep removes `runs/*/nuget.config` and `runs/*/.npmrc` older than the current session, because `finally` does not run when the process is killed.
- **`NUGET_PACKAGES` points at a shared cache** (`~/.lazyboy/cache/nuget`, or the user's existing `~/.nuget/packages` if configured), deliberately *outside* the run directory. The credential is per-run and ephemeral; the package cache is expensive and must be shared across runs and repos, or every run pays a full restore. Separating the two is what lets the credential be short-lived without making restore slow.

#### 9.2 JS/TS — `.npmrc` with a base64 token

npm's Artifacts flavour has no credential-provider equivalent, so the config-file path is the only one. The scoping rules are identical:

```ini
@zeal:registry=https://pkgs.dev.azure.com/{org}/{project}/_packaging/internal/npm/registry/
//pkgs.dev.azure.com/{org}/{project}/_packaging/internal/npm/registry/:username=lazyboy
//pkgs.dev.azure.com/{org}/{project}/_packaging/internal/npm/registry/:_password=${LAZYBOY_FEED_TOKEN_B64}
//pkgs.dev.azure.com/{org}/{project}/_packaging/internal/npm/registry/:email=lazyboy@localhost
always-auth=true
```

- The password field is the **base64 of the raw ADO token** (`base64(pat)` — not `base64(":"+pat)`, which is the git/Basic form; getting this wrong produces a 401 that looks like a permissions problem and wastes an afternoon). LazyBoy computes it; the user never pastes anything.
- `${VAR}` interpolation is native to npm, so again **the file holds a variable name and the environment holds the secret**.
- Written to `runs/<run-id>/.npmrc`, mode `0600`, and passed via `npm_config_userconfig=<path>` on the subprocess environment — *not* copied into the repo worktree, where it would be one `git add -A` away from a push. (Phase 7 never does `git add -A`, but defence in depth is cheap here.)
- Scope-limited by `@zeal:registry`: only the private scope resolves to Artifacts, everything else goes to public npm. A feed misconfiguration then breaks internal packages loudly rather than silently proxying the entire public registry through an authenticated endpoint.
- Same `finally` deletion and same startup sweep as the NuGet file.

#### 9.3 Token choice and refresh

PAT mode uses the PAT directly (it needs the **Packaging (read)** scope — added to the scope checklist on the PAT card). SSO mode uses an Entra access token for the ADO scope, which expires in ≤ 1 h. Restore can outlive that on a cold cache, so the same rule as the long-clone case (§6) applies: the token is re-acquired if under 15 minutes remain before a restore starts, and a 401 from the feed triggers exactly one re-acquire-and-retry before it is reported as a failure. `LAZYBOY_FEED_TOKEN` is materialised per subprocess invocation, never cached in a long-lived object.

#### 9.4 The restore probe — the gate that gives compile-only verification meaning

Reachability is not restore. A feed index can return 200 to a token that cannot actually resolve a package, and a warm package cache can make a broken feed look fine for weeks. So the probe **runs the real command**:

```
1. pick `feeds.probe_repo` from the catalog (or the first repo with a `dotnet` stack)
2. ensure a shallow worktree exists for it (reuse Phase 4's clone if present)
3. write the per-run credential files / set VSS_NUGET_EXTERNAL_FEED_ENDPOINTS
4. dotnet restore <probe repo> --configfile <temp> --force        # --force: ignore the warm cache
5. if a JS repo uses the feed:  npm ci --dry-run                   # resolves + authenticates, installs nothing
6. record {ok, command, exit_code, duration_ms, package_count, stderr_tail, checked_at}
7. delete the credential files
```

`--force` on the NuGet side is the point of the whole exercise: without it the probe would pass on cached packages and prove nothing. `npm ci --dry-run` is chosen for the same reason in reverse — it performs full resolution and authentication against the registry but writes no `node_modules`, so the probe is honest without being slow.

**Reporting is plain, not interpreted.** On failure the probe surfaces the raw NuGet/npm error text (`error NU1301: Unable to load the service index`, `E401 Unable to authenticate`) verbatim, plus the command that produced it. LazyBoy adds one sentence of consequence — *"Phase 7 cannot compile anything until this passes; every Change Report will be stamped restore failed"* — and nothing else. No guessing at the cause, no "this usually means…".

Cost control: the probe takes 10–90 s, so it is **not** part of the 60 s health poll. It runs once per app launch (in the background, after `ado_identity` goes green), on demand from the Settings button, and automatically after any credential change. `_package_feed` reports the cached result and distinguishes *"reachable but restore unproven"* (`warn`) from *"restore proven"* (`ok`) — a distinction that matters, because unproven is exactly the state in which someone would otherwise assume everything is fine.

**Who consumes it.** Phase 7's `verification.restored` and its refusal to claim `compiled: true` without a successful restore in the same run (§8.2 of the master doc); Phase 4's catalog scan, which records per-repo restore health so you know which repos LazyBoy can actually verify before you start a run; and the Diff Review UI, which deep-links back to this probe when a run's restore fails, because the fix is a credential refresh and not a code change.

### 10. Threat model

**In scope.** An attacker with a browser tab on the user's machine (drive-by page, malicious extension's content script in a same-site context, another localhost web app).

- They cannot read the session token: `SameSite=Strict`, `HttpOnly`, no CORS headers, and the token appears in the URL only on first load (immediately traded for a cookie and stripped by a 302).
- They cannot forge state-changing calls: `OriginGuardMiddleware` rejects any `Origin` outside `http://127.0.0.1:<port>`.
- DNS rebinding is blocked by the `Host` header check.

**Out of scope, stated plainly.** A process running **as the same OS user** owns the machine as far as LazyBoy is concerned: it can read `~/.lazyboy/lazyboy.db`, read the keychain (macOS ACLs are per-binary but a debugger defeats them), read `/proc/<pid>/environ`, and drive the API by reading the token from the process. This is the same posture as `az`, `gh`, and `git` itself. LazyBoy's job is to not *widen* that blast radius.

**What a compromised localhost would get, concretely:** the ADO PAT (if in PAT mode) → full work-item and code read/write within the PAT's scopes until it expires or is revoked; the Entra refresh token (if in SSO mode) → tokens for ADO/Log Analytics/ARM as the user, constrained by CA policy and revocable centrally. This asymmetry is the argument for preferring SSO: **refresh tokens are revocable and policy-bound; PATs are bearer secrets valid until their expiry date.** The UI says so, once, on the PAT card.

**The one place a secret is allowed near the filesystem** is the package-feed path (§9), and it is bounded deliberately: the credential provider route keeps the token in a subprocess environment variable only; the config-file route writes `%LAZYBOY_FEED_TOKEN%` — a variable *name* — into a `0600` file inside the run directory, passed explicitly with `--configfile` / `npm_config_userconfig` so no other build discovers it, deleted in a `finally`, and swept on startup for the kill-9 case. A leaked file therefore leaks a shape, not a credential. Nothing is ever written to `~/.nuget/NuGet.config`, `~/.npmrc`, or any path inside a repo worktree.

**Reducing dwell time:** PAT default expiry guidance is 90 days with a 7-day warning; disconnect purges everything; every credential is namespaced under the `lazyboy` service so a user can audit and revoke with the OS keychain UI. Nothing sensitive is ever written to `runs/<id>/audit.jsonl` — the audit records a token *fingerprint*, never a token.

---

## API

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/api/auth/status` | — | `{mode, connected, identity:{display_name,email,id}\|null, pat:{fingerprint,expires_on,days_left}\|null, keyring_backend}` |
| GET | `/api/auth/health` | `?force=false` | `{overall:"ok"\|"warn"\|"fail", probes:[Probe], checked_at}` |
| POST | `/api/auth/entra/connect` | `{tenant_id?, prefer:"auto"\|"browser"\|"device"}` | `202 {flow_id, method}` — non-blocking; progress on the global event stream |
| GET | `/api/auth/entra/pending` | — | `{flow_id, method, user_code?, verification_uri?, expires_at?, state}` (poll target for device code) |
| POST | `/api/auth/entra/cancel` | `{flow_id}` | `204` |
| POST | `/api/auth/pat` | `{pat, expires_on?}` | `200 PatMeta` or `400 problem` with `code ∈ {pat_invalid, pat_insufficient_scope, pat_no_identity}` |
| DELETE | `/api/auth/pat` | — | `204` |
| POST | `/api/auth/disconnect` | — | `200 {removed:["ado_pat","msal_cache"]}` |
| POST | `/api/auth/feed/restore-probe` | `{repo?}` | `202 {probe_id}` — runs the real restore (§9.4) in the background; result lands on the event stream and in the `package_feed` probe. Long-running, so never inline in `/health`. |
| GET | `/api/auth/feed/restore-probe` | — | `{ok, repo, command, exit_code, duration_ms, package_count, stderr_tail, checked_at}` — the last result, verbatim |
| GET | `/api/auth/config` | — | `{organization, default_project, tenant_id, auth_mode, app_insights:{...}, feeds:{nuget:{name,index_url}, npm:{name,registry,scope}, probe_repo}}` (never secrets) |
| PUT | `/api/auth/config` | `{organization, default_project?, tenant_id?, auth_mode?}` | `200` — writes `config.yaml`, invalidates the health cache |
| GET | `/api/events` | `?after=` | global SSE stream (auth events live here; run streams stay per-run) |

Global auth events: `auth.started`, `auth.device_code` (`{user_code, verification_uri, expires_at}`), `auth.succeeded`, `auth.failed` (`{code, detail, remedy}`), `credential.expiring` (`{days_left}`), `feed.probe_started`, `feed.probe_finished` (`{ok, repo, duration_ms, stderr_tail}`).

---

## UI — the Settings screen

Route `/settings`, four stacked sections in a single column, max-width 880 px.

**1. Organization** — a small form: `Organization` (with the resolved URL rendered below as `https://dev.azure.com/<org>` in mono), `Default project` (a `<select>` populated from the `ado_project` probe's `data.projects` once connected, free text before), `Tenant ID` (optional, GUID-validated, with a "how do I find this?" popover). Saving `PUT`s and re-runs health.

**2. Connection** — two mutually-exclusive cards, side by side on wide screens.

- **Entra SSO (recommended)** — a badge listing which chain leg succeeded (`Azure CLI` / `Browser` / `Device code`) and the signed-in UPN. When disconnected: a primary "Sign in with Microsoft" button and a secondary "Use device code" link. During a device flow the card expands into a panel showing the 9-character code in 3xl mono with a copy button, an "Open microsoft.com/devicelogin" button, and a countdown ring from `expires_at`; it closes itself on `auth.succeeded`. On failure the card turns amber/red and renders `detail` plus `remedy`, with any command in the remedy shown as a copyable `<code>` block (e.g. `az login --tenant 72f988bf-…`). A "Show technical details" disclosure reveals the raw AADSTS code — never shown by default.
- **Personal Access Token** — collapsed by default with the line "Use this if SSO is blocked in your tenant." Expanding shows: a link to `https://dev.azure.com/<org>/_usersSettings/tokens` that opens the PAT creation page, a scope checklist rendered from the table above (informational — ADO can't be pre-filled via URL), a password input, an optional expiry date picker, and a "Validate & save" button. On submit the button shows a spinner while `POST /api/auth/pat` runs the identity + scope probes; failures render inline against the specific missing scope ("Missing: Code — Read & write"). On success the input is replaced permanently by a summary row: `fp:1a2b3c4d · Francesco Colombo · expires 2026-11-14 (87 days)` with a "Replace" and a "Remove" button. If `days_left ≤ 7` the row is amber; if expired, red with "Create a new token".

**3. Package feed (ADO Artifacts)** — a connection card sitting alongside the other two, because the feed is a connection even though it reuses the ADO credential. It shows: the configured NuGet feed name and index URL (mono, truncated middle), the npm feed and its `@scope` if configured (or "public npm only"), which delivery mechanism is in use — `credential provider` (green, "token stays in memory") or `generated nuget.config` (amber, "per-run temp file, deleted after each run", with a "why?" popover pointing at §9.1) — and the PAT scope reminder `Packaging (read)` when in PAT mode.

Its centrepiece is the **restore probe**: a `Run restore probe` button, the last result rendered as `✓ dotnet restore portal-api — 42 s, 318 packages · 12 minutes ago` or, on failure, a red block containing the **verbatim** NuGet/npm error in a mono `<pre>` with a copy button, the exact command that produced it, and one line of consequence: *"LazyBoy cannot compile anything until this passes. Every Change Report will be stamped `restore failed` and no fix will be verified."* While running, the button becomes a progress row driven by `feed.probe_started` / `feed.probe_finished`. If no probe has ever run, the card is amber with `Reachable, but restore unproven` — never green on reachability alone.

**4. Health** — a list of six rows, each: a status dot (`ok` green / `warn` amber / `fail` red / `skipped` grey), the label, the detail, latency in ms right-aligned in mono, and — when not `ok` — an expandable remedy block. A "Re-check" button calls `?force=true` (it re-runs the five cheap probes; the restore probe has its own button, since a 90-second poll would be hostile). Above the list, a one-line summary: "5 ok · 1 warning" plus the keyring backend name in `dim` text (`Keychain via macOS.Keyring`), which is exactly the detail that makes a Linux keyring problem self-diagnosing.

The left-rail connection dot in `AppShell` mirrors `overall` from a `useQuery(["auth","health"], {refetchInterval: 60_000})`; clicking it navigates to `/settings`. The Inbox (Phase 2) renders a full-page "Connect to Azure DevOps" empty state whenever `ado_identity` is `fail`, with a single button to Settings — no half-working screens.

---

## Tests

Testing credential code without real secrets is the whole trick: **the vault, the credential chain, and the HTTP layer are each injected, and no test ever needs a live tenant.**

| File | Approach |
|---|---|
| `test_vault.py` | `keyring.set_keyring(keyring.backends.fail.Keyring())` → asserts `VaultUnavailable` with the libsecret remedy. A `MemoryKeyring` fixture (dict-backed, implements the 4-method backend API) for the happy path: set→get→delete, suffix namespacing, `delete_all` clears SQLite metadata too. |
| `test_pat.py` | `respx` mocks `connectionData` (200 / 401 / **203+HTML** / anonymous identity), the WIQL and repositories probes (200 / 403), and the PATs API (200 / 401). Asserts the exact `code` for each and that **nothing is written to the vault on any failure path**. |
| `test_entra_blockers.py` | A `FakeCredential` whose `get_token` raises `ClientAuthenticationError` with each real AADSTS message string (a table of ~10 recorded messages, scrubbed). Asserts `classify_auth_error` returns the right `code`/`remedy` for every one — this is a pure table test and is the cheapest high-value test in the phase. |
| `test_token_provider.py` | Fake credential returning `AccessToken(tok, now+3600)`. Asserts: cache hit inside the margin; refresh inside the margin; **single-flight** (20 concurrent `raw()` → exactly 1 `get_token`); `invalidate` forces re-acquire; `ClientAuthenticationError` → `ReauthRequired` with a classified code. |
| `test_ado_auth_flow.py` | `respx` returns 401 then 200 → asserts exactly two requests and one `invalidate`; 401 then 401 → `ReauthRequired`; PAT mode 401 → immediate `ReauthRequired` (no refresh attempt); 203+HTML treated as 401. |
| `test_health.py` | All five probes mocked; asserts Code Search 404→`warn` (and `overall` still usable), App Insights unconfigured→`skipped`, `claude` missing→`fail`, timeout path, and 60 s cache with `?force=true` bypass. `shutil.which` and `create_subprocess_exec` are monkeypatched. |
| `test_feed_auth.py` | Pure file-generation tests, no network. Asserts: the generated `nuget.config` contains `%LAZYBOY_FEED_TOKEN%` and **never the token itself** (grep the bytes for the sentinel); the generated `.npmrc` contains `${LAZYBOY_FEED_TOKEN_B64}` and the base64 is `b64(pat)` not `b64(":"+pat)`; both files are `0600` and land under `runs/<id>/`, never in `~`, never in a worktree; `--configfile` / `npm_config_userconfig` are always passed so config discovery can't apply; teardown unlinks both; a simulated kill-9 leaves orphans that the startup sweep removes. Credential-provider path: `VSS_NUGET_EXTERNAL_FEED_ENDPOINTS` is in the subprocess `env` and in **no** argv. |
| `test_feed_probe.py` | The restore subprocess is faked (`create_subprocess_exec` monkeypatched) with recorded stdout/stderr for: success, `NU1301 Unable to load the service index`, `401 Unauthorized`, `E401 Unable to authenticate`, and a timeout. Asserts the probe reports the stderr **verbatim** (no rewriting), sets the right `code`, that `--force` is present on the restore argv (a probe that can pass on a warm cache is a bug), that `npm ci --dry-run` is used rather than `npm ci`, and that credential files are deleted on every path including the timeout. Also asserts `package_feed` is `warn` (`feed_restore_unproven`) — not `ok` — when only reachability has been checked. |
| `test_git_creds.py` | Runs a real `git` against a **local bare repo over `http.` config**: asserts `GIT_CONFIG_*` env is honoured, the secret never appears in `.git/config`, the process `cmdline` (read from `/proc/self/cmdline` in the child on Linux; skipped elsewhere), or `~/.git-credentials`. |
| `test_no_secret_leak.py` | End-to-end: store the fixture PAT `sentinel-pat-0123…`, drive `/api/auth/*` + a health check + a faked restore probe, then grep `lazyboy.db` bytes, `config.yaml`, every file in `logs/`, **every file under `runs/` (including the generated `nuget.config` and `.npmrc`, before and after teardown)**, and the JSON of every response for the sentinel. Must be zero hits. This is the test that keeps D6 honest as the codebase grows. |
| `test_auth_api.py` | Route-level: 401 without session token, `PUT /api/auth/config` persists and invalidates the health cache, `disconnect` is idempotent. |

CI never has a keychain (headless Linux), so the `MemoryKeyring` fixture is installed via an autouse conftest fixture and `PYTHON_KEYRING_BACKEND` is pinned; a single opt-in marker (`-m keyring_real`) runs the backend tests locally.

---

## Risks & mitigations

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | Tenant blocks the public `az` client id, so *every* Entra leg fails | high | `ado.entra_client_id` config escape hatch (paste an app registration id from IT); PAT card promoted automatically; the failure is classified (`entra_public_client_blocked`) rather than generic |
| R2 | Conditional Access requires a compliant device and the laptop isn't enrolled | high | `az login` on the corporate profile usually satisfies it; if not, PAT is the documented answer. The card names the policy class rather than showing AADSTS53003 raw. |
| R3 | Linux without an unlocked keyring → no credential storage at all | medium | Hard `VaultUnavailable` with the `keyrings.cryptfile` remedy, never a silent plaintext fallback (`keyrings.alt` is not a dependency) |
| R4 | macOS re-prompts for keychain access after every `uvx` upgrade (new binary path) | low | One-line note on the card the first time it happens; "Always Allow" per version is acceptable |
| R5 | PAT expiry is invisible because `vso.tokenadministration` isn't granted | medium | Optional user-entered expiry date; a `pat_rejected` 401 produces an actionable message instead of a mystery |
| R6 | Long clone outlives the access token | medium | 15-minute refresh margin before long ops; one automatic re-auth + retry on `Authentication failed`; partial clone shortens the window |
| R7 | Guest/B2B account authenticates to Entra but has no ADO identity (203 + HTML) | medium | Explicit 203/HTML detection in every ADO call, mapped to `ado_identity_not_provisioned` with an admin-ask remedy |
| R8 | `azure-identity` changes chain/cache semantics across minor versions | low | Pinned `>=1.19,<2`; a smoke test asserts `TokenCachePersistenceOptions` and `ChainedTokenCredential` still import and that a cache file/entry appears after a fake acquisition |
| R9 | Device-code UI is a poll loop that leaks a flow across restarts | low | Flow state is in-memory only with an `expires_at`; a restart cancels it and the card resets to disconnected |
| R10 | Users paste a PAT into the org field or vice versa | low | The PAT input is `type=password` and validated by shape (52-char base32) before any network call; the org field rejects anything containing `/` or `dev.azure.com` and offers to extract the org from a pasted URL |
| R11 | **Feed restore fails and nobody notices** — Phase 7 then "verifies" nothing while looking green | high | The probe runs the real command with `--force` so a warm cache can't mask it; `package_feed` is `warn` until a restore is *proven*, never `ok` on reachability; Phase 7 clamps `compiled` to `restored` and stamps the reason on every artefact; the catalog records per-repo restore health from the Phase 4 scan |
| R12 | A generated credential file is left behind, or picked up by an unrelated build | medium | Token never in the file (env-var indirection); `0600` inside the run dir only; explicit `--configfile` / `npm_config_userconfig` so NuGet/npm config discovery cannot find it; `finally` deletion plus a startup sweep for the kill-9 case; `test_no_secret_leak.py` greps `runs/` |
| R13 | PAT issued without **Packaging (read)** — everything else works, restore 401s | medium | The scope is in the PAT checklist and validated at save time with a feed-index call, so the failure is named at credential-entry time rather than discovered mid-run |

---

## Effort

**1.5 days** (the master estimate's upper bound; the SSO edge-case matrix is where most of the time goes, and the feed work is what pins it to 1.5 rather than 1).

| Chunk | Est. |
|---|---|
| `CredentialVault` + backend detection + metadata table (migration v2) | 2 h |
| Entra chain, device-code callback → EventBus, MSAL cache purge | 2 h |
| `classify_auth_error` table + tests | 1.5 h |
| PAT validate/store/expiry | 1.5 h |
| `TokenProvider` + `httpx.Auth` 401 interception | 1.5 h |
| Health probes (six) + caching | 2 h |
| Git credential env + test | 1 h |
| Feed credentials: provider detection, per-run `nuget.config` / `.npmrc` generation, env-var indirection, teardown + startup sweep | 1.5 h |
| Restore probe: runner, timeout, verbatim error capture, background scheduling, events | 1.5 h |
| Settings screen (four sections, device-code panel, feed card + probe result, health list) | 3.5 h |

**Previous:** [Phase 0 — Foundation & Skeleton](phase-0-foundation.md) · **Next:** [Phase 2 — Work Item Inbox](phase-2-workitem-inbox.md).
