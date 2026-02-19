# CI/CD Workflows

Mealie uses a two-layer CI/CD setup: **Opsera** handles webhook-based triggering, and **GitHub Actions** handles execution.

## How it works

```
GitHub PR / Push event
        │
        ├─► Opsera (webhook) ──► workflow_dispatch ──► pull-requests.yml
        │                                                   └── test-backend.yml
        │                                                   └── test-frontend.yml
        │                                                   └── build-package.yml
        │                                                   └── e2e.yml
        │
        └─► codeql.yml (own push trigger — unchanged)
```

**Opsera pipelines:**

| Pipeline | Trigger | Dispatches |
|---|---|---|
| `mealie-pr-validation` | PR opened / updated targeting `main` | `pull-requests.yml` |
| `mealie-main-deploy` | Push / merge to `main` | `nightly.yml` |

## Workflow files

| File | Purpose |
|---|---|
| `pull-requests.yml` | Orchestrator for PR validation — runs tests, build, E2E, scanning |
| `nightly.yml` | Orchestrator for main branch — runs tests, builds and publishes `nightly` Docker image |
| `test-backend.yml` | Reusable — backend test suite |
| `test-frontend.yml` | Reusable — frontend test suite |
| `build-package.yml` | Reusable — Python package build |
| `e2e.yml` | Reusable — end-to-end tests |
| `publish.yml` | Reusable — Docker image publish |
| `partial-trivy-container-scanning.yml` | Reusable — container vulnerability scan |
| `codeql.yml` | GitHub Advanced Security — runs on push to `main` |

## Running workflows manually

`pull-requests.yml` and `nightly.yml` use `workflow_dispatch` as their sole trigger.
They can be run manually from the GitHub Actions tab using the **Run workflow** button.

## Troubleshooting webhooks

If the Opsera pipeline does not trigger automatically on a PR event:

1. **Check webhook delivery** — GitHub repo → Settings → Webhooks → click the Opsera webhook → Recent Deliveries. A green check with a `200` response means GitHub delivered the event successfully.
2. **Check the pipeline is active** — In Opsera, confirm the pipeline status is **Active**, not Draft.
3. **Check the workflow target** — In the Opsera step config, confirm the selected workflow is `pull-requests.yml` (PR pipeline) or `nightly.yml` (deploy pipeline), not another workflow file.
4. **Check branch protection** — GitHub repo → Settings → Branches → `main`. The required status check `Opsera PR Validation` must be listed and the Opsera pipeline must be active for it to report.
