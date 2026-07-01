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
    uses: cochecheee/sast-action/.github/workflows/sast-ci.yml@master
    with:
      language: python    # java | python | node | go
    secrets:
      # dashboard_url is declared as a SECRET (GitHub forbids secret refs in a
      # caller `with:` block), so it must go here — not under `with:`.
      dashboard_url:   ${{ secrets.MCP_GATEWAY_URL }}
      dashboard_token: ${{ secrets.MCP_WEBHOOK_TOKEN }}
      nvd_api_key:     ${{ secrets.NVD_API_KEY }}   # Java only
```

Pin to `@v0.2.0` once tagged (until then use `@master`).

## Tools per language

| Language | Universal | Language-specific |
|---|---|---|
| `java`   | Semgrep, CodeQL, Trivy-FS | SpotBugs, OWASP Dep-Check |
| `python` | Semgrep, CodeQL, Trivy-FS | Bandit, Safety |
| `node`   | Semgrep, CodeQL, Trivy-FS | ESLint-security, npm-audit |
| `go`     | Semgrep, CodeQL, Trivy-FS | gosec |

Each tool runs in its **own parallel job** (a language-aware `strategy: matrix`);
a `merge` job stitches the per-tool reports back into one `sast-reports-<run>`
bundle. See "Pipeline stages" below.

## Per-project DAST + deploy

DAST (OWASP ZAP) and deploy are opt-in per inheritor repo. The DAST job is
**decoupled** from the reusable CD, so it works with either deploy model.

### Model A — the project deploys itself (recommended)

Your repo deploys however it already does (its own `cd.yml`, Render, etc.); the
reusable just runs DAST against the live URL after SAST — independent of the
gate verdict, so even a gate-failing app still gets scanned.

```yaml
jobs:
  security:
    uses: cochecheee/sast-action/.github/workflows/sast-ci.yml@master
    with:
      language: java
      dast: true
      staging_url: ${{ vars.STAGING_URL }}   # repo Variable (not a secret)
      dast_scan_type: baseline               # baseline (~5m) | full (~30m)
    secrets:
      dashboard_url:   ${{ secrets.MCP_GATEWAY_URL }}
      dashboard_token: ${{ secrets.MCP_WEBHOOK_TOKEN }}
```

`deploy` stays `false` (default). If `STAGING_URL` is unset, the DAST job skips
safely.

### Model B — the reusable builds, deploys, then scans

```yaml
jobs:
  security:
    uses: cochecheee/sast-action/.github/workflows/sast-ci.yml@master
    with:
      language: java
      deploy: true
      image_repo: cochecheee/<repo>
      dast: true
      staging_url: https://<repo>.onrender.com
    secrets:
      dashboard_url:      ${{ secrets.MCP_GATEWAY_URL }}
      dashboard_token:    ${{ secrets.MCP_WEBHOOK_TOKEN }}
      docker_username:    ${{ secrets.DOCKER_USERNAME }}
      docker_password:    ${{ secrets.DOCKER_PASSWORD }}
      render_deploy_hook: ${{ secrets.RENDER_DEPLOY_HOOK }}
```

Here DAST waits for the reusable CD to deploy a fresh image, then scans it. The
security gate blocks CD (and therefore DAST) when it fails.

DAST output is uploaded as artifact `dast-reports-<run>` (ZAP JSON/MD/HTML); the
chat-system poller ingests `dast-reports-*` into findings. Full design notes:
`docs/research-per-project-dast-deploy.md`.

## Pipeline stages

The reusable workflow wires these jobs. The `scan` tools run **in parallel**
(one job per tool); everything else is sequential (each `needs` the one before):

```
setup ─▶ scan ×N (parallel) ─▶ merge ─▶ gate ─▶ cd ─▶ dast
(matrix)  (one job / tool)     (stitch) (block) (deploy) (ZAP)
```

| Job | Runs when | What it does |
|---|---|---|
| `setup` | always | Computes the per-language tool matrix (e.g. python → semgrep, codeql, trivy-fs, bandit, safety) |
| `scan`  | always | **One parallel job per tool** (`fail-fast:false`); each uploads `sast-reports-<run>-<tool>` |
| `merge` | always (`!cancelled()`) | Merges the per-tool artifacts into `sast-reports-<run>`, notifies dashboard |
| `gate`  | `gate_enabled: true` (default) | Downloads the merged artifact, counts critical/high, fails if over threshold → blocks `cd`/`dast` |
| `cd`    | `deploy: true` + gate pass/skip | Builds & pushes Docker image, triggers Render deploy |
| `dast`  | `dast: true` + `staging_url` set | OWASP ZAP baseline/full scan against staging |

## Layout

```
sast-action/
├── actions/
│   ├── sast-suite/action.yml               # composite — run language SAST/SCA tools
│   ├── security-gate/action.yml            # composite — count crit/high, fail over threshold
│   ├── notify-dashboard/action.yml         # composite — POST /webhook/pipeline-complete
│   ├── build-image/action.yml              # composite — Docker build + Trivy image scan + push
│   ├── deploy-staging/action.yml           # composite — trigger Render deploy hook
│   └── run-dast/action.yml                 # composite — OWASP ZAP baseline/full scan
└── .github/workflows/
    └── sast-ci.yml                         # reusable workflow — wires the composites into 4 jobs
```

You can also call individual composites directly:

```yaml
- uses: cochecheee/sast-action/actions/sast-suite@master
  with:
    language: python
- uses: cochecheee/sast-action/actions/notify-dashboard@master
  with:
    dashboard-url: ${{ secrets.MCP_GATEWAY_URL }}
    dashboard-token: ${{ secrets.MCP_WEBHOOK_TOKEN }}
```

## Versioning

| Tag | Status |
|---|---|
| `@master` | Active dev (current) |
| `@v0.2.0` | Pending — tag after V2 verification on chat-system |

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
