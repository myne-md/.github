# PR review kit

The Myne pull-request reviewer, shared by every public `myne-md` repository.

- `prompt.md` — the review brief: eleven dimensions, severity and verdict rules.
- `scripts/build-context.mjs` — reads the `pull_request` event, extracts the
  prior blocking signature, writes `packet.json`.
- `scripts/classify.mjs` — runs `opencode run` headless (read-only permissions)
  and writes `verdict.json`.
- `scripts/apply-verdict.mjs` — renders the verdict into one PR comment, posts
  inline Critical comments and cleans up the previous ones.
- `caller-workflow.yml` — copy this into a repository's
  `.github/workflows/review.yml`.

## How a caller runs it

```yaml
name: pr-review
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
permissions:
  contents: read
jobs:
  review:
    uses: myne-md/.github/.github/workflows/pr-review.yml@main
    secrets:
      OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }}
      MYNE_BOT_PAT: ${{ secrets.MYNE_BOT_PAT }}
```

Both secrets must exist in the calling repository. `OPENCODE_API_KEY` is an
opencode Go key; `MYNE_BOT_PAT` is a `repo`-scoped token for the `myne-bot`
account, used for every read and write against the PR.

Fork PRs are skipped by design: GitHub withholds secrets from them, and
`pull_request_target` would checkout fork code with the secrets present.

## What the reviewer never does

It does not execute repository code, does not write to the tree and has no
network access. Build and test status is left to CI. The workflow asserts the
tree is clean after the review; a dirty tree fails the job.
