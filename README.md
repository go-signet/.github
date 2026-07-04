# .github

Organization-wide defaults for **[go-signet](https://github.com/go-signet)** — the home of **Signet**, a production-ready OAuth 2.0 / OIDC toolkit for Go.

## What's in here

| Path                | Purpose                                                                                       |
| ------------------- | ---------------------------------------------------------------------------------------------- |
| `profile/README.md` | The organization profile shown at [github.com/go-signet](https://github.com/go-signet)        |

This is a special repository: GitHub renders `profile/README.md` on the organization landing page, and any community health files placed here (issue templates, `CONTRIBUTING.md`, `SECURITY.md`, workflow templates, …) become the defaults for every `go-signet` repository that doesn't define its own.

## About Signet

**Signet** covers the full OAuth 2.0 / OIDC stack:

- `signet` — Authorization Server (Auth Code + PKCE, Device Grant, Client Credentials; OIDC Core; MCP-ready with RFC 8707 / RFC 8414)
- [`sdk-go`](https://github.com/go-signet/sdk-go) / `sdk-python` — SDKs for token acquisition, secure storage, and JWT validation
- `helm-signet` — Helm chart for single-instance (SQLite) or HA (PostgreSQL + Redis) Kubernetes deployment
- `examples` — Multi-language usage examples (Go, Python, Bash)
- [`brand`](https://github.com/go-signet/brand) — Official brand assets and [interactive guidelines](https://go-signet.github.io/brand/)

See the full introduction in the [organization profile](profile/README.md).

## License

MIT
