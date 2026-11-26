# FOSSBilling Shared Workflows

Reusable GitHub Actions workflows for the [FOSSBilling](https://fossbilling.org/) organization. Pin each caller to a reviewed commit SHA.

| Workflow | Purpose |
| --- | --- |
| [`cloudflare-workers.yml`](.github/workflows/cloudflare-workers.yml) | Deploys Workers with `cf`, updates PR preview comments, and removes branch previews. |
| [`cloudflare-fork-preview.yml`](.github/workflows/cloudflare-fork-preview.yml) | Publishes static fork previews and deploys fork Worker code after approval. |
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

Build the exact PR head in `pull_request` CI and upload `fork-preview-${{ github.run_attempt }}`. Call `cloudflare-fork-preview.yml` on completed `workflow_run` events. Set `build-workflow` to the CI workflow filename.

- **Static sites.** Set `preview-type: static` and upload `.cloudflare/output/`. The workflow publishes only the static assets and posts the preview link. Grant the caller `actions: read`, `contents: read`, and `pull-requests: write`.
- **Workers with server code.** Set `preview-type: worker` and grant `actions: read`, `contents: read`, and `pull-requests: read`. Set required reviewers and the Cloudflare API token on the `fork-preview` environment. Configure test bindings and secrets in Previews Base.

  Upload compiled JavaScript modules under `worker/` and any static files under `assets/`. Set `worker-entrypoint` when the entry module differs from `index.js`. The workflow uses the default branch’s Node-loadable `cloudflare.config.ts` and npm lockfile for Worker settings.

Reserve `pr-<number>` names by excluding them from branch builds and setting `reserve-fork-preview-names: true` on branch cleanup. Call `cloudflare-fork-preview-cleanup.yml` on a schedule with `pull-requests: read` and the same `worker-name` and account. For static fork previews, put cleanup and publishing in the same concurrency group.

## Contributing

Run `actionlint` before opening a PR. If you change an action reference, run `gh actions-lock` and commit the updated [lockfile](.github/workflows/actions.lock).

Questions? Say hello on [Discord](https://fossbilling.org/discord) or the [forum](https://forum.fossbilling.org/).

## License

Released under the [GNU Affero General Public License v3.0](LICENSE).
