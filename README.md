# FOSSBilling Shared Workflows

Reusable GitHub Actions workflows for the [FOSSBilling](https://fossbilling.org/) organization. Pin each caller to a reviewed commit SHA.

| Workflow | Purpose |
| --- | --- |
| [`cloudflare-workers.yml`](.github/workflows/cloudflare-workers.yml) | Deploys Workers with `cf`, updates PR preview comments, and removes branch previews. |
| [`cloudflare-fork-preview.yml`](.github/workflows/cloudflare-fork-preview.yml) | Deploys fork previews after GitHub approval. |
| [`cloudflare-fork-preview-cleanup.yml`](.github/workflows/cloudflare-fork-preview-cleanup.yml) | Removes previews for closed fork PRs on a schedule. |

## Deploy a Worker

Commit `cloudflare.config.ts` at the repository root and pin `cf` 1.0.0-beta.18 or newer in the caller’s dependencies. This example expects a `build` job to upload production build output from `.cloudflare/output/` as `worker-build` with `include-hidden-files: true`.

```yaml
jobs:
  deploy:
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      actions: read
    uses: FOSSBilling/.workflows/.github/workflows/cloudflare-workers.yml@<commit-sha>
    with:
      mode: production
      environment: production
      artifact-name: worker-build
      artifact-path: .cloudflare/output/
      prebuilt: true
    secrets:
      CLOUDFLARE_ACCOUNT_ID: ${{ vars.CLOUDFLARE_ACCOUNT_ID }}
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

See the workflow files for their inputs and outputs. Wrangler callers can keep their existing workflow SHA.

## Fork previews

Build the exact fork PR head in `pull_request` CI and upload `.cloudflare/output/` as `fork-preview-${{ github.run_attempt }}` with `include-hidden-files: true`. Call `cloudflare-fork-preview.yml` on completed `workflow_run` events with the CI filename as `build-workflow`. Grant the caller `actions: read`, `contents: read`, and `pull-requests: read`.

Set required reviewers on the `fork-preview` environment and pass the existing `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` from the caller. GitHub shows the preview URL on the deployment. The workflow reads deployment settings from the default branch’s Node-loadable `cloudflare.config.ts` and npm lockfile. Configure test bindings and secrets in Cloudflare Previews Base for previews that run Worker code.

Reserve `pr-<number>` names by excluding them from branch builds and setting `reserve-fork-preview-names: true` on branch cleanup. Call `cloudflare-fork-preview-cleanup.yml` on a schedule with `pull-requests: read` and the same `worker-name` and account.

## Contributing

Run `actionlint` with the options in the [lint workflow](.github/workflows/actionlint.yml). If you change an action reference, run `gh actions-lock` and commit the updated [lockfile](.github/workflows/actions.lock).

Questions? Say hello on [Discord](https://fossbilling.org/discord) or the [forum](https://forum.fossbilling.org/).

## License

Released under the [GNU Affero General Public License v3.0](LICENSE).
