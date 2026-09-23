# claude-pr-review

Claude reviews every pull request from GitHub Actions and posts a formal GitHub
review as `claude[bot]`. This repo holds the reusable workflow and the review
skill that it installs as a Claude Code plugin. Each reviewed repo keeps a
short caller workflow.

## Use it in a repo

1. Install the [Claude GitHub App](https://github.com/apps/claude) on the repo.
2. Add a `CLAUDE_CODE_OAUTH_TOKEN` secret (from `claude setup-token`). An org
   secret works for repos in that org.
3. Add `.github/workflows/pr-review.yml`:

```yaml
name: PR Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

jobs:
  review:
    permissions:
      contents: read
      pull-requests: read
      id-token: write
    uses: PopoverAI/claude-pr-review/.github/workflows/pr-review.yml@main
    # with:
    #   release_branch: production  # if release PRs target a branch of their own
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

Don't add a `concurrency` block with the group `pr-review-<PR number>` to the
caller. The shared workflow already uses that group, and a matching group in
the caller cancels the run.

The review starts with the first PR after this file reaches the default branch.
The PR that adds it gets no review.

## Repo-specific notes

To tell the reviewer something particular to one repo, such as where its
tickets live, commit `.github/pr-review-notes.md` to the default branch. The
reviewer reads it from the PR's base branch, so a PR can't change the
instructions it's reviewed under.

## How it works

See [docs/working/pr-review.md](docs/working/pr-review.md).
