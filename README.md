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

- **`build.yml`** — the reusable build/push workflow. Computes a version from the
  repo's **`package.json` (`version` field) or a plain `.version` file** (Node and
  non-Node repos both work; package.json wins if both exist)
  (develop → `<default_branch>.minor+1.0-rc.N`, else that version as-is) and
  builds + pushes `ghcr.io/<owner>/<name>:<version>` — but **skips the build if that
  tag already exists** (e.g. a re-trigger of the same commit), still reporting the
  tag (as `details`) + `succeeded` back to Cent. Inputs (sent by Cent):
  `target_repo`, `target_ref`, `target_sha`, `cent_deployment_id`,
  `cent_callback_url`, **`default_branch`** (the app's release/base branch, set per
  repo on the Pipelines tab), and an **optional `context`** (Docker build context,
  default repo root — a rule can override it per repo).

- **`probe-ecs.yml`** — an environment **probe**: Cent dispatches it on "Refresh
  running version"; it reads the running image tag from ECS and reports it back so
  Cent can show built-vs-running drift. Inputs: `target_repo`, `environment`,
  `cent_callback_url`. Copy + adapt for other platforms.

Add more (e.g. `deploy.yml`, `migrate.yml`) using the same `workflow_dispatch` +
inputs shape; each becomes an action Cent can dispatch.

**Every workflow Cent dispatches must follow [`CONTRACT.md`](./CONTRACT.md)** — the
required inputs it must declare and the status callback it must POST back.

## Setup

Repo secrets (Settings → Secrets and variables → Actions):

| Secret | What |
| --- | --- |
| `APP_ID` | The GitHub App id (the same App Cent authenticates as) |
| `APP_PRIVATE_KEY` | The App's private key (PEM) — mints the token to check out targets |
| `CENT_API_TOKEN` | Must equal Cent's `CENT_API_TOKEN` (authenticates the status callback) |
| `GHCR_TOKEN` | A PAT with `write:packages` — pushes the built image to GHCR |

Prerequisites:

- The **GitHub App is installed** on this repo *and* on every app repo it builds,
  with **`contents: read`** (checkout); `actions: write` on this repo so Cent can
  dispatch.
- A **`GHCR_TOKEN`** secret — a PAT with `write:packages` — for the GHCR push.
  Neither App installation tokens nor `GITHUB_TOKEN` can reliably create a package
  in your namespace from a different repo, hence a PAT.
- Each **app repo has a `Dockerfile`** at the build context (root by default; set
  the `context` input / rule input otherwise).
- **Cent is reachable at the `cent_callback_url`** it sends (public HTTPS).
- This repo is registered in Cent's **Pipelines** tab with "This is a collection of
  actions" checked.
