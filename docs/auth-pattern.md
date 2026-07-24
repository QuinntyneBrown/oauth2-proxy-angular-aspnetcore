# Auth Pattern: oauth2-proxy + Okta with an Angular SPA and ASP.NET Core API

This document describes an authentication pattern in which an Angular single-page
application and an ASP.NET Core API sit behind [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/),
with **Okta** as the OIDC identity provider.

The defining properties of the pattern:

- **The browser holds only a session cookie.** No access token, ID token, or refresh
  token is ever stored in — or visible to — JavaScript.
- **Proxy Pass-Through ("Cookie → Token").** On every proxied request, oauth2-proxy
  exchanges its session cookie for the Okta ID token and forwards it to the upstream
  API as a standard `Authorization: Bearer` header.
- **The SPA gets the current userId without a backend API.** The Angular app fetches a
  static `.txt` file through the proxy and reads identity response headers that
  oauth2-proxy attaches. No .NET endpoint is required for this.

## Actors

| Actor | Role |
|---|---|
| Browser (Angular SPA) | Sends the session cookie on every request; never handles tokens |
| oauth2-proxy | Terminates authentication; owns the cookie ↔ token exchange |
| Okta | OIDC provider; runs the login flow and issues the ID token with the user's claims |
| ASP.NET Core API | Upstream; validates the Bearer token it receives from the proxy |
| Static assets | The built Angular app, including a static `whoami.txt` used for identity discovery |

## Architecture

```mermaid
flowchart LR
    B[Browser<br/>Angular SPA] -- "session cookie only" --> P[oauth2-proxy]
    P -- "static assets" --> S[Angular static files<br/>incl. /assets/whoami.txt]
    P -- "Authorization: Bearer &lt;id-token&gt;" --> A[ASP.NET Core API]
    P <-. "OIDC authorization-code flow" .-> O[(Okta)]
```

Everything the browser talks to goes through oauth2-proxy. The proxy serves (or
forwards to) the Angular static assets and proxies API calls to the ASP.NET Core
backend. Okta is only involved during login and token refresh — it never sees the
day-to-day traffic.

## Login and cookie flow

An unauthenticated request to any proxied path triggers the standard OIDC
authorization-code flow. Once it completes, oauth2-proxy stores the session (including
the Okta ID token) server-side or encrypted in the cookie, and the browser carries only
the `_oauth2_proxy` cookie from then on.

```mermaid
sequenceDiagram
    participant B as Browser
    participant P as oauth2-proxy
    participant O as Okta

    B->>P: GET / (no session cookie)
    P->>B: 302 redirect to Okta /authorize
    B->>O: Login (credentials, MFA, ...)
    O->>B: 302 redirect to /oauth2/callback?code=...
    B->>P: GET /oauth2/callback?code=...
    P->>O: Exchange code for tokens
    O->>P: ID token (+ refresh token)
    P->>B: Set-Cookie: _oauth2_proxy=... (HttpOnly, Secure)
    Note over B,P: All subsequent requests carry only the cookie
```

The cookie is `HttpOnly`, so it is invisible to the Angular application. From the SPA's
point of view, authentication simply "already happened" by the time it renders.

## Proxy Pass-Through: Cookie → Token

The API expects a normal JWT Bearer token, but the browser only has a cookie. The
bridge is oauth2-proxy's authorization-header configuration:

```
--pass-authorization-header=true
--set-authorization-header=true
```

- **`--pass-authorization-header=true`** — for every authenticated proxied request,
  oauth2-proxy looks up the session behind the cookie and injects
  `Authorization: Bearer <okta-id-token>` into the request it forwards to the upstream.
  This is the "Cookie → Token" exchange: the browser sends a cookie, the API receives a
  token.
- **`--set-authorization-header=true`** — additionally sets the `Authorization` header
  on the *response*. This matters in `auth_request`-style deployments (e.g. behind
  nginx), where the fronting server reads the header from oauth2-proxy's response and
  attaches it to the upstream request itself.

On the .NET side, nothing about this pattern is custom: the API validates the incoming
token as an ordinary JWT with `AddJwtBearer`, using the Okta authority
(`https://{okta-domain}/oauth2/default`) so signing keys and issuer validation come
from Okta's OIDC discovery document. The API does not need to know oauth2-proxy exists.

## Getting the userId in Angular — the static-file trick

The SPA often needs to know *who* is logged in (to show a name, key client-side state
by user, etc.). Because the cookie is opaque to JavaScript, the app can't decode
anything locally — and this pattern deliberately avoids adding a .NET endpoint for it.

Instead, one more oauth2-proxy flag is enabled:

```
--set-xauthrequest=true
```

With this flag, oauth2-proxy attaches identity **response headers** to the responses it
proxies:

| Response header | Populated from |
|---|---|
| `X-Auth-Request-User` | The user identifier from the session (by default the ID token's `sub`; with `--prefer-email-to-user=true`, the email) |
| `X-Auth-Request-Email` | The `email` claim |
| `X-Auth-Request-Preferred-Username` | The `preferred_username` claim |

Okta supplies these values as claims in the ID token during login; oauth2-proxy stores
them in the session and surfaces them as headers. The trick is that the headers appear
on *any* proxied response — so the cheapest possible request works: a static text file.

1. Ship an empty (or one-line) file with the Angular build, e.g.
   `src/assets/whoami.txt`.
2. At startup, the Angular app fetches it through the proxy and reads the header
   instead of the body.

```typescript
// identity.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { map, shareReplay } from 'rxjs/operators';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class IdentityService {
  readonly userId$: Observable<string | null>;

  constructor(http: HttpClient) {
    this.userId$ = http
      .get('/assets/whoami.txt', { observe: 'response', responseType: 'text' })
      .pipe(
        map(res => res.headers.get('X-Auth-Request-User')),
        shareReplay(1)
      );
  }
}
```

Because the request is **same-origin** (it goes to the same host the app was served
from, through the proxy), the browser lets JavaScript read all response headers — no
CORS or `Access-Control-Expose-Headers` configuration is needed.

**Caching caveat:** the headers come from the proxy, not the file. If the browser or a
CDN serves `whoami.txt` from cache, the request never reaches oauth2-proxy and the
headers will be missing or stale. Serve the file with `Cache-Control: no-store`, or
append a cache-busting query parameter (`/assets/whoami.txt?t=...`) when fetching it.

## Alternative: `GET /oauth2/userinfo`

oauth2-proxy also exposes a built-in endpoint, `GET /oauth2/userinfo`, which returns
the same identity information as JSON:

```json
{ "user": "00u1abcd...", "email": "jane@example.com", "preferredUsername": "jane" }
```

It uses the same session cookie and also requires no backend code. It is arguably more
conventional (a JSON body instead of headers on a static file) and immune to the
static-file caching caveat. This document standardizes on the static-file + response
header approach as the primary pattern, but `/oauth2/userinfo` is a drop-in alternative
if a JSON payload is preferred.

## Full example configuration

A consolidated oauth2-proxy configuration for this pattern with Okta:

```
oauth2-proxy
  --provider=oidc
  --provider-display-name="Okta"
  --oidc-issuer-url=https://{okta-domain}/oauth2/default
  --client-id={okta-client-id}
  --client-secret={okta-client-secret}
  --redirect-url=https://app.example.com/oauth2/callback
  --email-domain=*

  # Cookie (what the browser holds)
  --cookie-secret={32-byte-random-secret}
  --cookie-secure=true
  --cookie-httponly=true

  # Cookie -> Token pass-through
  --pass-authorization-header=true
  --set-authorization-header=true

  # Identity response headers for the SPA
  --set-xauthrequest=true

  # Upstreams
  --upstream=http://localhost:5000/          # ASP.NET Core API + Angular static files
  --http-address=0.0.0.0:4180
```

In Okta, this corresponds to a standard **Web** application (authorization-code grant)
whose sign-in redirect URI is `https://app.example.com/oauth2/callback`. The `sub`,
`email`, and `preferred_username` claims in the ID token are what populate the
`X-Auth-Request-*` headers above.

## Security notes

- **No tokens in the browser.** The session cookie is `HttpOnly` and `Secure`; XSS in
  the SPA cannot exfiltrate a token, because JavaScript never has one.
- **The headers are self-disclosure only.** `X-Auth-Request-*` headers reveal the
  requesting user's own identity to their own browser — nothing about other users.
- **The API must still validate the JWT.** The proxy is not a trust boundary the API
  should blindly rely on. `AddJwtBearer` validation (issuer, audience, signature,
  expiry against the Okta authority) must remain enabled, and the API should not be
  reachable except through the proxy.
- **Strip inbound identity headers.** If the deployment ever honors `X-Auth-Request-*`
  or `Authorization` from clients directly (bypassing the proxy), a client could forge
  them. Network topology should guarantee that all API traffic flows through
  oauth2-proxy.
