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

## Webhook setup

Each Opsera pipeline generates a unique webhook URL. To register it with GitHub:

1. Opsera pipeline → **Start of Workflow** gear icon → **Webhooks** tab → copy the **Webhook URL**
2. GitHub repo → **Settings → Webhooks → Add webhook**
3. Set **Payload URL** to the Opsera webhook URL, **Content type** to `application/json`, and **Secret** to the value stored in Opsera's Secrets Vault as `MEALIE_WEBHOOK_SECRET`
4. For `mealie-pr-validation` select **Pull requests** events only; for `mealie-main-deploy` select **Pushes** only
5. Click **Add webhook** — GitHub will send a ping; a green check confirms the URL is reachable

> Each time a pipeline is recreated in Opsera a new webhook URL is generated.
> Always delete the old GitHub webhook entry and register the new URL to avoid
> delivering events to stale pipelines.

## Troubleshooting webhooks

If the Opsera pipeline does not trigger automatically on a PR event:

1. **Check webhook delivery** — GitHub repo → Settings → Webhooks → click the Opsera webhook → Recent Deliveries. A green check with a `200` response means GitHub delivered the event successfully.
2. **Check for stale webhooks** — If multiple Opsera pipelines were created during setup, there may be multiple webhook entries in GitHub. Remove any that do not correspond to the current active pipeline.
3. **Check the pipeline is active** — In Opsera, confirm the pipeline status is **Active**, not Draft.
4. **Check the workflow target** — In the Opsera step config, confirm the selected workflow is `pull-requests.yml` (PR pipeline) or `nightly.yml` (deploy pipeline), not another workflow file.
5. **Check token permissions** — The GitHub account in Opsera's Tool Registry must use a PAT with `repo` and `actions:write` scopes for `workflow_dispatch` to work.
6. **Check branch protection** — GitHub repo → Settings → Branches → `main`. The required status check `Opsera PR Validation` must be listed and the Opsera pipeline must be active for it to report.
