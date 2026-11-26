# FOSSBilling Shared Workflows

This repository hosts the reusable GitHub Actions workflows shared across the [FOSSBilling](https://fossbilling.org/) organization. Instead of duplicating CI/CD logic in every project, call the workflows here from other repositories:

```yaml
uses: FOSSBilling/.workflows/.github/workflows/<workflow>.yml@main
```

## Workflows

| File | Type | Purpose |
| --- | --- | --- |
| [`cloudflare-workers.yml`](.github/workflows/cloudflare-workers.yml) | Reusable (`workflow_call`) | Deploys Workers with Wrangler or `cf`, syncs preview comments, and cleans up previews. |
| [`cloudflare-fork-preview-cleanup.yml`](.github/workflows/cloudflare-fork-preview-cleanup.yml) | Reusable (`workflow_call`) | Removes reserved `pr-<number>` previews for closed fork PRs on a schedule. |
| [`cloudflare-fork-worker-preview.yml`](.github/workflows/cloudflare-fork-worker-preview.yml) | Reusable (`workflow_call`) | Deploys an approved fork Worker bundle to a dedicated Cloudflare account. |
| [`actionlint.yml`](.github/workflows/actionlint.yml) | CI for this repo | Lints all workflow files with [actionlint](https://github.com/rhysd/actionlint), run via a digest-pinned Docker image on every push and pull request. |
| [`verify-lockfile.yml`](.github/workflows/verify-lockfile.yml) | CI for this repo | Verifies [`actions.lock`](.github/workflows/actions.lock) with `gh actions-lock --verify` whenever workflow files change. |

## Using the Cloudflare Workers workflow

```yaml
jobs:
  deploy:
    uses: FOSSBilling/.workflows/.github/workflows/cloudflare-workers.yml@main
    with:
      operation: deploy
      mode: production             # preview | production
      environment: production      # GitHub environment holding the secrets
      artifact-name: worker-build  # build artifact uploaded earlier in the run
      cli: wrangler                # wrangler | cf
      config-path: wrangler.jsonc  # required when cli is wrangler
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

Use `deployment-url` for production. For previews, `preview-url` follows the branch and `preview-build-url` points to a specific build.

For a branch push, the workflow checks that the branch still points to the run's commit immediately before deploying. If the branch has moved or been deleted, the deployment fails before contacting Cloudflare.

Use `sync-pr-comment` on `pull_request` for opened, reopened, and synchronized same-repository PRs, or on `workflow_run` for completed push builds. Give the caller `actions: read`, `contents: read`, and `pull-requests: write`. PR comment updates read build metadata and preview URLs via the GitHub API without executing PR code.

See the `inputs:` and `outputs:` blocks at the top of [`cloudflare-workers.yml`](.github/workflows/cloudflare-workers.yml) for the full list, including `pr-comment`, `worker-name`, and `prebuilt`.

Use `delete-preview` on `pull_request` with `types: [closed]` for same-repository PRs, or on `delete` for branches that are still absent when cleanup runs. Cleanup uses event metadata and GitHub API calls without checking out PR code. Set `worker-name` and `environment`, pass both Cloudflare secrets as shown above, and give the calling job `contents: read`, `actions: read`, and the same concurrency group as the branch's preview deployment.

For a named static preview, a trusted `workflow_run` caller authorizes the operation and sets `cli: cf`, `mode: preview`, and `prebuilt: false` (the default). Set `preview-name` to a lowercase slug starting with a letter or digit and containing only letters, digits, and hyphens. Set `artifact-name` to the checked static artifact from the caller’s current run, or supply `artifact-run-id` and the name of the Cloudflare build artifact from the approved run. For an approved run the workflow publishes only `v1/workers/default/assets/`. Grant the calling job `actions: read` and `contents: read`. To maintain the shared preview comment on a fork PR, pass `pr-comment: true` and `fork-pr-number` with the triggering successful PR run as `artifact-run-id`. Grant the caller `pull-requests: write`. Use the returned `preview-build-url` for a commit-specific link.

For static fork previews, call `cloudflare-fork-preview-cleanup.yml` on a schedule. It checks existing `pr-<number>` previews and deletes those for closed fork PRs targeting the default branch. Pass the Worker name and both Cloudflare secrets, grant `pull-requests: read`, and share the fork publishing concurrency group. Reserve `pr-<number>` names in the branch build and set `reserve-fork-preview-names: true` on branch cleanup.

## Fork previews for Workers with server code

Successful fork CI queues the commit for GitHub’s **Review deployments** approval. The deploy job verifies the approval and current PR head before uploading. To retry a failed upload, rerun all jobs in fork CI.

Set up the staging deployment before enabling the caller.

- Provision `<repository>-forks` in a separate Cloudflare account and restrict access to its preview hostname. Set `CLOUDFLARE_ACCOUNT_ID` to the production account ID in GitHub Actions variables.
- Configure [required reviewers](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/review-deployments) on the `fork-preview` environment. Store the staging account ID and token there as `FORK_PREVIEW_CLOUDFLARE_ACCOUNT_ID` and `FORK_PREVIEW_CLOUDFLARE_API_TOKEN`.
- Use only test bindings and credentials safe to expose to fork code, including in Cloudflare’s Previews Base. Set Worker secrets as a JSON object in the optional `FORK_PREVIEW_WORKER_SECRETS_JSON` environment secret. Existing previews can retain their previously deployed secrets.

Commit `.github/fork-preview.json` on the default branch. With `cli: cf`, use `main` for the artifact entry point and `env` for staging bindings. The workflow generates Preview Build Output from this file and deploys it with `cf previews deploy --prebuilt`. An `auth` configuration can start with the following fields and add its staging bindings.

```json
{
  "name": "auth-forks",
  "main": "worker/index.js",
  "compatibilityDate": "2026-11-26",
  "assets": {},
  "env": {
    "ASSETS": { "type": "assets" },
    "ENVIRONMENT": { "type": "text", "value": "fork-preview" }
  }
}
```

For `cli: wrangler`, use Wrangler JSON fields in the same file. Set `main`, `compatibility_date`, optional `assets`, and `previews` with the staging bindings. The workflow uses Wrangler's `no_bundle` and `--ignore-base-config` options.

Build the exact PR head in `pull_request` CI and upload `fork-worker-${{ github.run_attempt }}`. In the ZIP, put compiled `.js` and `.mjs` modules under `worker/` and static files under `assets/`, preserving relative paths. `auth` can use the Vite Build Output bundle and assets. `extensions` can use Astro’s server modules and client assets. The archive limit is 20,000 entries and 256 MiB uncompressed.

Call the reusable workflow from a completed `workflow_run`. Use the CI workflow's display name under `workflows` and its filename under `build-workflow`. Pin the reusable workflow to a reviewed commit and pass the environment secrets by name.

```yaml
name: Fork Worker Previews
on:
  workflow_run:
    workflows: ['CI/CD']
    types: [completed]
permissions:
  actions: read
  contents: read
  pull-requests: read
jobs:
  preview:
    uses: FOSSBilling/.workflows/.github/workflows/cloudflare-fork-worker-preview.yml@<commit-sha>
    with:
      build-workflow: ci-cd.yml
      cli: cf
    secrets:
      FORK_PREVIEW_CLOUDFLARE_ACCOUNT_ID: ${{ secrets.FORK_PREVIEW_CLOUDFLARE_ACCOUNT_ID }}
      FORK_PREVIEW_CLOUDFLARE_API_TOKEN: ${{ secrets.FORK_PREVIEW_CLOUDFLARE_API_TOKEN }}
      FORK_PREVIEW_WORKER_SECRETS_JSON: ${{ secrets.FORK_PREVIEW_WORKER_SECRETS_JSON }}
```

For scheduled cleanup, call `cloudflare-fork-preview-cleanup.yml` with `worker-name: <repository>-forks` and `environment: fork-preview-cleanup`. Store the staging credentials as `CLOUDFLARE_ACCOUNT_ID` and `CLOUDFLARE_API_TOKEN` in that environment and pass both by name. The cleanup environment can run without a deployment review and should target the same staging Worker.

## Dependency pinning and updates

Third-party actions are pinned as strictly as possible:

- Every action referenced by the workflows is locked to a commit SHA in [`actions.lock`](.github/workflows/actions.lock), generated by [`gh-actions-lock`](https://github.com/github/gh-actions-lock). CI rejects any workflow change that doesn't update the lockfile. Regenerate it locally with `gh actions-lock` after editing workflows.
- The actionlint check runs `rhysd/actionlint` by Docker digest (`@sha256:...`), not by tag.
- The `gh-actions-lock` extension itself is installed with `--pin v0.5.1` in `verify-lockfile.yml`.
- [Renovate](.github/renovate.json) opens PRs to bump action versions, the lockfile, and the pinned extension, with automerge schedules configured.

## Contributing

Open a pull request with your change. The `actionlint` and `verify-lockfile` workflows run automatically — make sure both pass before merging. If you add or change a third-party action, run `gh actions-lock` and commit the updated `actions.lock` in the same PR.

Questions? Say hello on [Discord](https://fossbilling.org/discord) or the [forum](https://forum.fossbilling.org/).

## License

Released under the [GNU Affero General Public License v3.0](LICENSE).
