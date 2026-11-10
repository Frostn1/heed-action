# Heed GitHub Action

Heed comments on pull requests that match rules learned from your repository's own merged-PR history. Each rule includes evidence: how often matching past pull requests needed a follow-up fix or revert within 14 days compared with the repository's rate, plus linked example pull requests.

## Install (about 5 minutes)

1. Get the Heed CLI. The Heed CLI is not public yet. Request access at https://heed-site.aui-svi.workers.dev to get it.
2. In a local checkout of your repository, run `heed init .` (it reads pull request history through the `gh` CLI, so run `gh auth login` first). It writes `.heed/rules.json` with only the rules that have enough evidence, and prints each one with its counts. Commit that file. Re-run it every month or two to refresh the rules.
3. Add `.github/workflows/heed.yml`:

```yaml
name: Heed
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
permissions:
  contents: read
  pull-requests: write
jobs:
  heed:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Frostn1/heed-action@v1
        # with:
        #   rules-file: .heed/rules.json # default
```

## Inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `rules-file` | `.heed/rules.json` | Path to the committed rules file, relative to the repository root |
| `github-token` | `${{ github.token }}` | Token used to read the pull request and write comments |

## What it does not do

Your code is read only on the GitHub Actions runner and is never sent anywhere else. The action's own code makes no network calls except to the GitHub API (`GITHUB_API_URL`, default `https://api.github.com`) using the workflow token. The `oven-sh/setup-bun` step downloads the Bun runtime before the action runs. Heed does not change code, push commits, or approve or block pull requests.

## License

Free to use under the Heed GitHub Action license in LICENSE. Proprietary; (c) Creste LLC.
