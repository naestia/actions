# Cent workflow contract

Any workflow in this repo that Cent dispatches **must** declare a specific set of
`workflow_dispatch` inputs and report its result back. Cent is workflow-agnostic —
it always sends the inputs below. **GitHub rejects a dispatch with
`422 Unexpected inputs provided` if the workflow doesn't declare an input Cent
sends**, so keep the `inputs:` block in sync with what your rules pass.

## Required inputs (Cent always sends these)

Declare all five under `on: workflow_dispatch: inputs:`:

| input | what it is |
| --- | --- |
| `target_repo` | `owner/repo` of the app that pushed — check this out |
| `target_ref` | the branch that was pushed (e.g. `develop`) |
| `target_sha` | exact commit to check out |
| `cent_deployment_id` | Cent's correlation id (handy in `run-name:`) |
| `cent_callback_url` | where to POST the final status (see below) |

## Conditional inputs

- **`default_branch`** — sent whenever the service has a default branch set
  (it always does — defaults to `master`). Declare it (with a `default`) in any
  workflow Cent dispatches.
- **`environment`** — sent whenever the rule has an env label set. Declare it
  (with a `default`) if any rule pointing at this workflow uses an env label.
- **Custom rule inputs** — any extra `key=value` you set on a rule are sent as
  top-level inputs too. **Declare each key you use**, or the dispatch is rejected.

## Report status back (required for Cent to track the run)

At the end of the job, POST the outcome to `cent_callback_url` — otherwise Cent
leaves the deployment stuck in `running`, since the callback is how it learns the
result:

```yaml
      - name: Report status to Cent
        if: always()
        env:
          STATUS: ${{ job.status == 'success' && 'succeeded' || 'failed' }}
        run: |
          curl -fsS -X POST "${{ inputs.cent_callback_url }}" \
            -H "Authorization: Bearer ${{ secrets.CENT_API_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d "{\"status\":\"${STATUS}\",\"version\":\"${VERSION}\",\"external_url\":\"${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"}"
```

Body fields:
- **`status`** (required) — one of `pending`, `running`, `succeeded`, `failed`,
  `rolled_back` (Cent's deployment statuses).
- **`version`** (optional) — the real version/tag you computed (e.g. an image
  tag). Cent only knew the short SHA at dispatch, so this is what shows per
  environment in the Services view. Omit or leave empty to keep the SHA.
- **`external_url`** (optional) — a link to the run.
- **`signal`** (optional) — a name you choose to announce that this run finished
  (e.g. `build-service`). On `succeeded`, Cent arms any **arm rule** of this service
  whose "Arm when" is set to **on signal** with the *same name* — making that
  commit ready for a human to Deploy. It's a pure name match, set independently
  here and on the Cent rule; Cent never interprets what the run did. Omit or leave
  empty for no signal.

## Minimal compliant workflow

```yaml
name: example
run-name: "example ${{ inputs.target_repo }}@${{ inputs.target_sha }} (dep ${{ inputs.cent_deployment_id }})"
on:
  workflow_dispatch:
    inputs:
      target_repo:        { required: true }
      target_ref:         { required: true }
      target_sha:         { required: true }
      cent_deployment_id: { required: true }
      cent_callback_url:  { required: true }
      environment:        { required: false, default: "" }   # if any rule sets an env label
      # declare each custom rule input you use, e.g.:
      # registry:         { required: false, default: "" }
permissions: {}
jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: ${{ inputs.target_repo }}
          ref: ${{ inputs.target_sha }}
          token: ${{ steps.apptoken.outputs.token }}   # see build.yml for the App-token step
      # ... do the work ...
      - name: Report status to Cent
        if: always()
        env: { STATUS: "${{ job.status == 'success' && 'succeeded' || 'failed' }}" }
        run: |
          curl -fsS -X POST "${{ inputs.cent_callback_url }}" \
            -H "Authorization: Bearer ${{ secrets.CENT_API_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d "{\"status\":\"${STATUS}\"}"
```

`build.yml` in this repo is the reference implementation.

## Probe workflows (environment status)

A **probe** reports the version *actually running* in an environment — Cent
dispatches it on "Refresh running version" and compares it to what was built.
Cent sends a smaller input set (no commit/deployment):

| input | value |
| --- | --- |
| `target_repo` | `owner/repo` being probed |
| `environment` | the environment to read (e.g. `develop`) |
| `cent_callback_url` | where to POST the running version |

The probe reads the running version from the real environment (it has the cloud
creds — Cent doesn't) and POSTs it back (bearer `CENT_API_TOKEN`):

```json
{ "environment": "develop", "version": "0.3.0-rc.8", "source": "ecs" }
```

`probe-ecs.yml` is a reference implementation (ECS). Configure a service's probe
(actions repo + workflow) on the Pipelines tab's Edit form.

> Note on custom inputs: because GitHub requires every dispatch input to be
> declared, truly "free-form" rule inputs mean editing this workflow to declare
> each new key. If that becomes annoying, Cent can instead pack custom inputs into
> a single JSON string input (`cent_inputs`) that the workflow parses — ask to
> switch to that.
