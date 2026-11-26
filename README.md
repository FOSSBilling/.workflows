# FOSSBilling Shared Workflows

Reusable GitHub Actions workflows for the [FOSSBilling](https://fossbilling.org/) organization. Pin each caller to a reviewed commit SHA.

| Workflow | Purpose |
| --- | --- |
| [`cloudflare-workers.yml`](.github/workflows/cloudflare-workers.yml) | Deploys Workers with `cf`, updates PR preview comments, and removes branch previews. |
| [`cloudflare-fork-worker-preview.yml`](.github/workflows/cloudflare-fork-worker-preview.yml) | Deploys fork Worker code after GitHub deployment approval. |
| [`cloudflare-fork-preview-cleanup.yml`](.github/workflows/cloudflare-fork-preview-cleanup.yml) | Removes previews for closed fork PRs on a schedule. |

## Deploy a Worker

Commit `cloudflare.config.ts` at the repository root and pin `cf` in the caller’s dependencies. This example expects a `build` job to upload production build output from `.cloudflare/output/` as `worker-build` with `include-hidden-files: true`.

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

**Static sites.** Use `cloudflare-workers.yml` from a trusted `workflow_run` caller. Verify the successful build matches the current fork PR head, then pass its `artifact-name` and `artifact-run-id`. Set `mode: preview`, `prebuilt: false`, and `preview-name`.

**Workers with server code.** Call `cloudflare-fork-worker-preview.yml` on `workflow_run` and set `build-workflow` to the CI workflow filename.

- Set required reviewers on the `fork-preview` environment and store the Cloudflare API token there. Use test bindings and credentials in the preview configuration and Cloudflare’s Previews Base.
- Commit `.github/fork-preview.json` on the default branch. Set `name` to the existing Worker, `main` to the entry point under `worker/`, and `compatibilityDate`. Add `assets` and `env` as needed.
- Build the exact PR head in `pull_request` CI. Upload `fork-worker-${{ github.run_attempt }}` with compiled JavaScript modules under `worker/` and any static files under `assets/`.

Reserve `pr-<number>` names by excluding them from branch builds and setting `reserve-fork-preview-names: true` on branch cleanup. Call `cloudflare-fork-preview-cleanup.yml` on a schedule with `pull-requests: read` and the same `worker-name` and account. For static fork previews, put cleanup and publishing in the same concurrency group.

## Contributing

Run `actionlint` before opening a PR. If you change an action reference, run `gh actions-lock` and commit the updated [lockfile](.github/workflows/actions.lock).

Questions? Say hello on [Discord](https://fossbilling.org/discord) or the [forum](https://forum.fossbilling.org/).

## License

Released under the [GNU Affero General Public License v3.0](LICENSE).
