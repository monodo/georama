# Implementation Plan: Georama BFF Authentication

## Overview

Georama becomes a Backend-For-Frontend and the single confidential OAuth client against the identity provider. The plan below builds the feature in the order design.md's components depend on each other: baseline safety-net tests first, then settings, then the one security-critical override (`GeoramaAccountAdapter.is_safe_url`), then the three viewer-facing endpoints, then the lower-risk and parallelizable pieces (Cognito middleware adaptation, `TokenStore`, CSRF re-enablement, provider-migration tooling), then end-to-end integration tests, then documentation.

Two sequencing decisions carried over from the brief, stated here so the reasoning travels with the plan:

- **Task 1 (backward-compatibility baseline tests) is deliberately first and is NOT marked optional**, even though it is pure test code. Its purpose is not coverage of new code but a safety net captured against *current* behaviour before CSRF re-enablement (task 15) or any other ungated change lands. Skipping it would defeat that purpose, so it is treated as required infrastructure rather than disposable test coverage.
- **CSRF re-enablement (task 15) is the highest-risk task.** It is not gated behind `BFF_ENABLED` (Requirement 10.10 fixes the closed list of ungated changes, and this plan does not add to it). Its exemptions (`require_viewer_origin`, the two `csrf_exempt` class-level decorators) land in the same task as the middleware switch, never after, and task 17 is a checkpoint dedicated to it.

`gg-viewer` requires zero code changes and is out of scope for every task below. No new runtime or test dependency is introduced anywhere in this plan (Requirement 14.1, 14.2); in particular no property-based testing library is added — the correctness properties in design.md are discharged by parametrised `pytest` cases.

## Tasks

- [ ] 1. Capture backward-compatibility baseline tests
  - These tests pin *existing* behaviour before any implementation change lands, so later tasks (especially CSRF re-enablement) can be checked against a known-good baseline rather than against assumptions.
  - [ ] 1.1 Write baseline test for the Bearer-token authentication path
    - Golden-test `GeoGirafeAuthenticationMiddleware`: a valid Bearer token with no session cookie resolves the expected `request.user` and the targeted endpoint returns its current status and parsed body.
    - _Requirements: 10.2, 10.8_
  - [ ] 1.2 Write baseline test for HTTP Basic authentication mode
    - Golden-test the `BASIC_HTTP` authentication method currently configured in `Dev`/`Prod`.
    - _Requirements: 10.3_
  - [ ] 1.3 Write baseline test for the Keycloak browser login flow and the admin login
    - Golden-test the current `openid_connect` (Keycloak) browser redirect flow and `/login` (`core/views/auth.py:Login`) under session and password authentication.
    - _Requirements: 10.4, 10.5_
  - [ ] 1.4 Write baseline test for `themes.json` auth-mode equivalence
    - Golden-test that the same user, same permission state, gets an identical `themes.json` body whether authenticated by session cookie or by Bearer token, as it does today.
    - _Requirements: 10.6, 10.9_

- [ ] 2. Checkpoint - Ensure all baseline tests pass
  - Ensure all tests pass, ask the user if questions arise. These baseline tests must stay green for the rest of this plan; a later task breaking one is a regression, not an accepted behaviour change.

- [ ] 3. Implement the BFF settings and configuration surface
  - [ ] 3.1 Declare `BFF_*` `values.*` descriptors on `Base`
    - Add `BFF_ENABLED` (`BooleanValue`, default `False`), `BFF_ALLOWED_VIEWER_HOSTS` (`ListValue`, single-space separator, default `[]`), `BFF_IDP_LOGOUT_URL` (`Value`, default `""`), `BFF_LOGINUSER_VERIFY_IDP` (`BooleanValue`, default `False`), each with `environ_prefix="GEORAMA"`.
    - _Requirements: 11.1, 11.9_
  - [ ] 3.2 Declare `SESSION_COOKIE_*` / `CSRF_COOKIE_*` settings and the `Dev`/`Prod` differences
    - Add `SESSION_COOKIE_HTTPONLY=True`, `SESSION_COOKIE_SECURE` (default `True`), `SESSION_COOKIE_SAMESITE` (default `"Lax"`), `SESSION_COOKIE_DOMAIN` (default unset), `SESSION_COOKIE_AGE` (default `43200`), `SESSION_SAVE_EVERY_REQUEST` (default `True`), `CSRF_COOKIE_SECURE`/`CSRF_COOKIE_SAMESITE` mirroring the session settings, `CSRF_COOKIE_HTTPONLY=False`, each with `environ_prefix="GEORAMA"`. Override `SESSION_COOKIE_SECURE=False` on `Dev`. Reject non-host entries in `BFF_ALLOWED_VIEWER_HOSTS` (containing `://`, `/`, `?`, or `#`) by raising at settings load time, naming the offending entry.
    - _Requirements: 6.1, 6.2, 6.3, 6.10, 6.11, 11.4, 11.6, 11.10_
  - [ ] 3.3 Set `ACCOUNT_ADAPTER` and `SOCIALACCOUNT_STORE_TOKENS` on `Base`
    - Point `ACCOUNT_ADAPTER` at the dotted path `georama.core.auth.bff.GeoramaAccountAdapter` (the module is created in task 6) and set `SOCIALACCOUNT_STORE_TOKENS = True`.
    - _Requirements: 11.5_
  - [ ]* 3.4 Write unit tests for settings defaults, environment overrides, and `Test` config inheritance
    - Cover: each `BFF_*` default and its `GEORAMA_`-prefixed override; `Dev` defaulting `SESSION_COOKIE_SECURE` to `False`; `Test` inheriting every `Base` declaration and resolving per-test overrides via Django's settings-override mechanism; the malformed-host-entry rejection from 3.2.
    - _Requirements: 11.1, 11.4, 11.6, 11.10, 11.11_

- [ ] 4. Implement startup configuration checks
  - [ ] 4.1 Implement `check_bff_configuration` in `georama/core/checks.py`
    - Register with `@register()`. Return `[]` when `BFF_ENABLED` is `False`. Emit `Error georama.bff.E001` for an empty effective `BFF_ALLOWED_VIEWER_HOSTS`; `Error georama.bff.E002` for `SESSION_COOKIE_SAMESITE == "None"` (case-insensitive, trimmed, distinct from Python `None`) with `SESSION_COOKIE_SECURE` false; `Error georama.bff.E003` for `SESSION_COOKIE_SAMESITE == "Strict"`; `Error georama.bff.E004` for an `OIDC_SERVER_URL` that is empty, whitespace, or not an absolute `http`/`https` URL with a host. Per allowlisted host, emit `Warning georama.bff.W001` when absent from `CORS_ALLOWED_ORIGINS` and `ALLOWED_HOSTS`, and `Warning georama.bff.W002` when not same-site with a configured `SESSION_COOKIE_DOMAIN`. Apply the single host-matching rule (case-insensitive equality ignoring port, or `.`-prefix suffix match) uniformly across E001/W001/W002. Perform no I/O; never raise.
    - _Requirements: 9.1, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8, 9.11, 9.12_
  - [ ] 4.2 Register the check module so Django's system check framework loads it
    - Import `georama.core.checks` from `CoreConfig.ready()` so `@register()` runs exactly once per process.
    - _Requirements: 9.1_
  - [ ]* 4.3 Write unit tests for every check id and the disabled-feature empty case
    - One test per message id using `override_settings`, plus `BFF_ENABLED=False` returning `[]` across varied other-setting combinations, plus order-independence of the message set.
    - _Requirements: 9.2, 9.9, 9.10_

- [ ] 5. Checkpoint - Ensure all tests pass

- [ ] 6. Implement the redirect-safety guard (`GeoramaAccountAdapter.is_safe_url`)
  - This is the security-critical piece and must land with its parametrised bypass corpus; do not defer the tests to a later task.
  - [ ] 6.1 Implement `GeoramaAccountAdapter.is_safe_url` in `georama/core/auth/bff.py`
    - Follow the corrected shape from design.md exactly: call `super().is_safe_url(url)` first; return `False` on `None`/empty; strip `"*"` from `BFF_ALLOWED_VIEWER_HOSTS` and return `False` if nothing remains; guard `urlsplit` against `ValueError`; return `False` on an empty `netloc`; split host/port with `split_domain_port`; match the host with `validate_host`; finally re-delegate to `url_has_allowed_host_and_scheme(url, allowed_hosts={netloc}, require_https=not settings.DEBUG)`. Emit exactly one `WARNING` log record on rejection containing only the parsed scheme and a truncated, control-character-escaped authority (or a fixed marker), never the full value.
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.8, 2.9, 2.10, 2.11, 2.12, 2.13, 2.14, 2.15, 11.2, 11.3_
  - [ ]* 6.2 Write parametrised bypass-corpus tests for `is_safe_url`
    - At least two cases each for: scheme confusion, protocol-relative URLs, userinfo authorities, host-suffix confusion, host-prefix confusion, control characters in the authority, host case variation, port variation, punycode/IDNA forms. Run the full corpus once with `DEBUG=True` and once with `DEBUG=False`; assert one expected boolean per case, reported individually. Place in `tests/unit/core/test_redirect_guard_corpus.py`.
    - **Validates Properties 1, 2, 3 (design.md)**
    - _Requirements: 14.4_
  - [ ]* 6.3 Write non-regression and totality tests for `is_safe_url`
    - Assert `super().is_safe_url(s) == True ⟹ S(s) == True` for relative URLs, the request host, `ALLOWED_HOSTS` patterns, and `CSRF_TRUSTED_ORIGINS`-derived hosts; assert `S` never raises, including inputs that make `urlsplit` raise `ValueError`; assert an empty or `["*"]`-only allowlist yields `False` for every input; assert monotonicity (adding a non-`"*"` host never turns a prior `True` into `False`). Place in `tests/unit/core/test_redirect_guard_regression.py`.
    - **Validates Property 4 (design.md)**
    - _Requirements: 2.6, 2.7, 10.1, 10.7_

- [ ] 7. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise. Confirm 100 percent branch coverage of `GeoramaAccountAdapter.is_safe_url` before moving on (Requirement 14.2) — this is the one piece of this feature that does not get a second chance.

- [ ] 8. Implement the `OidcLogin` login shim
  - [ ] 8.1 Implement `OidcLogin` in `georama/core/views/bff.py`
    - `GET`-only view. Read `came_from` (last occurrence if repeated); if the session is already authenticated, redirect to `came_from` when `adapter.is_safe_url()` accepts it (calling the hook directly, since allauth is not in the loop on this shortcut), else to `LOGIN_REDIRECT_URL`. Otherwise resolve the provider login URL via `reverse()` with `OIDC_PROVIDER_ID` as the `provider_id` kwarg; if `came_from` is absent, blank/whitespace-only, or longer than 8192 characters, redirect to that URL with no `next` parameter (logging a `WARNING` with the length only for the oversized case); otherwise redirect to that URL with `came_from` re-emitted as `next`, URL-encoded only. Respond `405` naming `GET` for any other method, and `500` if `OIDC_PROVIDER_ID` is unset or `reverse()` fails. Never log in/out, never touch the session beyond Django's own touch.
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10, 1.11, 1.12, 1.13, 3.3, 3.4_
  - [ ] 8.2 Add the upgrade-guard test pinning the allauth provider-login URL
    - Resolve the provider login URL through `reverse()` with `OIDC_PROVIDER_ID` as `provider_id` and assert the resolved path literally, so a URL-name or signature change in a future allauth version fails here rather than in production.
    - _Requirements: 14.3_
  - [ ]* 8.3 Write unit tests for `OidcLogin` branching behaviour
    - Cover: authenticated + accepted/rejected `came_from`; anonymous + absent/whitespace/oversized/normal `came_from`; repeated `came_from` query parameter (last occurrence wins); non-GET method; `BFF_ENABLED=False` (route absent, see task 11); misconfigured `OIDC_PROVIDER_ID`.
    - **Validates Properties 6, 14 (design.md)**
    - _Requirements: 1.4, 1.5, 1.6, 1.7, 1.9, 1.11, 1.12, 1.13, 3.1, 3.3_

- [ ] 9. Implement the `LoginUser` identity probe
  - [ ] 9.1 Implement `LoginUser` in `georama/core/views/bff.py`
    - `GET`-only, always HTTP 200 with a JSON object body, `Cache-Control: no-store`, `Vary: Cookie`. Anonymous: `username` absent/`null`/empty, `is_staff` false and `roles` empty if present. Authenticated: `username` (`request.user.get_username()`), `email`, `display_name` (full name or username), `is_staff`, `roles` (group names). When `BFF_LOGINUSER_VERIFY_IDP` is true, call `TokenStore.access_token_for` (task 14) and log the user out on failure. Wrap body construction so any unhandled error still yields `200` with an anonymous-shaped body and an `ERROR` log record. Never include `SocialToken.token`/`token_secret`, `SocialAccount.extra_data`, or session keys.
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 4.10, 4.11, 4.12_
  - [ ]* 9.2 Write unit tests for `LoginUser` response shape and headers
    - Cover: anonymous shape; authenticated shape including `email`/`display_name`/`is_staff`/`roles`; no session cookie, expired cookie, unresolvable cookie, malformed cookie all yielding `200` JSON; headers present; no secret-bearing key in the body; read-only (no `User`/`SocialAccount` writes) on anonymous probes. Mark the `BFF_LOGINUSER_VERIFY_IDP=True` branch's behaviour (opt-in, default off) as its own case.
    - **Validates Properties 8, 9, 10, 13 (design.md)**
    - _Requirements: 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 4.11, 4.12_

- [ ] 10. Implement the dual-mode `Logout` view
  - [ ] 10.1 Implement `Logout` in `georama/core/views/bff.py`, replacing `georama/core/views/auth.py:Logout`
    - `GET` and `POST`. Terminate via `get_adapter().logout(request)` before selecting a response shape. If `Sec-Fetch-Mode` (compared case-sensitively) equals `navigate`, respond `302` to `reverse("landing")`. Otherwise respond `200` with `{"loggedOut": true, "wasAuthenticated": <bool>}`, adding `idpLogoutUrl` verbatim when `BFF_IDP_LOGOUT_URL` is non-empty. Clear the session cookie with the attributes it was issued with. Remove the now-unused `Logout` class from `georama/core/views/auth.py`.
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9, 5.10, 5.12, 12.3, 12.4_
  - [ ]* 10.2 Write unit tests for `Logout` branching and idempotence
    - Cover: `Sec-Fetch-Mode` absent/`navigate`/other values; repeated calls on the same cookie (`wasAuthenticated` false after the first, every call answered); anonymous caller; session actually flushed (a replayed old cookie reports logged out at `loginuser`); `idpLogoutUrl` present only when configured, returned unmodified.
    - **Validates Properties 11, 12 (design.md)**
    - _Requirements: 5.3, 5.4, 5.6, 5.7, 5.8, 5.10_

- [ ] 11. Wire the three BFF routes into `georama/core/urls.py`
  - [ ] 11.1 Add literal `path()` entries for `oidc/login`, `loginuser`, `logout`
    - Register `path("oidc/login", bff.OidcLogin.as_view(), name="bff-oidc-login")`, `path("loginuser", bff.LoginUser.as_view(), name="bff-loginuser")`, and `path("logout", bff.Logout.as_view(), name="logout")` (replacing the current `auth.Logout` registration), matching exactly with no trailing slash since `APPEND_SLASH=False`. Gate `oidc/login` and `loginuser` so they resolve to `404` when `BFF_ENABLED` is false (`logout` keeps serving its JSON/redirect behaviour regardless, since it has no BFF-only contract beyond the JSON branch).
    - _Requirements: 1.1, 1.10, 4.1, 5.1_
  - [ ]* 11.2 Write a unit test asserting `404` at `oidc/login` and `loginuser` when `BFF_ENABLED` is false
    - Confirm pre-existing URL configuration and login behaviour otherwise stay in force.
    - _Requirements: 1.10_

- [ ] 12. Checkpoint - Ensure all tests pass

- [ ] 13. Adapt Cognito/OIDC specifics in `GeoGirafeAuthenticationMiddleware`
  - [ ] 13.1 Guard the eager discovery fetch against startup failure
    - Wrap `__init__`'s discovery fetch with the configured 5-second timeout; on a failed or non-2xx fetch, log at `ERROR` and complete initialisation without raising, so georama still boots and serves anonymous and Bearer-authenticated traffic. Fetch discovery lazily (at most once) on first use at `oidc/login` if it was not available at startup.
    - _Requirements: 12.6, 12.8_
  - [ ] 13.2 Derive the username from `sub` when `preferred_username` is absent
    - In `create_user_via_allauth`, fall back to deriving the username from the `sub` claim when `preferred_username` is missing, creating no duplicate `User` for a `sub` already stored; continue resolving existing accounts by the `(provider, uid)` pair regardless of `preferred_username`.
    - _Requirements: 12.10_
  - [ ] 13.3 Support `AUTH_PARAMS` `prompt=login` and tolerate an absent `end_session_endpoint`
    - Ensure the configured `AUTH_PARAMS` (including `prompt=login` when set) reach every authorization request. Access `end_session_endpoint` (and any other discovery key) through absent-tolerant lookups everywhere it is read, so a Cognito-shaped discovery document missing that key never raises.
    - _Requirements: 12.2, 12.5_
  - [ ]* 13.4 Write unit tests for discovery failure, the `preferred_username` fallback, and the absent `end_session_endpoint`
    - Cover: unreachable/slow/non-2xx discovery at startup not raising; `oidc/login` triggering exactly one lazy discovery fetch when none loaded; `sub`-derived username creating one `User` per `sub`; login and logout completing with no `KeyError` against a discovery document missing `end_session_endpoint`.
    - _Requirements: 12.2, 12.6, 12.7, 12.8, 12.9, 12.10_

- [ ] 14. Implement `TokenStore`
  - `TokenStore` is not on the critical path of `loginuser`, `themes.json`, or `/webgis/maps` (Requirements 7.8, 7.13), so it can land independently of the endpoint tasks above.
  - [ ] 14.1 Implement `TokenStore.get`/`is_expired`/`refresh`/`access_token_for` in `georama/core/auth/bff.py`
    - Read/write through `SocialToken` (`token`, `token_secret`, `expires_at`). `access_token_for`: return the stored token when `expires_at` is more than the 30-second leeway away; otherwise attempt exactly one refresh via a server-to-server POST to the discovery `token_endpoint` with `client_id`/`client_secret`, 5-second connect and read timeouts. On success, write `token` and `expires_at` in one transaction, replacing `token_secret` only if a new refresh token was returned. On a connection error, non-2xx non-rejection response, or timeout: log `WARNING` naming the failure class (no token value), return `None`, leave all rows unchanged. On an explicit invalid/expired/revoked rejection: clear `token` and `token_secret`, return `None`. Never raise.
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.9, 7.10, 7.12_
  - [ ] 14.2 Add row-level locking for concurrent refresh callers
    - Serialize concurrent callers that each find the stored token within the leeway on a row-level lock over the `SocialToken` row held for the duration of the exchange, so at most one refresh reaches the identity provider and every waiting caller returns the single refreshed token or `None`.
    - _Requirements: 7.11_
  - [ ]* 14.3 Write unit tests for `TokenStore` expiry, refresh outcomes, and concurrency
    - Cover: fresh token returned with no network call; missing token/empty refresh token returning `None` with no call; successful refresh updating `token`/`expires_at` and conditionally `token_secret`; connection error, non-2xx, and timeout all returning `None` with rows unchanged and a `WARNING` log; explicit rejection clearing both token fields; concurrent callers around expiry triggering exactly one exchange.
    - **Validates Property 24 (design.md)**
    - _Requirements: 7.2, 7.3, 7.4, 7.5, 7.11, 7.12_

- [ ] 15. Re-enable CSRF protection
  - This is the highest-risk task for existing deployments: it is not gated behind `BFF_ENABLED` (Requirement 10.10's closed list of ungated changes), so its exemptions must land in this same task, never after the middleware switch.
  - [ ] 15.1 Enable `CsrfViewMiddleware` in `georama/core/settings.py`
    - Uncomment `django.middleware.csrf.CsrfViewMiddleware` in `Base.MIDDLEWARE`, positioned per Django's documented ordering.
    - _Requirements: 8.1_
  - [ ] 15.2 Implement the `require_viewer_origin` decorator (Origin_Guard)
    - New `georama/core/decorators/viewer_origin.py`. For an unsafe-method request to a decorated view, pass through when `Origin` equals georama's own origin or matches an entry of `BFF_ALLOWED_VIEWER_HOSTS` (including `.`-prefixed patterns); respond `403` without executing the view when `Origin` is absent or matches neither, deciding on `Origin` alone and never consulting `Referer`.
    - _Requirements: 8.5, 8.6_
  - [ ] 15.3 Apply `csrf_exempt` + `require_viewer_origin` to the two narrow exemptions
    - `@method_decorator(csrf_exempt, name="dispatch")` and `@method_decorator(require_viewer_origin, name="dispatch")` on `UrlShortenerCreate` (`georama/webgis/views.py`) and on the WFS `GetFeature` POST view (`georama/maps/views.py`, `OgcServer.post` around line 394). Confirm every other POST/PUT/PATCH/DELETE endpoint (`/login`, the two permission endpoints, the project-delete endpoint, entity form endpoints) is left validating a CSRF token — no additional exemptions.
    - _Requirements: 8.2, 8.3, 8.4, 8.7, 8.9_
  - [ ]* 15.4 Write unit tests for CSRF-protected server-rendered endpoints
    - For `/login`, the permission assign/remove endpoints, the project-delete endpoint, and an entity form endpoint: a valid token still produces the pre-change status and persisted side effects; an absent, malformed, or non-matching token now produces `403` with the view unexecuted and no database write. Include a case where the request `Origin` is in `CSRF_TRUSTED_ORIGINS` but the token is still required.
    - **Validates Property 20 (design.md)**
    - _Requirements: 8.2, 8.3, 8.7_
  - [ ]* 15.5 Write unit tests for the two exempted routes under allowed/disallowed `Origin`
    - WFS `GetFeature` POST and `/webgis/short/create`: `2xx` with `CsrfViewMiddleware` enabled and no token, from an `Origin` equal to georama's own origin or an allowlisted viewer host; `403` without the view running for an absent or non-allowlisted `Origin`, independent of any `Referer` header.
    - **Validates Property 21 (design.md)**
    - _Requirements: 8.4, 8.5, 8.6_

- [ ] 16. Implement provider-identifier migration tooling
  - Independent of the endpoint tasks above; relevant only to deployments changing `OIDC_PROVIDER_ID` on data that already has `SocialAccount` rows (e.g. Keycloak to Cognito).
  - [ ] 16.1 Write a reversible data migration re-pointing `SocialAccount.provider`/`uid` and cleaning up superseded tokens
    - Re-point `SocialAccount.provider` from the previous value to the new value; re-point `uid` per an operator-supplied previous-`sub`-to-new-`sub` mapping, leaving `uid` unchanged for rows absent from that mapping; delete the `SocialToken` rows belonging to the re-pointed accounts, since they are not redeemable at the new provider. Implement the reverse migration.
    - _Requirements: 13.2, 13.6_
  - [ ] 16.2 Write a management command reporting `SocialAccount` row counts before and after the migration
    - A database-touching management command (not a system check, which performs no database access) reporting the count of rows carrying the previous provider value, run before and after the migration; document the expectation of non-zero before and zero after.
    - _Requirements: 13.4_
  - [ ]* 16.3 Write unit tests for the migration and the management command
    - Cover: rows re-pointed per the mapping; rows outside the mapping left with their original `uid`; `SocialToken` rows deleted for migrated accounts; the reverse migration restoring the previous state; the command's before/after counts.
    - _Requirements: 13.2, 13.4, 13.6_

- [ ] 17. Checkpoint - Ensure all tests pass, ask the user if questions arise
  - This checkpoint follows the highest-risk task (15) and the migration tooling (16). Confirm the backward-compatibility baseline from task 1 still passes unchanged before continuing.

- [ ] 18. Integration tests for the full login/logout round trip and import-ban enforcement
  - [ ] 18.1 Build local test doubles for the discovery, authorization, and token endpoints
    - A local responder serving a discovery document (Cognito-shaped, `end_session_endpoint` absent), `/authorize`, and `/oauth2/token`, used by the tests below so no outbound network request occurs in the default `pytest` selection.
    - _Requirements: 14.7_
  - [ ] 18.2 Write an integration test for `redirect_uri` fragment-freedom and `came_from` byte-for-byte preservation
    - Drive the full round trip through `OidcLogin`, the test-double provider, and the callback; assert the `redirect_uri` sent in both the authorization request and the token exchange contains no `#` and no query component; assert the final `Location` equals the original `came_from` character for character, including its query string, fragment, and percent-encoded octets.
    - **Validates Properties 5, 7 (design.md)**
    - _Requirements: 1.2, 1.7, 3.1, 3.2, 3.4_
  - [ ] 18.3 Write an integration test for concurrent pending-flow and stale-state handling
    - Start at least 3 concurrent login flows from one session and assert each completing flow returns to its own recorded `came_from`; assert a flow evicted by exceeding the retained maximum is handled as an unrecognized `state`; assert a callback with an absent, unrecognized, or expired (>600s) `state` leaves the request unauthenticated with a non-5xx response and redirects nowhere taken from that request.
    - **Validates Properties 22, 23 (design.md)**
    - _Requirements: 3.7, 3.8, 3.9_
  - [ ] 18.4 Write a test enforcing the allauth private-API import ban
    - Statically scan the georama source tree (e.g. via `ast` on each module under `georama/`) and fail if any module imports from `allauth.account.internal` or `allauth.socialaccount.internal`.
    - _Requirements: 14.6_

- [ ] 19. Checkpoint - Ensure all tests pass
  - Confirm coverage thresholds: `fail_under` at or above 13, at least 90 percent branch coverage of the modules this feature adds, 100 percent branch coverage of `GeoramaAccountAdapter.is_safe_url`.

- [ ] 20. Write operator documentation
  - [ ] 20.1 Document every BFF/session/secret environment variable
    - List each variable declared in tasks 3 and the OIDC client secret, with its type, default, and whether BFF mode requires a non-default value.
    - _Requirements: 6.8, 11.12_
  - [ ] 20.2 Document the three deployment topologies and the ungated settings changes
    - Same-origin, same-site, and cross-site topologies with their cookie/CORS settings and risk notes; the closed list of changes this feature applies outside the `BFF_ENABLED` gate (`CsrfViewMiddleware`, the `SESSION_COOKIE_*`/`CSRF_COOKIE_*` declarations, `SESSION_SAVE_EVERY_REQUEST`, `ACCOUNT_ADAPTER`) and the operator action required to preserve prior behaviour on a plain-HTTP deployment.
    - _Requirements: 6.5, 6.6, 10.12_
  - [ ] 20.3 Document the Cognito global sign-out limitation and the provider-migration procedure
    - Record that viewer logout terminates only the georama session, that the Cognito session may allow a silent subsequent SSO login, and the `prompt=login` / navigation-based-logout / shortened-session mitigations; record the `OIDC_PROVIDER_ID`-change procedure (task 16) and the outcome of skipping it.
    - _Requirements: 12.3, 13.4_

- [ ] 21. Final checkpoint - Ensure all tests pass, ask the user if questions arise

## Notes

- Tasks marked with `*` are optional test-related sub-tasks and can be skipped for a faster MVP, except task 1, which captures a pre-change behavioural baseline and is intentionally not marked optional (see Overview).
- No property-based testing library is added anywhere in this plan (Requirement 14.2); properties 1–24 from design.md are discharged by parametrised `pytest` cases, called out by number next to the test task that covers them.
- Each task references the granular acceptance criteria it implements for traceability back to `requirements.md`.
- Checkpoints ensure incremental validation; task 17 in particular gates on the highest-risk, ungated CSRF change.
- Tasks 13, 14, 15, and 16 touch disjoint files and have no dependency on one another; they are independent once task 11 (URL wiring) is complete and can be executed in any order or in parallel.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2", "1.3", "1.4"] },
    { "id": 1, "tasks": ["3.1"] },
    { "id": 2, "tasks": ["3.2"] },
    { "id": 3, "tasks": ["3.3"] },
    { "id": 4, "tasks": ["3.4", "4.1"] },
    { "id": 5, "tasks": ["4.2"] },
    { "id": 6, "tasks": ["4.3"] },
    { "id": 7, "tasks": ["6.1"] },
    { "id": 8, "tasks": ["6.2", "6.3"] },
    { "id": 9, "tasks": ["8.1"] },
    { "id": 10, "tasks": ["8.2", "8.3"] },
    { "id": 11, "tasks": ["9.1"] },
    { "id": 12, "tasks": ["9.2"] },
    { "id": 13, "tasks": ["10.1"] },
    { "id": 14, "tasks": ["10.2"] },
    { "id": 15, "tasks": ["11.1"] },
    { "id": 16, "tasks": ["11.2"] },
    { "id": 17, "tasks": ["13.1", "14.1", "15.1", "15.2", "16.1", "16.2"] },
    { "id": 18, "tasks": ["13.2", "14.2", "15.3", "16.3"] },
    { "id": 19, "tasks": ["13.3", "14.3", "15.4", "15.5"] },
    { "id": 20, "tasks": ["13.4"] },
    { "id": 21, "tasks": ["18.1"] },
    { "id": 22, "tasks": ["18.2", "18.3", "18.4"] },
    { "id": 23, "tasks": ["20.1"] },
    { "id": 24, "tasks": ["20.2"] },
    { "id": 25, "tasks": ["20.3"] }
  ]
}
```
