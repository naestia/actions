# actions

The **central actions collection** that [Cent](https://github.com/) dispatches.
This repo is registered in Cent as an `actions` repo (not a deployable app), so a
push here never triggers a build — its workflows are **dispatch targets** that
Cent triggers to build/deploy *other* repos.

## How it works

```
push to an app repo (e.g. naestia/mira)
   └─▶ GitHub App emits a push webhook to Cent
          └─▶ Cent decides what to do (its push policy)
                 └─▶ workflow_dispatch  →  this repo's build.yml
                        • mints a GitHub App token scoped to the target repo
                        • checks out the target repo @ the pushed commit
                        • builds / deploys it
                        • POSTs status back to Cent (Deployments + Services update)
```

- App repos need **no workflow files** — the trigger is the native push webhook →
  Cent → dispatch. Cent is the control point.
- These workflows reach into other repos via a short-lived **GitHub App
  installation token** scoped to the target. "Repos it has access to" = repos the
  App is installed on (with `contents: read`).
- Cent picks the driver + inputs; the workflow just does the work and reports back.

## Workflows

- **`build.yml`** — the reusable `workflow_dispatch` build/deploy. Inputs (sent by
  Cent): `target_repo`, `target_ref`, `target_sha`, `cent_deployment_id`,
  `cent_callback_url`.

Add more (e.g. `deploy.yml`, `migrate.yml`) using the same `workflow_dispatch` +
inputs shape; each becomes an action Cent can dispatch.

## Setup

Repo secrets (Settings → Secrets and variables → Actions):

| Secret | What |
| --- | --- |
| `APP_ID` | The GitHub App id (the same App Cent authenticates as) |
| `APP_PRIVATE_KEY` | The App's private key (PEM) — mints the token to check out targets |
| `CENT_API_TOKEN` | Must equal Cent's `CENT_API_TOKEN` (authenticates the status callback) |

Prerequisites:

- The **GitHub App is installed** on this repo *and* on every app repo it builds
  (with `contents: read`; `actions: write` on this repo so Cent can dispatch).
- **Cent is reachable at the `cent_callback_url`** it sends (public HTTPS).
- This repo is registered in Cent's **Pipelines** tab with "This is a collection of
  actions" checked.
