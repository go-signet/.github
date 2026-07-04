<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/go-signet/brand/main/logo-dark.svg">
    <img src="https://raw.githubusercontent.com/go-signet/brand/main/logo.svg" alt="Signet — OAuth 2.0 / OIDC Authorization Server" width="420">
  </picture>
</p>

# go-signet

**Signet** is a production-ready OAuth 2.0 / OIDC toolkit for Go — covering the **Authorization Server**, **Go & Python SDKs** for secure token storage and validation, and a **Helm chart** for Kubernetes. MCP-ready out of the box with Resource Indicators (RFC 8707) and AS Metadata (RFC 8414).

> A signet ring stays with its owner — pressing it into wax produces a mark anyone can verify but nobody else can make. The ring is the private key you hold; the impression is the verifiable signature it stamps onto every token.

![License](https://img.shields.io/badge/License-MIT-blue.svg)

---

## Key Features

- **Three OAuth 2.0 grants** — Authorization Code + PKCE, Device Authorization Grant, Client Credentials
- **OIDC support** — Signed `id_token` (OIDC Core 1.0), `/oauth/userinfo`, `/.well-known/openid-configuration`, JWKS, `nonce`, `at_hash`, and `email_verified` claims
- **Flexible JWT signing** — HS256 (symmetric), RS256 (RSA), ES256 (ECDSA P-256); asymmetric keys let resource servers verify via JWKS without shared secrets
- **MCP-ready** — Drop-in OAuth 2.1 authorization server for [Model Context Protocol](https://modelcontextprotocol.io/) servers, with Resource Indicators (RFC 8707), AS Metadata (RFC 8414), and CORS on the well-known group
- **Resource Indicators (RFC 8707)** — Per-request `resource` binds the JWT `aud` to a target resource server; refresh requests enforce subset-narrowing so a granted audience can never be widened
- **Pluggable auth backends** — Local database, HTTP API, GitHub, Gitea, GitLab, Microsoft Entra ID
- **Database support** — SQLite (zero-config) and PostgreSQL
- **Security hardened** — Rate limiting, CSRF protection, session fingerprinting, encrypted cookies, PKCE S256 enforcement
- **Audit logging** — Async batch writes, sensitive data masking, CSV export
- **Observability** — Prometheus metrics, health checks, Swagger/OpenAPI docs
- **User & admin management** — Self-service session and per-app consent revocation; admins create/disable/enable users, reset passwords, and inspect third-party connections
- **Per-client token profiles** — `short` / `standard` / `long` access & refresh TTLs per OAuth client
- **Refresh token rotation** — Fixed (reusable) or rotation (one-time use) modes
- **Dynamic Client Registration (RFC 7591)** & **Token Introspection (RFC 7662)** — Opt-in programmatic registration and RFC-compliant token metadata inspection
- **Optional HTTPS** — Serve TLS directly or terminate at a reverse proxy
- **Single binary** — All templates and static assets embedded via `//go:embed`

---

## Projects

| Repository                                          | Description                                                                                                               |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `signet`                                            | OAuth 2.0 / OIDC Authorization Server — Device Grant + Auth Code + PKCE + Client Credentials, MCP-ready                   |
| [`sdk-go`](https://github.com/go-signet/sdk-go)     | Go SDK — token client, OIDC discovery, CLI flow orchestration, keyring storage, JWKS/online middleware                    |
| [`sdk-python`](https://github.com/go-signet/sdk-python) | Python SDK — sync & async auth, with FastAPI / Flask / Django integrations                                           |
| [`examples`](https://github.com/go-signet/examples) | Multi-language usage examples (Go, Python, Bash) — CLI login, M2M, and API protection                                     |
| `helm-signet`                                       | Helm chart — single-instance (SQLite) or HA (PostgreSQL + Redis) Kubernetes deployment                                    |
| [`brand`](https://github.com/go-signet/brand)       | Official brand assets — logos, marks, favicons, palette, and [interactive guidelines](https://go-signet.github.io/brand/) |

---

## Supported OAuth 2.0 Flows

### 1. Authorization Code Flow + PKCE (RFC 6749 + RFC 7636)

Designed for web apps, SPAs, and mobile apps. PKCE replaces client secrets for public clients, preventing authorization code interception. Issues an `id_token` (OIDC) when the `openid` scope is granted.

```mermaid
sequenceDiagram
    participant App    as App (SPA / Mobile / CLI)
    participant Browser as Browser
    participant Server  as Signet Server

    App->>App: Generate code_verifier + code_challenge (S256)
    App->>Browser: Redirect to /oauth/authorize?code_challenge=...&state=...
    Browser->>Server: GET /oauth/authorize (Login + Consent page)
    Server-->>Browser: Render login / consent UI

    Browser->>Server: POST /oauth/authorize (user approves)
    Server->>Browser: 302 redirect_uri?code=XXXXX&state=...
    Browser->>App: Authorization code delivered to callback

    App->>App: Verify state parameter (CSRF check)
    App->>Server: POST /oauth/token (code + code_verifier + client_id)
    Server->>Server: Verify SHA256(code_verifier) == code_challenge
    Server-->>App: access_token + refresh_token + expires_in [+ id_token]
```

**Security properties:**

- PKCE (RFC 7636) with S256 enforced — `plain` is rejected; prevents authorization code interception attacks
- `state` parameter — CSRF protection on the callback
- No client secret required for public clients (SPA, mobile, CLI)
- Callback server binds to `127.0.0.1` only (CLI mode)

---

### 2. Device Authorization Grant (RFC 8628)

Designed for CLI tools, IoT devices, and headless environments where a browser is not available.

```mermaid
sequenceDiagram
    participant CLI    as CLI / Device
    participant Server as Signet Server
    participant User   as User (Browser on another device)

    CLI->>Server: POST /oauth/device/code (client_id + scope)
    Server-->>CLI: device_code + user_code + verification_uri + expires_in + interval

    Note over CLI: Display to user:<br/>"Visit verification_uri<br/>and enter user_code"

    User->>Server: GET /device (enter user_code)
    Server-->>User: Render code entry page (login required)
    User->>Server: POST /device/verify (user_code)
    Server-->>User: ✅ Authorization approved

    loop Poll every interval seconds (default 5 s)
        CLI->>Server: POST /oauth/token (device_code + grant_type)
        alt still waiting
            Server-->>CLI: error: authorization_pending
        else user approved
            Server-->>CLI: access_token + refresh_token + expires_in
        else slow down
            Server-->>CLI: error: slow_down (increase interval)
        else expired / denied
            Server-->>CLI: error: expired_token / access_denied
        end
    end
```

**Polling behavior:**

- Respects server-specified interval (default 5 s)
- Exponential backoff on `slow_down` response (up to 60 s, per RFC 8628)
- Device codes expire after 30 minutes
- Resource-bound device codes (RFC 8707) always route through an explicit confirmation screen before authorization

---

### 3. Client Credentials Grant (RFC 6749 Section 4.4)

Designed for microservices, daemons, and CI/CD pipelines — machine-to-machine authentication where no user is involved.

```mermaid
sequenceDiagram
    participant Service as Service / Daemon
    participant Server  as Signet Server
    participant API     as Protected API

    Service->>Server: POST /oauth/token (client_id + client_secret + grant_type=client_credentials)
    Server->>Server: Validate client credentials
    Server-->>Service: access_token + expires_in (no refresh_token)

    Service->>API: Request with Bearer access_token
    API->>Server: GET /oauth/tokeninfo (verify token)
    Server-->>API: Token valid + scopes
    API-->>Service: Protected resource
```

**Key characteristics:**

- No user interaction — the client authenticates with its own credentials
- Requires **confidential clients** (client_secret must be securely stored server-side)
- No refresh tokens issued — the client requests a new token when the current one expires
- Scoped access — tokens carry only the scopes assigned to the client (`openid` and `offline_access` are not permitted)

---

## Architecture Overview

```mermaid
sequenceDiagram
    participant Client  as Client (CLI / Web / IoT)
    participant Router  as Gin Router + Middleware
    participant Handler as Handlers
    participant Service as Services
    participant Store   as Store (GORM)
    participant DB      as SQLite / PostgreSQL
    participant Ext     as External Auth (optional)

    Client->>Router: HTTP Request
    Router->>Router: Rate limiting, CSRF, Session check
    Router->>Handler: Route matched
    Handler->>Service: Business logic
    Service->>Store: Data access
    Store->>DB: Query / Write
    DB-->>Store: Result
    Store-->>Service: Data
    opt External auth provider (GitHub / GitLab / Microsoft / Gitea / HTTP API)
        Service->>Ext: Authenticate or validate
        Ext-->>Service: User info / Token
    end
    Service-->>Handler: Result
    Handler-->>Client: HTTP Response
```

### Technology Stack

| Layer         | Technology                                                                  |
| ------------- | --------------------------------------------------------------------------- |
| Language      | Go                                                                          |
| Web Framework | [Gin](https://gin-gonic.com/)                                               |
| Templates     | [templ](https://templ.guide/) (type-safe Go HTML)                           |
| ORM           | [GORM](https://gorm.io/)                                                    |
| Database      | SQLite (default) / PostgreSQL                                               |
| JWT           | [golang-jwt/jwt](https://github.com/golang-jwt/jwt) — HS256 / RS256 / ES256 |
| Sessions      | Encrypted cookies via gin-contrib/sessions                                  |
| API Docs      | Swagger/OpenAPI via [swaggo](https://github.com/swaggo/swag)                |
| Metrics       | Prometheus (memory / Redis / Redis-aside cache)                             |

---

## Use Cases

### CLI Tools

One binary, two flows, zero configuration — mirrors the strategy used by **GitHub CLI**, **Azure CLI**, and **Google Cloud SDK**.

```mermaid
sequenceDiagram
    participant User   as User
    participant CLI    as CLI Tool
    participant Server as Signet Server

    User->>CLI: Launch CLI
    CLI->>CLI: Detect environment
    alt Developer Workstation (browser available)
        CLI->>Server: Auth Code + PKCE Flow
        Server-->>CLI: access_token + refresh_token
    else SSH / Headless (no display)
        CLI->>Server: Device Code Flow
        Server-->>CLI: access_token + refresh_token
    end
    CLI-->>User: Authenticated
```

---

### Microservices & CI/CD

Service-to-service authentication — no user context needed, no browser involved.

```mermaid
sequenceDiagram
    participant CI     as CI/CD Pipeline
    participant Server as Signet Server
    participant Deploy as Deploy Service

    CI->>Server: POST /oauth/token (client_credentials)
    Server-->>CI: access_token
    CI->>Deploy: Trigger deploy with Bearer token
    Deploy->>Server: Verify token
    Server-->>Deploy: Valid (scopes: deploy:trigger)
    Deploy-->>CI: Deployment started
```

---

### MCP & Multi-Resource APIs

A single Signet server fronts multiple resource servers; each token's audience is bound at issuance and cannot be replayed elsewhere.

```mermaid
sequenceDiagram
    participant Client as MCP Client
    participant Server as Signet Server
    participant RS_A   as api-a.corp
    participant RS_B   as api-b.corp

    Client->>Server: GET /.well-known/oauth-authorization-server
    Server-->>Client: AS metadata (RFC 8414)
    Client->>Server: POST /oauth/token (resource=https://api-a.corp)
    Server-->>Client: access_token (aud bound to api-a.corp)
    Client->>RS_A: Bearer token (aud matches) ✅
    Client->>RS_B: Bearer token (aud mismatch) ❌ rejected
```

**Hardening:** Every bearer token carries a `type` claim, so a refresh token can never be accepted as an access token. Resource-bound device codes require explicit user confirmation of the client and target resource.

---

### IoT & Smart Devices

No keyboard, no embedded secret — the user authorizes from any nearby device.

```mermaid
sequenceDiagram
    participant TV    as Smart TV / IoT
    participant Server as Signet Server
    participant Phone as User (Phone / Laptop)

    TV->>Server: POST /oauth/device/code
    Server-->>TV: user_code + verification_uri

    Note over TV: Display short URL + user_code<br/>(or QR code)

    Phone->>Server: Visit URL, enter user_code
    Server-->>Phone: ✅ Approved

    TV->>Server: Poll /oauth/token
    Server-->>TV: access_token + refresh_token
```

---

### Web Applications (Confidential Client)

Server-side backend holds the client secret; it is never exposed to the browser.

```mermaid
sequenceDiagram
    participant User   as User (Browser)
    participant App    as Backend Server
    participant Server as Signet Server

    User->>App: ① Visit app
    App->>User: ② Redirect to /oauth/authorize
    User->>Server: ③ Login + Consent
    Server->>User: ④ 302 redirect_uri?code=XXXXX
    User->>App: ⑤ code delivered to callback
    App->>Server: ⑥ POST /oauth/token (code + client_secret)
    Server-->>App: access_token + refresh_token [+ id_token]
```

---

### Single-Page Apps & Mobile (Public Client + PKCE)

No client secret on device — PKCE binds the token exchange to the original requestor.

```mermaid
sequenceDiagram
    participant App     as SPA / Mobile App
    participant Browser as Browser
    participant Server  as Signet Server

    App->>App: Generate code_verifier + code_challenge (S256)
    App->>Browser: Redirect to /oauth/authorize?code_challenge=...&method=S256
    Browser->>Server: Login + Consent
    Server->>Browser: 302 redirect_uri?code=XXXXX&state=...
    Browser->>App: code delivered to callback
    App->>App: Verify state (CSRF check)
    App->>Server: POST /oauth/token (code + code_verifier)
    Server->>Server: Verify SHA256(code_verifier) == code_challenge
    Server-->>App: access_token + refresh_token [+ id_token]
```

---

## Which Flow Should I Use?

| Scenario                                | Recommended Flow                                   |
| --------------------------------------- | -------------------------------------------------- |
| CLI tools, IoT devices, TV apps         | **Device Code Flow** (RFC 8628)                    |
| Web apps with a server-side backend     | **Authorization Code Flow** (confidential client)  |
| Single-page apps (SPA), mobile apps     | **Authorization Code Flow + PKCE** (public client) |
| Microservices, daemons, CI/CD pipelines | **Client Credentials Grant** (RFC 6749 §4.4)       |
| MCP servers, multi-resource APIs        | **Any grant + `resource` parameter** (RFC 8707)    |

---

## Key Endpoints

| Endpoint                                  | Method     | Purpose                                                          |
| ----------------------------------------- | ---------- | ---------------------------------------------------------------- |
| `/.well-known/openid-configuration`       | GET        | OIDC Discovery metadata                                          |
| `/.well-known/oauth-authorization-server` | GET        | OAuth 2.0 AS Metadata (RFC 8414) — required by MCP clients       |
| `/.well-known/jwks.json`                  | GET        | JWKS public keys for RS256/ES256 verification (RFC 7517)         |
| `/oauth/device/code`                      | POST       | Request device + user codes (accepts optional `resource`)        |
| `/oauth/authorize`                        | GET / POST | Authorization consent page (accepts optional `resource`)         |
| `/oauth/token`                            | POST       | Exchange code / device_code / client_credentials / refresh_token |
| `/oauth/tokeninfo`                        | GET        | Verify token validity                                            |
| `/oauth/userinfo`                         | GET / POST | OIDC UserInfo — profile claims for the token owner               |
| `/oauth/revoke`                           | POST       | Revoke tokens (RFC 7009)                                         |
| `/oauth/register`                         | POST       | Dynamic client registration (RFC 7591, opt-in)                   |
| `/oauth/introspect`                       | POST       | Token introspection (RFC 7662)                                   |
| `/device`                                 | GET        | User code entry page (browser)                                   |
| `/account/sessions`                       | GET        | Manage active sessions                                           |
| `/account/authorizations`                 | GET        | Manage per-app consent grants                                    |
| `/admin/users`                            | GET / POST | Admin: list / create users                                       |
| `/admin/clients/:id/revoke-all`           | POST       | Admin: force re-auth for all users of a client                   |
| `/health`                                 | GET        | Health check with database connectivity test                     |
| `/metrics`                                | GET        | Prometheus metrics (optional auth)                               |
| `/api/swagger/*`                          | GET        | Interactive API documentation                                    |

---

## Brand

The Signet identity — a **signet ring** with an octagonal bezel and a carved "S" — lives in the [`brand`](https://github.com/go-signet/brand) repository, with logos, marks, favicons, the color palette, and usage rules. Browse the interactive guidelines at [go-signet.github.io/brand](https://go-signet.github.io/brand/).

---

## References

- [RFC 6749 — OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 6749 §4.4 — Client Credentials Grant](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4)
- [RFC 7636 — PKCE for OAuth Public Clients](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 7009 — OAuth 2.0 Token Revocation](https://datatracker.ietf.org/doc/html/rfc7009)
- [RFC 7591 — OAuth 2.0 Dynamic Client Registration](https://datatracker.ietf.org/doc/html/rfc7591)
- [RFC 7662 — OAuth 2.0 Token Introspection](https://datatracker.ietf.org/doc/html/rfc7662)
- [RFC 8414 — OAuth 2.0 Authorization Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414)
- [RFC 8628 — OAuth 2.0 Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628)
- [RFC 8707 — Resource Indicators for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc8707)
- [RFC 8725 — JWT Best Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [RFC 9700 — Best Current Practice for OAuth 2.0 Security](https://datatracker.ietf.org/doc/html/rfc9700)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)
