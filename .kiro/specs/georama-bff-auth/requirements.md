# Requirements Document

## Introduction

Georama becomes the Backend-For-Frontend (BFF) and the single confidential OAuth client against AWS Cognito. It performs the authorization code exchange server-side with its client secret, keeps every Cognito token in the database, and hands the gg-viewer SPA nothing but an `HttpOnly` Django session cookie. The viewer then reaches protected resources (`themes.json`, `/webgis/maps`) with that cookie alone.

The driver is concrete: Cognito rejects gg-viewer's current browser-side OIDC login because the viewer's `redirect_uri` carries a URL fragment, and Cognito's discovery document omits `end_session_endpoint` so the viewer's logout throws. The BFF topology removes both problems structurally — the only `redirect_uri` Cognito ever sees is georama's existing fragment-free allauth callback, and logout becomes a server-side session concern.

The viewer side is already implemented upstream. **gg-viewer requires zero code changes**: its `GMFConnectManager` is activated by the `gmfauth` config block and hardcodes three backend paths — `oidc/login`, `loginuser`, `logout` — which georama must serve, plus a session cookie usable from the viewer's origin.

Scope is deliberately small. django-allauth 65.16.1, already installed, supplies the return-URL plumbing (`next` parameter, safety validation, OAuth-state propagation) and the token store. What is bespoke is three viewer-facing endpoint shapes, one safety hook override, settings, and system checks. **No new runtime or test dependency is introduced** — this is a stated design constraint, not an accident, and it includes the deliberate rejection of a property-based testing library.

These requirements are derived from the approved `design.md` in this directory. Every component, formal specification, and correctness property in that document maps to at least one acceptance criterion below.

## Glossary

- **BFF (Backend-For-Frontend)**: a server-side component that acts as the OAuth client on behalf of a browser application, holds the resulting tokens, and exposes session-cookie-authenticated endpoints to that application. Here, georama.
- **Confidential client**: an OAuth client that can keep a `client_secret` and therefore authenticates itself at the token endpoint. Georama in BFF mode.
- **Public client**: an OAuth client that cannot keep a secret, such as a browser SPA. gg-viewer's current mode, which this feature removes.
- **Georama_BFF**: the georama Django application operating with `BFF_ENABLED` true.
- **Viewer**: the gg-viewer SPA, configured with a `gmfauth` block and `loginMode=oidc`.
- **Identity_Provider**: the OIDC authorization server. AWS Cognito for this feature, previously Keycloak.
- **Login_Shim**: the `oidc/login` view. Renames `came_from` to `next` and hands off to django-allauth.
- **Redirect_Guard**: `GeoramaAccountAdapter.is_safe_url`, the allauth adapter hook deciding which absolute redirect targets allauth will honour.
- **Identity_Probe**: the `loginuser` view, a GMF-compatible "who am I" endpoint.
- **Logout_Endpoint**: the `logout` view, serving both the viewer's `fetch()` and server-rendered browser navigation.
- **Token_Store**: the component owning read, expiry evaluation, and refresh of Cognito tokens held in allauth's `SocialToken`.
- **Auth_Middleware**: the existing `GeoGirafeAuthenticationMiddleware`, which resolves Bearer tokens and short-circuits when a session has already authenticated the request.
- **Origin_Guard**: the `require_viewer_origin` decorator applied to the two CSRF-exempt endpoints.
- **Startup_Check**: `check_bff_configuration`, registered with Django's system check framework.
- **`came_from`**: the query parameter gg-viewer appends when navigating to `oidc/login`, holding the viewer URL to return to. It carries viewer application state in a URL fragment and `?authentified=true` in its query.
- **`next`**: django-allauth's return-URL query parameter. Validated once by `adapter.is_safe_url` through the public `allauth.account.utils.get_next_redirect_url` entry point, then carried across the Identity_Provider round trip inside django-allauth's integrity-protected **OAuth state**, reached only through the public `Provider.redirect` entry point, rather than in the `redirect_uri`.
- **Audience**: the gg-viewer `gmfauth.audience` host list. The viewer's service worker sends `credentials: include` on requests to those hosts.
- **Same-origin**: identical scheme, host, and port. No CORS and no cookie policy question arises.
- **Same-site**: different hosts sharing a registrable domain (for example `app.georama.example.org` and `georama.example.org`). The session cookie remains first-party; CORS is still required.
- **Cross-site**: different registrable domains. The session cookie becomes a third-party cookie requiring `SameSite=None; Secure`.
- **RP-initiated logout**: the OIDC flow where the relying party redirects the user to the provider's `end_session_endpoint` to terminate the provider session. Unavailable with Cognito, whose discovery document omits that key.

## Out of Scope

The following are explicitly excluded from this feature and recorded here so the exclusion is a decision rather than an omission:

- **The gg-viewer `redirect_uri` fragment fix.** Moving viewer application state out of the fragment (or into the `state` parameter) will be filed upstream independently. It requires an upstream merge and a viewer release, and would still leave tokens in the browser and still leave Cognito's missing `end_session_endpoint` to handle.
- **Adopting `allauth.headless`.** Evaluated and rejected in design.md's Package evaluation: its provider-redirect view is POST-only (the Viewer initiates login with a GET navigation), its session response shape does not match `GMFManager.getUserInfo`, and its logout responds 401 on success while the Viewer throws on any non-2xx. Recorded as the natural migration target if gg-viewer ever gains a native headless mode.
- **Making the three backend path names configurable in gg-viewer's `gmfauth` block.** Cleaner as an upstream contribution; requires a viewer release.
- **Running two OIDC providers concurrently.** The current single-app `SOCIALACCOUNT_PROVIDERS` property does not model it; a side-by-side Keycloak and Cognito deployment is a separate change.
- **Serving the gg-viewer bundle from Django** to achieve the same-origin end state. The stub exists (`webgis/views.py:465`) and is the documented target topology, but implementing it is not part of this feature.

## Requirements

### Requirement 1: BFF login entry point

**User Story:** As a gg-viewer user, I want to start login by navigating to georama, so that I authenticate without the browser application ever acting as an OAuth client.

#### Acceptance Criteria

1. WHERE `BFF_ENABLED` is true, THE Georama_BFF SHALL serve HTTP GET at the path `oidc/login` relative to the georama root, matched literally without a trailing slash because `APPEND_SLASH` is false, and SHALL emit the response to every such request within 500 ms of receiving it, measured server-side and excluding network transit.
2. WHEN a GET request arrives at `oidc/login` without an authenticated session and with a `came_from` query parameter whose value contains at least one non-whitespace character and is at most 8192 characters long, THE Login_Shim SHALL respond with HTTP 302 whose `Location` is the django-allauth provider login URL carrying that value as a URL-encoded `next` query parameter.
3. THE Login_Shim SHALL obtain the django-allauth provider login URL from Django `reverse()` with the configured `OIDC_PROVIDER_ID` as the `provider_id` argument.
4. WHEN a GET request arrives at `oidc/login` with `came_from` absent, empty, or consisting only of whitespace characters (space, tab, carriage return, line feed), THE Login_Shim SHALL respond with HTTP 302 to the provider login URL carrying no `next` parameter.
5. IF a GET request at `oidc/login` carries an authenticated session and a `came_from` value that Redirect_Guard accepts, THEN THE Login_Shim SHALL respond with HTTP 302 to that `came_from` value and SHALL complete the request without any request to the Identity_Provider.
6. IF a GET request at `oidc/login` carries an authenticated session and a `came_from` value that is absent, empty, whitespace-only, longer than 8192 characters, or rejected by Redirect_Guard, THEN THE Login_Shim SHALL respond with HTTP 302 to `LOGIN_REDIRECT_URL` and SHALL complete the request without any request to the Identity_Provider.
7. THE Login_Shim SHALL select the last occurrence of `came_from` when the query string repeats the parameter, and SHALL apply URL encoding as the only transformation to the selected value, so the `next` parameter decodes to the received value character for character, including its query string and its URL fragment.
8. THE Login_Shim SHALL leave `request.user` as the middleware resolved it, SHALL not invoke Django login or logout, SHALL not rotate or flush the session key, and SHALL delegate every authentication state change to the django-allauth callback.
9. THE Login_Shim SHALL carry the return URL only in django-allauth's `next` parameter and OAuth state, leaving the georama session contents after the response identical to their contents before it apart from Django's own session touch, with no feature-owned return-URL key present.
10. WHERE `BFF_ENABLED` is false, THE Georama_BFF SHALL leave the pre-existing URL configuration and login behaviour in force, SHALL resolve no route at `oidc/login`, and SHALL respond with HTTP 404 to any request at that path.
11. IF a request arrives at `oidc/login` with any HTTP method other than GET, THEN THE Login_Shim SHALL respond with HTTP 405 naming GET as the only allowed method, SHALL make no request to the Identity_Provider, and SHALL leave the session unmodified.
12. IF a GET request arrives at `oidc/login` with a `came_from` value longer than 8192 characters, THEN THE Login_Shim SHALL respond with HTTP 302 to the provider login URL carrying no `next` parameter and SHALL emit a log record at WARNING level stating the received length and omitting the received value.
13. IF `OIDC_PROVIDER_ID` is unset or empty, or Django `reverse()` resolves no provider login URL for its value, THEN THE Login_Shim SHALL respond with HTTP 500 and an error indication identifying the login provider as misconfigured, SHALL make no request to the Identity_Provider, and SHALL leave the session unmodified.

### Requirement 2: Redirect-target safety

**User Story:** As a georama operator, I want a single auditable rule deciding where a user's browser may be sent after authentication, so that an attacker-supplied return URL cannot become an open redirect and existing login flows keep working.

#### Acceptance Criteria

1. IF `DefaultAccountAdapter.is_safe_url` returns true for the received value, THEN THE Redirect_Guard SHALL return true without consulting `BFF_ALLOWED_VIEWER_HOSTS`, preserving acceptance of relative URLs, the request host, `ALLOWED_HOSTS` patterns including `.`-prefixed subdomain forms, and hosts derived from `CSRF_TRUSTED_ORIGINS`, and preserving the exclusion of the `"*"` wildcard.
2. WHEN `DefaultAccountAdapter.is_safe_url` returns false for a received value carrying an authority component, THE Redirect_Guard SHALL return true if and only if the authority's host matches an entry of `BFF_ALLOWED_VIEWER_HOSTS` under `django.http.request.validate_host` semantics and `django.utils.http.url_has_allowed_host_and_scheme` then accepts that value against the parsed authority with HTTPS required unless `DEBUG` is true.
3. WHILE `DEBUG` is false, THE Redirect_Guard SHALL accept through the `BFF_ALLOWED_VIEWER_HOSTS` branch only a received value whose scheme is `https`, returning false for an `http` value and for a scheme-less protocol-relative value of the form `//host/path`, whatever that value's authority.
4. WHILE `DEBUG` is true, THE Redirect_Guard SHALL accept through the `BFF_ALLOWED_VIEWER_HOSTS` branch the schemes `http` and `https` and no other scheme, treating a scheme-less protocol-relative value as `http`.
5. THE Redirect_Guard SHALL delegate host matching to `django.http.request.validate_host` and SHALL delegate URL parsing and scheme checking to one call of `django.utils.http.url_has_allowed_host_and_scheme`, applying no percent-decoding, no Unicode-to-ASCII host conversion, and no other rewriting of the received value.
6. THE Redirect_Guard SHALL return `True` or `False` for every received value, raising no exception, performing no network access, and performing no database query, including for `None`, the empty string, a whitespace-only string, a string of 8192 characters, a value whose parsing raises `ValueError`, protocol-relative URLs, authorities embedding user information, host-suffix and host-prefix confusion strings, non-HTTP schemes, and strings containing control characters.
7. THE Redirect_Guard SHALL return true only for a received value that `DefaultAccountAdapter.is_safe_url` accepted, or whose scheme is `http` or `https` and whose authority host matches an entry of `BFF_ALLOWED_VIEWER_HOSTS`.
8. THE Redirect_Guard SHALL leave the received value, the request, the session, and every persisted model unmodified, and SHALL write no response header.
9. WHEN a host is added to `BFF_ALLOWED_VIEWER_HOSTS` while `DEBUG`, `ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS`, and the request host stay unchanged, THE Redirect_Guard SHALL continue to return true for every received value for which it returned true before that addition.
10. WHILE every entry of `BFF_ALLOWED_VIEWER_HOSTS` is absent, empty, whitespace-only, or the `"*"` wildcard, THE Redirect_Guard SHALL return exactly the result of `DefaultAccountAdapter.is_safe_url` for every received value.
11. IF the Redirect_Guard returns false, THEN THE Redirect_Guard SHALL emit exactly one log record at WARNING level holding the parsed scheme and the parsed authority truncated to 253 characters with control characters escaped, or a fixed marker when no authority could be parsed, and SHALL omit the path, the query, the fragment, and the full received value.
12. WHEN the received value's authority host differs from an entry of `BFF_ALLOWED_VIEWER_HOSTS` only by letter case, THE Redirect_Guard SHALL treat that host as matching that entry, because `validate_host` compares both sides case-insensitively.
13. IF the received value's authority differs from every entry of `BFF_ALLOWED_VIEWER_HOSTS` by Unicode-versus-`xn--` encoding of a label, by an explicit default port `:80` or `:443`, by a trailing dot, by percent-encoding, or by user information preceding `@`, THEN THE Redirect_Guard SHALL return false, and an operator requiring one of those spellings SHALL declare it literally in `BFF_ALLOWED_VIEWER_HOSTS`.
14. IF an entry of `BFF_ALLOWED_VIEWER_HOSTS` is `"*"`, empty, or whitespace-only, THEN THE Redirect_Guard SHALL discard that entry before matching, so it matches no authority, because `validate_host` would otherwise treat `"*"` as matching every host and yield an unconditional open redirect.
15. IF the received value begins with a character in Unicode category C, begins with three or more consecutive slash characters, or resolves to a different authority when each backslash is read as a forward slash, THEN THE Redirect_Guard SHALL return false.

### Requirement 3: Return-URL propagation across the Identity_Provider round trip

**User Story:** As a gg-viewer user, I want my map state to survive login, so that I return to the exact view I left, and as an operator I want the `redirect_uri` to stay fragment-free so Cognito accepts it.

#### Acceptance Criteria

1. WHEN a login completes successfully for a `came_from` value of at most 2048 characters that Redirect_Guard accepted, THE Georama_BFF SHALL respond with HTTP status 302 whose `Location` header equals that `came_from` value character for character, including its query string, its URL fragment, and every percent-encoded octet it contains such as `%23` and `%26`, applying no decoding, re-encoding, reordering, or normalization.
2. THE Georama_BFF SHALL send the Identity_Provider a `redirect_uri` equal to the django-allauth callback `/accounts/oidc/<provider_id>/login/callback/` on the georama origin, containing no `#` character, no fragment component, and no query component, and byte-identical in the authorization request and in the token exchange request.
3. IF the `came_from` value is absent, empty, longer than 2048 characters, not parseable as an HTTP or HTTPS URL, or rejected by Redirect_Guard, THEN THE Georama_BFF SHALL complete the login, establish the authenticated session, and respond with HTTP status 302 whose `Location` is `LOGIN_REDIRECT_URL`.
4. THE Georama_BFF SHALL carry the return URL across the Identity_Provider round trip inside django-allauth's integrity-protected OAuth state, placing no `came_from`, `next`, or other return-URL value in any query parameter of the authorization request.
5. WHERE `SOCIALACCOUNT_LOGIN_ON_GET` is true, WHEN the Viewer navigates by HTTP GET to the django-allauth provider login URL, THE Georama_BFF SHALL respond with HTTP status 302 whose `Location` is the Identity_Provider authorization endpoint, serving no intermediate page that requires a user action to continue.
6. THE Georama_BFF SHALL perform the authorization code exchange as a server-to-server request to the Identity_Provider `token_endpoint` authenticated with `client_id` and `client_secret`, so that no access token, ID token, or refresh token value appears in any response header, response body, redirect `Location`, or cookie the browser receives.
7. IF a callback request from the Identity_Provider carries a `state` value that is absent, unrecognized for the request's session, or older than the pending-flow lifetime of 600 seconds, THEN THE Georama_BFF SHALL leave the request unauthenticated, SHALL redirect to no URL taken from that request, and SHALL respond with a status outside the 5xx range and a page indicating that login could not be completed and can be retried.
8. WHEN two or more login flows started from the same session are pending at the same time, THE Georama_BFF SHALL return each completing flow to the `came_from` value recorded for that flow's own `state`, retaining at least 3 concurrent pending flows per session.
9. IF the number of pending login flows for one session exceeds the retained maximum, THEN THE Georama_BFF SHALL discard the oldest pending flow and SHALL handle a later callback for that flow as a callback carrying an unrecognized `state`.

### Requirement 4: Identity probe endpoint

**User Story:** As a gg-viewer instance, I want a session identity endpoint that always answers with JSON, so that I can determine login state and poll session liveness without status-code handling.

#### Acceptance Criteria

1. WHERE `BFF_ENABLED` is true, THE Georama_BFF SHALL serve HTTP GET at the path `loginuser` relative to the georama root, matched literally without a trailing slash.
2. THE Identity_Probe SHALL respond to every GET request with HTTP status 200, a `Content-Type` header whose media type is `application/json`, and a body that parses as a single top-level JSON object, for every request whatever cookies or headers it carries, including a request with no session cookie, with an expired session cookie, with a session cookie whose key resolves to no server-side session, and with a malformed or unparseable session cookie value.
3. WHEN the request carries an authenticated session, THE Identity_Probe SHALL return a body whose `username` value is a non-empty string equal to `request.user.get_username()`.
4. WHEN the request carries no authenticated session, THE Identity_Probe SHALL return a body in which `username` is absent, `null`, or the empty string, which the Viewer reads as logged out, and in which `is_staff` is false and `roles` is empty if those fields are present.
5. WHEN the request carries an authenticated session, THE Identity_Probe SHALL additionally include in the body `email` holding the `User.email` value or `null` when that value is empty, `display_name` holding the user's full name or the `username` value when the full name is empty, `is_staff` holding the `User.is_staff` boolean, and `roles` holding the names of the Django groups the user belongs to as a JSON array, empty when the user belongs to none.
6. THE Identity_Probe SHALL set a `Cache-Control` header containing `no-store` and a `Vary` header containing `Cookie`.
7. THE Identity_Probe SHALL restrict the response body to identity fields, keeping `SocialToken.token`, `SocialToken.token_secret`, `SocialAccount.extra_data`, session keys, and permission internals out of it.
8. WHEN the request carries no authenticated session, THE Identity_Probe SHALL leave the `User` table, the `SocialAccount` table, and the authentication state of the request unchanged.
9. WHERE `BFF_LOGINUSER_VERIFY_IDP` is false, THE Identity_Probe SHALL answer from the Django session alone, performing no request to the Identity_Provider, no call to Token_Store, and at most one session read and one group query per request.
10. WHERE `BFF_LOGINUSER_VERIFY_IDP` is true, IF Token_Store does not supply a valid access token for an authenticated user originating from the Identity_Provider within 5 seconds of being called, THEN THE Identity_Probe SHALL log that user out, clear the session cookie, and respond with HTTP status 200 and a body in which `username` is absent, `null`, or the empty string.
11. IF an unhandled error occurs while the Identity_Probe builds its response, THEN THE Identity_Probe SHALL still respond with HTTP status 200, a `Content-Type` header whose media type is `application/json`, and a body that parses as a JSON object in which `username` is absent, `null`, or the empty string, and SHALL emit a log record at ERROR level, so the Viewer's unchecked `r.json()` call never receives a 5xx or a non-JSON body.
12. WHEN GET requests arrive repeatedly for the same session at the Viewer's 10-minute poll interval or faster, THE Identity_Probe SHALL respond to each of them with HTTP status 200 and a JSON object body, applying no request-rate limiting or throttling that would yield any other status or a non-JSON body.

### Requirement 5: Logout endpoint

**User Story:** As a gg-viewer instance, I want logout to answer with a 2xx status, and as a georama user browsing directly, I want logout to return me to the landing page, so that one path serves both callers.

#### Acceptance Criteria

1. WHERE `BFF_ENABLED` is true, THE Georama_BFF SHALL serve HTTP GET and HTTP POST at the path `logout` relative to the georama root, matched literally without a trailing slash, replacing the existing logout view.
2. THE Logout_Endpoint SHALL terminate the session through django-allauth's `get_adapter().logout(request)`, so allauth's logout signal handling and cleanup run, and SHALL perform that termination before it selects a response shape, for every request whatever its HTTP method, its `Sec-Fetch-Mode` value, and its authentication state.
3. WHEN the request header `Sec-Fetch-Mode` is absent or holds any value other than the lowercase token `navigate` compared case-sensitively, THE Logout_Endpoint SHALL respond with HTTP 200 and a body parsing as a JSON object whose `loggedOut` value is true and whose `wasAuthenticated` value is true if and only if the request was authenticated before the termination required by criterion 2.
4. WHEN the request header `Sec-Fetch-Mode` holds exactly the lowercase token `navigate` compared case-sensitively, THE Logout_Endpoint SHALL respond with HTTP 302 whose `Location` is the URL that `reverse()` resolves for the URL name `landing`, which is the behaviour the existing server-rendered flow relies on.
5. WHERE `BFF_IDP_LOGOUT_URL` holds a non-empty value, THE Logout_Endpoint SHALL include in the JSON body an `idpLogoutUrl` field whose value equals that setting character for character.
6. WHEN the Logout_Endpoint receives two or more successive calls carrying the same session cookie value, THE Logout_Endpoint SHALL answer every call in that sequence with the status required by criterion 3 or criterion 4 for that call, SHALL report `wasAuthenticated` as false on every call after the first, and SHALL leave the request anonymous on every call.
7. WHEN logout has completed, THE Georama_BFF SHALL resolve the prior session key to no server-side session, so a replay of the old cookie at `loginuser` reports logged out.
8. THE Logout_Endpoint SHALL clear the session cookie in every response it returns, expiring it with the same domain and path attributes with which it was issued, so the browser retains no usable session cookie value.
9. THE Logout_Endpoint SHALL complete using georama session state alone, performing no request to the Identity_Provider.
10. WHEN the Logout_Endpoint receives a request carrying no authenticated session, THE Logout_Endpoint SHALL answer with the status required by criterion 3 or criterion 4, SHALL report `wasAuthenticated` as false where it answers with HTTP 200, and SHALL leave the `User`, `SocialAccount`, and `SocialToken` tables unchanged.
11. WHERE the Viewer origin is configured in `CORS_ALLOWED_ORIGINS` with `CORS_ALLOW_CREDENTIALS` true, WHEN the Logout_Endpoint answers an HTTP 200 to a request whose `Origin` header matches that configured origin, THE Logout_Endpoint SHALL return that response with the cross-origin headers that permit the Viewer's credentialed `fetch()` to read its status, so the Viewer observes a 2xx rather than a network error.
12. THE Logout_Endpoint SHALL limit the effect of every request it serves to terminating the georama session, leaving the `User`, `SocialAccount`, and `SocialToken` rows and every georama permission assignment unchanged, so a forged cross-site request at this path, which is served without CSRF token validation, costs the affected user no more than one re-login.

### Requirement 6: Session cookie and deployment topology

**User Story:** As a georama operator, I want the session cookie configured for my deployment topology, so that the Viewer's credentialed requests actually carry it and page JavaScript cannot read it.

#### Acceptance Criteria

1. THE Georama_BFF SHALL set `SESSION_COOKIE_HTTPONLY` to true, emitting the `HttpOnly` attribute on every session `Set-Cookie` header, so the session cookie value is absent from `document.cookie` in the browser.
2. THE Georama_BFF SHALL default `SESSION_COOKIE_SECURE` to true, emitting the `Secure` attribute on every session `Set-Cookie` header.
3. THE Georama_BFF SHALL default `SESSION_COOKIE_SAMESITE` to `Lax`, which is the weakest setting under which the Identity_Provider's top-level cross-site callback navigation still carries the session cookie holding allauth's OAuth state, and SHALL treat `Strict` as unsupported, which Startup_Check reports as `georama.bff.E003`.
4. WHERE the Viewer is served from the same origin as the Georama_BFF, THE Georama_BFF SHALL operate with a first-party session cookie, requiring no `SESSION_COOKIE_DOMAIN` and no CORS configuration.
5. WHERE the Viewer is served from a different host sharing a registrable domain with the Georama_BFF, THE Georama_BFF SHALL send the session cookie with `SESSION_COOKIE_DOMAIN` set to that shared domain, SHALL require the Viewer origin in `CORS_ALLOWED_ORIGINS` with `CORS_ALLOW_CREDENTIALS` true, and the operator documentation SHALL record that this exposes the session cookie to every subdomain of that shared domain.
6. WHERE the Viewer is served from a different registrable domain, THE Georama_BFF SHALL require `SESSION_COOKIE_SAMESITE` set to `None` together with `SESSION_COOKIE_SECURE` true, and the operator documentation SHALL record this topology as the fallback carrying third-party-cookie risk, whose symptom is that the Viewer's credentialed requests arrive without a session cookie and `loginuser` reports logged out immediately after a successful login.
7. WHILE `SESSION_SAVE_EVERY_REQUEST` is true, THE Georama_BFF SHALL write the session on every request and SHALL set the session expiry to the request time plus `SESSION_COOKIE_AGE` seconds, so the Viewer's 10-minute `loginuser` poll acts as a sliding session renewal at the cost of one session-store write per request.
8. THE Georama_BFF SHALL expire an idle session after `SESSION_COOKIE_AGE` seconds, defaulting to 43200, so that replaying an expired session cookie at `loginuser` reports logged out, and the operator documentation SHALL record aligning that value with the Identity_Provider refresh-token lifetime.
9. WHEN a user authenticates, THE Georama_BFF SHALL issue a session identifier different from any identifier the request carried before authentication, and SHALL resolve that prior identifier to no server-side session afterwards.
10. THE Georama_BFF SHALL set `CSRF_COOKIE_SECURE` and `CSRF_COOKIE_SAMESITE` to the values configured for the corresponding session cookie settings, and SHALL require every origin that submits a server-rendered form to appear in `CSRF_TRUSTED_ORIGINS`.
11. WHERE the Georama_BFF shares a registrable domain with another application, THE Georama_BFF SHALL use a `SESSION_COOKIE_NAME` and a `CSRF_COOKIE_NAME` that no co-hosted application on that domain uses, so neither application overwrites the other's cookie.

### Requirement 7: Token storage and lazy refresh

**User Story:** As a security reviewer, I want every Cognito token held server-side with a single owner for refresh, so that no token reaches the browser and no request path depends on a live token.

#### Acceptance Criteria

1. THE Token_Store SHALL persist Identity_Provider tokens in django-allauth's `SocialToken`, holding the access token in `token`, the refresh token in `token_secret`, and the access-token expiry in `expires_at`, which requires `SOCIALACCOUNT_STORE_TOKENS` set to true because django-allauth 65.x defaults it to false.
2. WHEN a caller requests an access token for a user and the stored `expires_at` lies more than the configured leeway of 30 seconds in the future, THE Token_Store SHALL return the stored access token and SHALL perform no request to the Identity_Provider.
3. IF no `SocialToken` exists for the user and the configured `OIDC_PROVIDER_ID`, or the stored refresh token is empty or holds only whitespace, THEN THE Token_Store SHALL return `None` and SHALL perform no request to the Identity_Provider.
4. IF the refresh exchange fails with a connection error, with a response status outside the 2xx range that is not a refresh-token rejection, or by returning no complete response within the configured 5-second connect timeout or 5-second read timeout, THEN THE Token_Store SHALL emit a log record at WARNING level naming the failure class and containing no token value, SHALL return `None`, and SHALL leave the stored `SocialToken` row, the Django session, the `User` row, and the `SocialAccount` row unchanged.
5. WHEN a refresh exchange succeeds, THE Token_Store SHALL write `token`, SHALL set `expires_at` to the response receipt time plus the returned `expires_in` seconds, and SHALL replace `token_secret` only when the Identity_Provider returned a new refresh token, committing those fields in a single database transaction so no concurrent caller observes an access token paired with a superseded refresh token.
6. THE Token_Store SHALL perform the refresh exchange as a server-to-server POST to the discovery document `token_endpoint` with `client_id` and `client_secret`, a 5-second connect timeout, and a 5-second read timeout.
7. WHEN a caller requests an access token for a user whose stored `expires_at` is NULL, absent, or no more than 30 seconds in the future, THE Token_Store SHALL make at most one refresh attempt for that call and SHALL return either an access token whose `expires_at` lies more than 30 seconds in the future or `None`, leaving retry policy to the caller.
8. THE Georama_BFF SHALL serve `themes.json` and `/webgis/maps` with zero Token_Store calls and zero requests to the Identity_Provider per request, whatever the value of `BFF_LOGINUSER_VERIFY_IDP`, because those paths authorize from the Django session and Django permissions alone.
9. THE Georama_BFF SHALL keep `SocialToken.token` and `SocialToken.token_secret` out of every HTTP response body, every log record, every error page or traceback rendered to a client, and every Django admin page, registering no admin view that renders either field.
10. IF Token_Store returns `None` to a caller that requires an access token for a downstream call, THEN THE Georama_BFF SHALL fail that single request with an error indicating the downstream call could not be completed, SHALL leave the Django session valid, and SHALL continue to serve subsequent requests that need no access token with the user's georama permissions unchanged.
11. WHEN two or more concurrent requests for the same user and the configured `OIDC_PROVIDER_ID` each find the stored access token within 30 seconds of expiry, THE Token_Store SHALL serialize those callers on a row-level lock over that `SocialToken` row held for the duration of the exchange, so at most one refresh exchange reaches the Identity_Provider and every waiting caller returns the single refreshed access token or `None`.
12. IF the Identity_Provider rejects the refresh exchange as an invalid, expired, or revoked refresh token, THEN THE Token_Store SHALL clear the stored `token` and `token_secret`, SHALL return `None`, and SHALL leave the Django session, the `User` row, and the `SocialAccount` row intact, so every later call returns `None` without a request to the Identity_Provider.
13. WHERE `BFF_LOGINUSER_VERIFY_IDP` is false, which is its default, THE Georama_BFF SHALL serve `loginuser` with zero Token_Store calls and zero requests to the Identity_Provider per request.

### Requirement 8: CSRF protection re-enablement

**User Story:** As a security reviewer, I want CSRF protection active now that ambient cookies authenticate requests, so that unsafe-method endpoints stop being forgeable.

#### Acceptance Criteria

1. THE Georama_BFF SHALL include Django's `CsrfViewMiddleware` in the active middleware list for every request, whatever the value of `BFF_ENABLED`, so token validation applies to the unsafe methods POST, PUT, PATCH, and DELETE in both modes.
2. WHEN a POST carrying a CSRF token that Django accepts, supplied either as the `csrfmiddlewaretoken` form field or the `X-CSRFToken` header, reaches `/login`, the permission assign endpoint, the permission remove endpoint, the project delete endpoint, or an entity form endpoint, THE Georama_BFF SHALL execute the view and respond with the same status and the same persisted side effects as that request produced before this feature.
3. IF a POST reaches one of those server-rendered endpoints with a CSRF token that is absent, empty, malformed, or not matching the one bound to the session, THEN THE Georama_BFF SHALL respond with HTTP 403 and a body indicating CSRF verification failure, SHALL leave the view unexecuted, and SHALL leave every database row the view would have written unchanged.
4. THE Georama_BFF SHALL exempt exactly two routes from CSRF token validation — the WFS GetFeature POST at `/webgis/maps` and the POST at `/webgis/short/create` — applying `csrf_exempt` to each class's `dispatch` method because the Viewer sends no token and requires zero code changes, and SHALL validate a CSRF token on every other URL pattern that accepts an unsafe method.
5. WHEN an unsafe-method request reaches an exempted route carrying an `Origin` header whose scheme, host, and port equal the Georama_BFF's own origin, or whose host matches an entry of `BFF_ALLOWED_VIEWER_HOSTS` including `.`-prefixed subdomain patterns, THE Origin_Guard SHALL pass the request to the view unaltered.
6. IF an unsafe-method request reaches an exempted route carrying no `Origin` header, or carrying an `Origin` header matching neither the Georama_BFF's own origin nor an entry of `BFF_ALLOWED_VIEWER_HOSTS`, THEN THE Origin_Guard SHALL respond with HTTP 403 without executing the view, deciding on the `Origin` header alone and consulting no `Referer` header.
7. THE Georama_BFF SHALL require a CSRF token on every unsafe-method request to a non-exempted route, including a request whose `Origin` appears in `CSRF_TRUSTED_ORIGINS`, because that setting relaxes Django's Origin and Referer check without substituting for a token.
8. THE Georama_BFF SHALL set `CSRF_COOKIE_SAMESITE` to `Lax` and SHALL leave `CSRF_COOKIE_HTTPONLY` false, so the existing permission-table fetch can read the cookie value and send it as `X-CSRFToken`.
9. THE Georama_BFF SHALL serve the GET endpoints `oidc/login`, `loginuser`, and `logout` without CSRF token validation because `CsrfViewMiddleware` validates tokens only on unsafe methods, and SHALL validate a CSRF token on a POST to `logout`, which the Viewer reaches by GET.

### Requirement 9: Startup configuration checks

**User Story:** As a georama operator, I want misconfiguration reported at startup, so that I do not debug a silent login loop in the browser.

#### Acceptance Criteria

1. THE Startup_Check SHALL be registered with Django's system check framework, so that `manage.py check`, `runserver`, and `migrate` each invoke it exactly once and include its messages in their output.
2. WHERE `BFF_ENABLED` is false, THE Startup_Check SHALL return an empty message list for every combination of `BFF_ALLOWED_VIEWER_HOSTS`, `SESSION_COOKIE_SAMESITE`, `SESSION_COOKIE_SECURE`, `SESSION_COOKIE_DOMAIN`, `CORS_ALLOWED_ORIGINS`, and `OIDC_SERVER_URL` values, because none of the feature's invariants apply.
3. IF `BFF_ENABLED` is true and `BFF_ALLOWED_VIEWER_HOSTS` holds zero entries once whitespace-only entries are discarded, THEN THE Startup_Check SHALL return exactly one `Error` with id `georama.bff.E001`, because every login would otherwise fall back to `LOGIN_REDIRECT_URL`.
4. WHILE `BFF_ENABLED` is true, IF `SESSION_COOKIE_SAMESITE` holds the string `None`, matched case-insensitively after surrounding whitespace is stripped and treated as distinct from Python `None`, which omits the attribute, and `SESSION_COOKIE_SECURE` is false, THEN THE Startup_Check SHALL return exactly one `Error` with id `georama.bff.E002`, because browsers reject that cookie.
5. WHILE `BFF_ENABLED` is true, IF `SESSION_COOKIE_SAMESITE` holds the string `Strict`, matched case-insensitively after surrounding whitespace is stripped, THEN THE Startup_Check SHALL return exactly one `Error` with id `georama.bff.E003`, because the session cookie would be suppressed on the Identity_Provider callback navigation.
6. WHILE `BFF_ENABLED` is true, IF an entry of `BFF_ALLOWED_VIEWER_HOSTS` matches the host component of no entry of `CORS_ALLOWED_ORIGINS` and matches no entry of `ALLOWED_HOSTS`, THEN THE Startup_Check SHALL return one `Warning` with id `georama.bff.W001` naming that entry, because credentialed fetches from that host would be blocked, while an entry georama itself serves is same-origin and needs no CORS entry.
7. WHILE `BFF_ENABLED` is true, IF `SESSION_COOKIE_DOMAIN` holds a value that is neither empty nor whitespace-only and an entry of `BFF_ALLOWED_VIEWER_HOSTS` is not same-site with that value, THEN THE Startup_Check SHALL return one `Warning` with id `georama.bff.W002` naming that entry, because no session cookie would be sent to that host.
8. THE Startup_Check SHALL draw every message id from the `georama.bff.*` namespace, assigning one fixed id per distinct misconfiguration condition and reusing that id across every host that triggers the same condition, so a deployment can silence one condition deliberately through `SILENCED_SYSTEM_CHECKS`.
9. THE Startup_Check SHALL return a message list from settings values alone, without any database query, file read, or network request, and without raising, for every combination of settings values including unset settings, empty strings, Python `None`, non-string allowlist entries, and allowlists of up to 100 entries.
10. THE Startup_Check SHALL name in its messages only hosts drawn from `BFF_ALLOWED_VIEWER_HOSTS` together with the setting names involved, and SHALL produce the same message set, compared as an unordered collection of id and host pairs, whatever the order of that list.
11. WHILE `BFF_ENABLED` is true, IF `OIDC_SERVER_URL` is empty, consists only of whitespace, or does not parse as an absolute URL with an `http` or `https` scheme and a non-empty host, THEN THE Startup_Check SHALL return exactly one `Error` with id `georama.bff.E004`, because the server-side code exchange could not complete and login would fail after the Identity_Provider redirect.
12. THE Startup_Check SHALL treat a `BFF_ALLOWED_VIEWER_HOSTS` entry as matching a host value when the two are equal ignoring case and ignoring any port component, or when the entry begins with `.` and the host value ends with that entry ignoring case, applying that single rule to `ALLOWED_HOSTS` entries, to `CORS_ALLOWED_ORIGINS` host components, and to the same-site comparison against `SESSION_COOKIE_DOMAIN`.

### Requirement 10: Backward compatibility

**User Story:** As a georama operator running existing deployments, I want every current authentication and authorization path to behave exactly as before, so that enabling this feature is additive.

#### Acceptance Criteria

1. THE Georama_BFF SHALL default `BFF_ENABLED` to false, and that flag SHALL gate exactly the Login_Shim route, the Identity_Probe route, the Logout_Endpoint JSON response branch, and Token_Store, so a deployment that does not opt in reaches none of them.
2. WHILE `BFF_ENABLED` holds either value, WHEN a request carries a valid Bearer token and no session cookie, THE Auth_Middleware SHALL resolve the same `request.user` and the endpoint SHALL return the same HTTP status and the same parsed response body as the pre-upgrade georama version returned under identical configuration and database state.
3. WHERE HTTP Basic mode is configured, WHILE `BFF_ENABLED` holds either value, THE Georama_BFF SHALL authenticate requests through it and return the same HTTP status and the same parsed response body as the pre-upgrade georama version returned under identical configuration and database state.
4. WHERE the configured provider is Keycloak, THE Georama_BFF SHALL complete the existing browser login flow and redirect to the same URL as the pre-upgrade georama version under identical configuration and database state.
5. THE Georama_BFF SHALL continue to serve the Django admin login at `/login` with session and password authentication, returning the same HTTP status and the same redirect target as the pre-upgrade georama version under identical configuration and database state.
6. THE Georama_BFF SHALL return the same parsed `themes.json` body for a given user and permission state whether that user authenticated by session cookie or by Bearer token.
7. WHEN a login is initiated anywhere other than `oidc/login`, THE Georama_BFF SHALL redirect after login to `LOGIN_REDIRECT_URL`, and WHILE `BFF_ALLOWED_VIEWER_HOSTS` is empty THE Redirect_Guard override SHALL return exactly the result of `DefaultAccountAdapter.is_safe_url`, so the globally installed `ACCOUNT_ADAPTER` widens nothing an operator has not declared.
8. IF a request arrives already authenticated by the session, THEN THE Auth_Middleware SHALL short-circuit on `request.user.is_authenticated` and SHALL perform no token decoding, no JWKS lookup, no signature verification, and no request to the Identity_Provider.
9. THE Georama_BFF SHALL drive per-user permission filtering in `themes.json` and `/webgis/maps` from Django permissions and `request.user` alone, returning an identical permitted set of themes and maps for a given user and permission state whether that request was authenticated by session cookie, by Bearer token, or by HTTP Basic.
10. THE Georama_BFF SHALL apply outside the `BFF_ENABLED` gate exactly these changes and no others: enabling `CsrfViewMiddleware`, declaring the `SESSION_COOKIE_*` and `CSRF_COOKIE_*` settings, setting `SESSION_SAVE_EVERY_REQUEST`, and installing `ACCOUNT_ADAPTER`.
11. IF an existing deployment serves georama over plain HTTP, THEN `SESSION_COOKIE_SECURE` defaulting to true SHALL cause the browser to withhold the session cookie so admin and password login stop working while Bearer-token and HTTP Basic authentication continue to work, and the operator SHALL restore that deployment by serving georama over HTTPS or by setting `GEORAMA_SESSION_COOKIE_SECURE` to false.
12. THE Deployment_Procedure SHALL document each change listed in criterion 10, the behaviour it alters for a deployment that has not enabled `BFF_ENABLED`, and the operator action required to preserve prior behaviour, including restoring previous session-lifetime values.

### Requirement 11: Configuration surface

**User Story:** As a georama operator, I want BFF settings declared the same way as every other georama setting, so that I configure them from the environment with typed values.

#### Acceptance Criteria

1. THE Georama_BFF SHALL declare `BFF_ENABLED` as a `values.BooleanValue` defaulting to false, `BFF_ALLOWED_VIEWER_HOSTS` as a `values.ListValue` with a single-space separator defaulting to the empty list, `BFF_IDP_LOGOUT_URL` as a `values.Value` defaulting to the empty string, and `BFF_LOGINUSER_VERIFY_IDP` as a `values.BooleanValue` defaulting to false, each a `django-configurations` `values.*` descriptor on the `Base` configuration with `environ_prefix` set to `GEORAMA`, so each setting is overridden by the environment variable whose name is that setting name prefixed with `GEORAMA_`.
2. THE Georama_BFF SHALL interpret each `BFF_ALLOWED_VIEWER_HOSTS` entry as a host of 1 to 253 characters, optionally prefixed with `.` to denote a subdomain pattern and optionally suffixed with a colon and a port in the range 1 to 65535, and SHALL pass the parsed entries to `django.http.request.validate_host` without rewriting or stripping any character other than the separating spaces.
3. THE Georama_BFF SHALL match a `BFF_ALLOWED_VIEWER_HOSTS` entry against the host component of the candidate URL authority after any port is separated by `django.http.request.split_domain_port`, so an entry carrying a port matches only a candidate URL whose authority carries that identical port, and an entry carrying no port matches only a candidate URL whose authority carries no explicit port.
4. THE Georama_BFF SHALL declare `SESSION_COOKIE_SECURE` defaulting to true, `SESSION_COOKIE_SAMESITE` defaulting to `Lax`, `SESSION_COOKIE_DOMAIN` defaulting to unset, `SESSION_COOKIE_AGE` defaulting to 43200 seconds, and `SESSION_SAVE_EVERY_REQUEST` defaulting to true, each as a `values.*` descriptor with `environ_prefix` set to `GEORAMA`, and SHALL introduce no feature-owned session or cookie setting beyond these stock Django setting names.
5. THE Georama_BFF SHALL set `ACCOUNT_ADAPTER` to the dotted path of `GeoramaAccountAdapter`, so that `allauth.account.adapter.get_adapter()` returns an instance of that class, and SHALL set `SOCIALACCOUNT_STORE_TOKENS` to true.
6. WHERE the `Dev` configuration is active, THE Georama_BFF SHALL default `SESSION_COOKIE_SECURE` to false, so that plain-HTTP localhost development works.
7. WHERE the `Prod` configuration is active, IF `SESSION_COOKIE_SECURE` resolves to false, THEN THE Georama_BFF SHALL emit a startup check error that blocks startup and SHALL identify the setting that caused it.
8. THE Georama_BFF SHALL read the Identity_Provider client secret from a `values.*` descriptor supplied by the environment, SHALL carry no secret value as a committed default, and SHALL exclude that value from every HTTP response body, log record, and startup check message.
9. THE Georama_BFF SHALL default `BFF_LOGINUSER_VERIFY_IDP` to false, so the Viewer's periodic poll answers from one indexed session read and at most one group query and issues no Identity_Provider request.
10. IF an entry in the environment value for `BFF_ALLOWED_VIEWER_HOSTS` contains `://`, a `/`, a `?`, or a `#`, THEN THE Georama_BFF SHALL emit a startup check error that blocks startup, SHALL name the offending entry in that message, and SHALL NOT normalise the entry into a bare host.
11. WHERE the `Test` configuration is active, THE Georama_BFF SHALL declare no BFF or session setting of its own, inheriting every declaration named in criteria 1, 4, and 5 from `Base`, and SHALL resolve each of them to a per-test value when Django's settings-override mechanism supplies one.
12. THE Georama_BFF SHALL ship operator documentation listing every environment variable derived from criteria 1, 4, and 8, each with its type, its default, and whether BFF mode requires a non-default value.

### Requirement 12: Identity_Provider specifics for AWS Cognito

**User Story:** As a georama operator adopting Cognito, I want georama to work within Cognito's exact-match callback rule and its missing `end_session_endpoint`, so that login and logout both complete.

#### Acceptance Criteria

1. WHERE the Identity_Provider is AWS Cognito, THE Georama_BFF SHALL send exactly one `redirect_uri` value in every authorization request, equal to `/accounts/oidc/<provider_id>/login/callback/` on the georama origin, carrying no fragment component and no query component, and using the `https` scheme unless its host is `localhost`, so that the Identity_Provider's exact string comparison against the single registered callback URL succeeds.
2. IF the Identity_Provider discovery document omits `end_session_endpoint`, THEN THE Georama_BFF SHALL complete both login and logout with response statuses outside the 5xx range, reading that key through an absent-tolerant lookup in every code path rather than through a provider-specific branch.
3. THE Logout_Endpoint SHALL terminate the georama session as its entire logout contract, performing no request to the Identity_Provider and leaving the Identity_Provider session active, and the operator documentation SHALL record that a subsequent `oidc/login` may therefore complete with no credential prompt through provider SSO.
4. WHERE `BFF_IDP_LOGOUT_URL` holds a non-empty value, THE Logout_Endpoint SHALL return that value unmodified in the `idpLogoutUrl` field of its JSON body, its own session termination being complete whether or not the caller performs a top-level navigation to that value.
5. WHERE the provider `AUTH_PARAMS` contains `prompt=login`, THE Georama_BFF SHALL include that parameter in every authorization request, so that the Identity_Provider prompts for credentials at each login and no existing provider session is reused.
6. IF the Identity_Provider discovery document cannot be fetched within the configured timeout of 5 seconds at process start, or the fetch returns a status outside the 2xx range, THEN THE Auth_Middleware SHALL complete its initialisation without raising, logging the failure at ERROR level and leaving the Georama_BFF serving anonymous and Bearer-authenticated traffic.
7. IF the authorization request, the token exchange, or ID token validation fails, THEN THE Georama_BFF SHALL respond with an error page served from the georama origin indicating that authentication failed, leaving the request unauthenticated, creating no `User` row and no `SocialAccount` row, and leaving a subsequent request to `oidc/login` able to succeed.
8. WHEN a request reaches `oidc/login` while no discovery document has been loaded, THE Georama_BFF SHALL attempt exactly one discovery fetch bounded by the same 5-second timeout before determining that request's outcome.
9. THE Georama_BFF SHALL accept an ID token only when its signature verifies against a key published at the discovery document `jwks_uri`, its signing algorithm is one the discovery document lists as supported and is not the `none` algorithm, its `iss` claim equals the discovery document issuer, and its `aud` claim equals `OIDC_CLIENT_ID`.
10. IF the Identity_Provider ID token omits the `preferred_username` claim, THEN THE Georama_BFF SHALL resolve the account by the `SocialAccount.provider` and `SocialAccount.uid` pair and derive the username from the `sub` claim, creating no additional `User` row for a `sub` already stored.

### Requirement 13: Provider identifier migration

**User Story:** As a georama operator replacing Keycloak with Cognito on an existing deployment, I want the account-linking hazard spelled out, so that I do not silently duplicate every user.

#### Acceptance Criteria

1. THE Georama_BFF SHALL resolve a user's social account by exact, case-sensitive match on the pair of `SocialAccount.provider` equal to `settings.OIDC_PROVIDER_ID` and `SocialAccount.uid` equal to the Identity_Provider `sub` claim, which is the key both Auth_Middleware and django-allauth already use, and SHALL treat a mismatch in either element of the pair as no social account existing for that identity.
2. WHERE `OIDC_PROVIDER_ID` changes on a deployment holding `SocialAccount` rows for the previous value, THE Deployment_Procedure SHALL, before the first login against the new Identity_Provider, run a reversible data migration that re-points `SocialAccount.provider` from the previous value to the new value and re-points `SocialAccount.uid` for every row covered by an operator-supplied mapping from previous-provider `sub` to new-provider `sub`, leaving `uid` unchanged for rows absent from that mapping.
3. IF the data migration is omitted and a returning identity authenticates through Auth_Middleware with a Bearer token, THEN THE Auth_Middleware SHALL find no account for the new provider and uid pair and SHALL enter account creation, which either creates a second `User` that orphans the previous `SocialAccount` row, or fails the request with a server error when the `preferred_username` claim collides with an existing `User.username` under its uniqueness constraint, because account creation assigns that claim verbatim and performs no email-based matching.
4. WHERE `OIDC_PROVIDER_ID` changes on a deployment holding `SocialAccount` rows for the previous value, THE Deployment_Procedure SHALL report the count of `SocialAccount` rows carrying the previous provider value both before and after the migration, SHALL require that count to be non-zero before the migration and zero after it, SHALL produce that report through a database-touching management step rather than through Startup_Check because system checks perform no database access, and SHALL record in the operator documentation that skipping the migration produces the outcomes in criteria 3 and 5.
5. WHERE `SOCIALACCOUNT_EMAIL_AUTHENTICATION` and `SOCIALACCOUNT_EMAIL_AUTHENTICATION_AUTO_CONNECT` are both true, as they already are in georama's settings, WHEN a browser login completes against the new Identity_Provider with no matching provider and uid pair and carries an email address that the Identity_Provider reports as verified and that matches a verified `EmailAddress` row of an existing `User`, THE Georama_BFF SHALL attach the new `SocialAccount` row to that existing `User` instead of creating a second one, and SHALL create a second `User` whenever no such verified `EmailAddress` row exists, which is the case for every `User` that Auth_Middleware created because its account creation writes no `EmailAddress` row.
6. WHERE the migration re-points `SocialAccount` rows to the new provider value, THE Deployment_Procedure SHALL delete the `SocialToken` rows belonging to those accounts, because those tokens were issued by the previous Identity_Provider and are not redeemable at the new one, so that Token_Store returns `None` on first use and the user re-authenticates rather than Token_Store spending a refresh attempt on a stale refresh token.

### Requirement 14: Dependency and test-tooling constraint

**User Story:** As a maintainer, I want this feature to add no dependency, so that the risky parts rest on packages georama already installs and reviews.

#### Acceptance Criteria

1. THE Georama_BFF SHALL implement every component of this feature using only distributions georama already installs — `django==5.2.16`, `django-allauth[socialaccount]==65.16.1`, `django-cors-headers==4.9.0`, `requests`, and `pyjwt` — adding no entry to the declared runtime dependencies, and SHALL leave `django-allauth[socialaccount]` pinned to the exact version `65.16.1` rather than a range, a minimum, or a wildcard.
2. THE Test_Suite SHALL cover this feature using only the already-declared `pytest` and `pytest-django`, adding no entry to the declared test dependencies and in particular no property-based testing library, and SHALL leave the configured coverage threshold `fail_under` no lower than its current value of 13 while reaching at least 90 percent branch coverage of the modules this feature adds and 100 percent branch coverage of Redirect_Guard.
3. THE Test_Suite SHALL contain an upgrade-guard test that resolves the django-allauth provider login URL through Django `reverse()` with the configured `OIDC_PROVIDER_ID` as the `provider_id` argument and asserts the resolved path, so that a URL-name or signature change in a future django-allauth version fails in the suite rather than in production.
4. THE Test_Suite SHALL exercise Redirect_Guard against a parametrised corpus holding at least two cases for each of the bypass classes scheme confusion, protocol-relative URLs, authorities embedding user information, host-suffix confusion, host-prefix confusion, control characters in the authority, host case variation, port variation, and punycode or IDNA host forms, SHALL run that whole corpus once with `DEBUG` true and once with `DEBUG` false, and SHALL assert one expected boolean result per case and `DEBUG` value, reporting each case as an individual pass or fail.
5. THE Test_Suite SHALL contain an upgrade-guard test asserting that, with `SOCIALACCOUNT_STORE_TOKENS` true, a completed login persists a `SocialToken` row carrying the access token, the refresh token, and a non-null `expires_at`, and that with the setting false no such row is persisted, so that a change to the django-allauth default or to its token-storage behaviour fails in the suite.
6. THE Georama_BFF SHALL import no module whose path lies under `allauth.account.internal` or `allauth.socialaccount.internal`, SHALL reach the OAuth-state storage that carries the return URL only through the public `Provider.redirect` and `allauth.account.utils.get_next_redirect_url` entry points, and THE Test_Suite SHALL assert the return-URL round trip from the final redirect `Location` alone, inspecting no state-store contents.
7. THE Test_Suite SHALL exercise the Identity_Provider round trip against local test doubles for the discovery, authorization, and token endpoints, performing no outbound network request in the default `pytest` selection, and SHALL carry the explicit-only `network_bound` marker on any test that requires a live Identity_Provider.
