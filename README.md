# sast-action

Reusable GitHub Actions library for the [chat-system](https://github.com/cochecheee/ChatSystem) DevSecOps dashboard.

Drop one block in your repo's workflow and get language-aware SAST + dashboard notification automatically.

## Quick start (inheritor repo)

```yaml
# .github/workflows/security.yml
name: Security
on:
  push:
    branches: [main, develop]
  pull_request:
  workflow_dispatch:

jobs:
  security:
    uses: cochecheee/sast-action/.github/workflows/sast-ci.yml@main
    with:
      language: python    # java | python | node | go
      dashboard_url: ${{ secrets.MCP_GATEWAY_URL }}
    secrets:
      dashboard_token: ${{ secrets.MCP_WEBHOOK_TOKEN }}
      nvd_api_key: ${{ secrets.NVD_API_KEY }}   # Java only
```

Pin to `@v0.2.0` once tagged.

## Tools per language

| Language | Universal | Language-specific |
|---|---|---|
| `java`   | Semgrep, Trivy-FS | SpotBugs, OWASP Dep-Check |
| `python` | Semgrep, Trivy-FS | Bandit, Safety |
| `node`   | Semgrep, Trivy-FS | ESLint-security, npm-audit |
| `go`     | Semgrep, Trivy-FS | gosec |

## Layout

```
sast-action/
├── action.yml                              # legacy notify-only single-action (v0.1.0)
├── actions/
│   ├── notify-dashboard/action.yml         # composite — POST webhook
│   ├── sast-suite/action.yml               # composite — run language tools
│   └── aggregate-sarif/action.yml          # composite — merge SARIF
└── .github/workflows/
    └── sast-ci.yml                         # reusable workflow — wires the three composites
```

You can also call individual composites directly:

```yaml
- uses: cochecheee/sast-action/actions/sast-suite@main
  with:
    language: python
- uses: cochecheee/sast-action/actions/notify-dashboard@main
  with:
    dashboard-url: ${{ secrets.MCP_GATEWAY_URL }}
    dashboard-token: ${{ secrets.MCP_WEBHOOK_TOKEN }}
```

## Versioning

| Tag | Status |
|---|---|
| `@main`   | Active dev |
| `@v0.2.0` | Pending — tag after V2 verification on chat-system |
| `@v0.1.0` | Legacy single-action notify (root `action.yml`) |

## Webhook contract

`notify-dashboard` POSTs to `${dashboard-url}/webhook/pipeline-complete`:

```json
{
  "run_id": 1234567890,
  "run_number": "42",
  "repository": "owner/repo",
  "ref": "refs/heads/main",
  "sha": "abc123...",
  "actor": "user",
  "event": "push",
  "pipeline_status": "passed",
  "timestamp": "2026-05-09T12:34:56Z"
}
```

The chat-system mcp service then pulls SARIF artifacts via the GitHub API and parses them into findings. See `docs/webhook-schema.md` in chat-system for the full spec.

## License

TBD.
