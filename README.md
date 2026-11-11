# Heed GitHub Action

Heed comments on pull requests that match rules learned from your repository's own merged-PR history. Each rule includes evidence: how often matching past pull requests needed a follow-up fix or revert compared with the repository's rate, plus linked example pull requests.

## Install (about 5 minutes)

1. Get the Heed CLI. The Heed CLI is not public yet. Request access at https://heed-site.aui-svi.workers.dev to get it.
2. In a local checkout of your repository, run `heed init .` (it reads pull request history through the `gh` CLI, so run `gh auth login` first). It writes `.heed/rules.json` with only the rules that have enough evidence, and prints each one with its counts. Commit that file. Re-run it every month or two to refresh the rules.
3. Add `.github/workflows/heed.yml`:

```yaml
name: Heed
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  issue_comment:
    types: [created]
permissions:
  contents: read
  pull-requests: write
jobs:
  heed:
    if: (github.event_name != 'issue_comment' && github.event.pull_request.draft == false) || (github.event.issue.pull_request && startsWith(github.event.comment.body, '/heed'))
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        if: github.event_name != 'issue_comment'
      - uses: Frostn1/heed-action@v1
        # with:
        #   rules-file: .heed/rules.json # default
```

## Open-source repos (PRs from forks)

Use `pull_request_target` when the repository accepts PRs from forks:

```yaml
name: Heed
on:
  pull_request_target:
    types: [opened, synchronize, reopened, ready_for_review]
  issue_comment:
    types: [created]
permissions:
  contents: read
  pull-requests: write
jobs:
  heed:
    if: (github.event_name != 'issue_comment' && github.event.pull_request.draft == false) || (github.event.issue.pull_request && startsWith(github.event.comment.body, '/heed'))
    runs-on: ubuntu-latest
    steps:
      - uses: Frostn1/heed-action@v1
```

No checkout step is needed. Under `pull_request_target`, Heed reads `.heed/rules.json` from the base branch through the API, so a fork cannot swap in its own rules. This is safe because Heed only reads the PR through the GitHub API and never runs code from the PR.

An `issue_comment` run also reads through the GitHub API: it never checks out or runs PR code.

Do not add steps that check out the PR head (for example, `actions/checkout` with `ref: ${{ github.event.pull_request.head.sha }}`) or run its code in the same job, since that job has a write token. Heed needs only `contents: read` and `pull-requests: write`, which also covers replies to `/heed` commands. Add `checks: write` only if you turn on check output.

Keep the `pull_request` workflow above for private repositories. Under `pull_request`, a fork PR falls back to the step summary; under `pull_request_target`, it comments normally.

## Inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `rules-file` | `.heed/rules.json` | Path to the committed rules file, relative to the repository root |
| `github-token` | `${{ github.token }}` | Token used to read the pull request and write comments |
| `min-rules` | `1` | Minimum number of filtered matches required to report |
| `min-lift` | `0` | Ignore rules below this lift |
| `max-inline` | `5` | Maximum inline notes (hard cap 10) |
| `inline` | `true` | Enable inline notes |
| `check` | `false` | Publish a Heed check run |
| `fail-on-match` | `false` | Fail the job when matches meet the threshold |

## Repository config (.heed/config.json)

Commit optional repository defaults on the base branch:

```json
{
  "minRules": 2,
  "minLift": 1.5,
  "maxInline": 3,
  "inline": true,
  "muted": ["learned.risk.example"]
}
```

Precedence is Action input, then `.heed/config.json` on the base branch, then the default. Config is always read from the PR base SHA, never from the PR branch.

## Muting a rule

An OWNER, MEMBER, or COLLABORATOR can comment `/heed ignore <rule-id>`, or reply `/heed ignore` to an inline Heed note. Mutes are recomputed from authorized comments every run; delete the command comment to unmute. `/heed ignore-repo <rule-id>` replies with instructions for adding the id to `muted` in `.heed/config.json`; Heed never edits files.

A mute applies right away when the workflow listens to `issue_comment` (see the examples above), otherwise on the next push. A reply to an inline note applies on the next push or `/heed` comment. Rule ids are the `id` fields in `.heed/rules.json`. The summary then says, for example, `1 rule(s) muted on this PR by @name`, written so it does not notify that person.

## Checks output (opt-in)

Set `check: true` and grant `checks: write`. A match has a neutral conclusion and never blocks unless `fail-on-match: true`; a 403 from the Checks API is skipped with a warning.

## Audit trail

Every summary comment, check run and step summary ends with a line such as `Rules: .heed/rules.json at 1f3c9a0, version 1, learned from acme/widget, built 2026-11-10, sha256 0846685d4f49, 23 rules.` that names the commit the rules file was read from (the base SHA under `pull_request_target` and `issue_comment`, otherwise the workspace path and its HEAD) and the first 12 hex characters of the SHA-256 of its text, plus a refresh note when the rules are more than 120 days old or have no build date.

## Refreshing rules monthly

Rules are rebuilt by `heed init`, which is part of the licensed Heed CLI, not this Action. The CLI reads PR history with `git` and `gh`, so it is not bundled here; this Action never runs a subprocess. If you have a Heed licence, the workflow below rebuilds `.heed/rules.json` once a month (or on demand) and opens a pull request with the result. It never pushes to the default branch, so the new rules go through review like any other change.

Save it as `.github/workflows/heed-refresh.yml`. It downloads the Linux `heed` binary from the link in your licence (store the link as the `HEED_CLI_URL` secret and its SHA-256 as `HEED_CLI_SHA256`). On a self-hosted runner that already has `heed` on `PATH`, drop the download step.

```yaml
name: Heed rules refresh
on:
  schedule:
    - cron: "0 6 1 * *" # 06:00 UTC on the 1st of each month
  workflow_dispatch:
permissions:
  contents: write
  pull-requests: write
jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install the Heed CLI
        env:
          HEED_CLI_URL: ${{ secrets.HEED_CLI_URL }}
          HEED_CLI_SHA256: ${{ secrets.HEED_CLI_SHA256 }}
        run: |
          curl -fsSL "$HEED_CLI_URL" -o "$RUNNER_TEMP/heed"
          echo "$HEED_CLI_SHA256  $RUNNER_TEMP/heed" | sha256sum -c -
          chmod +x "$RUNNER_TEMP/heed"
          echo "$RUNNER_TEMP" >> "$GITHUB_PATH"
      - name: Rebuild .heed/rules.json
        env:
          GH_TOKEN: ${{ github.token }}
        run: heed init . --repo "${{ github.repository }}"
      - uses: peter-evans/create-pull-request@v7
        with:
          branch: heed/refresh-rules
          add-paths: .heed/rules.json
          commit-message: "chore: refresh Heed rules"
          title: "chore: refresh Heed rules"
          body: Monthly `heed init` run. Review the rule changes before merging.
```

Allow GitHub Actions to create pull requests (Settings -> Actions -> General -> Workflow permissions). A pull request opened with the workflow token does not start other workflows; close and reopen it, or use a GitHub App token, if your required checks must run on it.

## What it does not do

Your code is read only on the GitHub Actions runner and is never sent anywhere else. The action's own code makes no network calls except to the GitHub API (`GITHUB_API_URL`, default `https://api.github.com`) using the workflow token. The `oven-sh/setup-bun` step downloads the Bun runtime before the action runs. Heed does not change code, push commits, or approve pull requests, and it never blocks a merge unless you set `fail-on-match: true`.

## License

Free to use under the Heed GitHub Action license in LICENSE. Proprietary; (c) Creste LLC.
