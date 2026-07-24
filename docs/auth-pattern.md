# Auth Pattern: oauth2-proxy + Okta with an Angular SPA and ASP.NET Core API

This document describes an authentication pattern in which an Angular single-page
application and an ASP.NET Core API run in **Kubernetes** behind
[oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/), with **Okta** as the OIDC
identity provider. Every building block is stock: a standard Kubernetes Ingress, a
standard oauth2-proxy deployment in reverse-proxy (upstream) mode, and Okta's standard
authorization-code flow — nothing is custom to this implementation.

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
| Kubernetes Ingress | Routes all `app.example.com` traffic to the oauth2-proxy Service |
| oauth2-proxy (Deployment/Service) | Terminates authentication; owns the cookie ↔ token exchange |
| Okta | OIDC provider; runs the login flow and issues the ID token with the user's claims |
| ASP.NET Core API (Deployment/Service) | Upstream; validates the Bearer token it receives from the proxy |
| Angular assets (nginx Deployment/Service) | Serves the built Angular app, including a static `whoami.txt` used for identity discovery |

## Architecture

![Architecture](./architecture.png)

*Source: [`architecture.puml`](./architecture.puml)*

Everything the browser talks to enters through the Ingress, which routes all traffic
for the host to the oauth2-proxy Service. oauth2-proxy runs in its standard
reverse-proxy (upstream) mode, forwarding to the Angular assets Service and the
ASP.NET Core API Service. Okta is only involved during login and token refresh — it
never sees the day-to-day traffic.

## Login and cookie flow

An unauthenticated request to any path triggers the standard OIDC authorization-code
flow: the Ingress forwards the request to oauth2-proxy, which redirects the browser to
Okta to sign in. Once the flow completes, oauth2-proxy stores the session (including
the Okta ID token) encrypted, and the browser carries only the encrypted
`_oauth2_proxy` cookie from then on.

![Login and session-cookie flow](./login-flow.png)

*Source: [`login-flow.puml`](./login-flow.puml)*

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
   `src/assets/whoami.txt`, served by the Angular assets nginx pod.
2. At startup, the Angular app fetches it through the Ingress and oauth2-proxy and
   reads the header instead of the body.

![userId via whoami.txt response headers](./userid-via-whoami-headers.png)

*Source: [`userid-via-whoami-headers.puml`](./userid-via-whoami-headers.puml)*

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

  # Upstreams (in-cluster Service DNS names)
  --upstream=http://api.default.svc.cluster.local/api/            # ASP.NET Core API Service
  --upstream=http://angular-assets.default.svc.cluster.local/     # Angular assets nginx Service
  --http-address=0.0.0.0:4180
```

In Kubernetes, oauth2-proxy runs as an ordinary Deployment with a Service, and a
standard Ingress routes all traffic for the host to that Service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: oauth2-proxy
                port:
                  number: 4180
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
