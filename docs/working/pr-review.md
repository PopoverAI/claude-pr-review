# Shared PR review

## Vision

Every pull request is reviewed by Claude from CI, as a **formal GitHub review
from `claude[bot]`** — native primitives throughout, no custom state plumbing.
The engine is the built-in `/code-review` skill (canonical, benchmarked by
Anthropic, not ours to evaluate); the judgment around it — how hard to look,
what findings mean, the outcome — lives in the skill
[`ci-review-pr`](../../plugins/pr-review/skills/ci-review-pr/SKILL.md).

This repo is the one place that setup is maintained. It holds the reusable
workflow and a plugin marketplace carrying the skill; each reviewed repo keeps
a caller workflow of about fifteen lines (see the README).

Each fact in the flow has an idiomatic carrier:

| Fact | Carrier |
|---|---|
| Findings, actionable by an agent | Inline review comments (path, line, body) |
| Outcome headline, remaining-finding counts, summary, findings in sections (remaining blocking / remaining non-blocking / closed), what ends the review | The review body |
| Outcome | Formal review state: APPROVE / REQUEST_CHANGES / COMMENT |
| "Was it approved?" | Latest `claude[bot]` review state |
| "Which commit was reviewed?" | The review's `commit_id` |
| Open vs. addressed findings | Review-thread resolution |
| Round continuity | Prior review bodies, unresolved threads with their replies, and PR comments, re-read each round |
| "Did a review happen?" | A review exists for the head SHA, or it doesn't |
| "Reopen review after approval" | `[re-review]` in a commit subject since the approved `commit_id` |
| Waking the work session when the round would otherwise be silent — a body-only finding, or a clean approve that would leave the session hanging | A single issue comment pointing at the review (skill step 5) |

- **Every push gets a round** (`opened`, `synchronize`, `ready_for_review`,
  `reopened`; drafts skipped). A new push cancels an in-progress round
  (Actions `concurrency`).
- **Approval ends the review.** The workflow's first step reads the latest
  `claude[bot]` review and exits before invoking Claude when it is APPROVED —
  so polish after approval is free, which is the point: without this, a worker
  answering "this comment could be clearer" must either buy a full round or
  leave the nit unfixed to keep the blessing. Reopening review is a deliberate
  act, never a side effect of pushing: a human dismisses the review, or the
  pusher puts `[re-review]` in a commit subject. The tag
  buys one fresh round whose outcome then governs as usual; on a PR that isn't
  approved it is inert. Subject line only — a body mentioning the tag (a
  changelog, a revert, prose about this feature) requests nothing. The gate scans commits **since the approved review's
  `commit_id`**, not just the pushed head, so a tagged round cancelled by a
  fast follow-up push doesn't drop the request. The commits endpoint returns
  at most 250 commits; at that cap the gate can't prove a tag absent and buys
  the round rather than silently skipping. The rule lives in both places
  that independently enforce the approval skip: the workflow's decide step and
  the skill's backstop. The assert step exempts a run from the
  review-was-posted check when the PR stands APPROVED — except on a re-review
  round, where that standing approval is the review being reopened.
- **Model is the workflow's call, level is the agent's.** Every round runs
  on Opus 5.5, pinned by full ID (`claude-opus-5-5`). The CLI's `opus` alias
  follows whatever Claude Code version the Action bundles, so it would change
  the reviewer silently. Release PRs get their extra scrutiny from the level
  (`high` on round 1), not a different model. The level is picked per round in
  proportion to risk — a blog-post release legitimately gets `low` at the
  release gate.
- **REQUEST_CHANGES is advisory.** No branch protection; the outcome surfaces
  in the merge box and blocks nothing.
- **Work sessions keep their pickup — but the review body alone won't wake
  them.** The desktop "Auto-fix pull requests" monitor relays inline review
  comments, issue comments, and CI failures into the linked session; it does
  **not** relay a review's summary body, whatever the review's state. Verified
  by reading the linked session's delivered events: an APPROVE whose
  only finding lived in the body woke nothing (round 1), while an APPROVE
  carrying an inline comment relayed the *comment*, not the body (round 3) —
  which also rules out review state as the discriminator. So a round whose open
  findings live only in the body — the norm for an approve-with-nits, and any
  finding on a line the diff doesn't touch — reaches no one on its own. The
  skill bridges this (step 5): whenever a submitted review leaves any open
  finding with no inline comment of its own — including a review that mixes an
  inline finding with a body-only one, where the body-only finding may be
  blocking — it posts one issue comment pointing at the review. It posts on a
  **clean APPROVE** too, findings or not: approve is the terminal state and the
  session should learn the loop closed and the PR is ready to merge rather than
  hang open at the finish line. The comment is skipped only on a non-approving
  round whose every open finding already has an inline comment. The issue comment is a proven relay carrier and, not being a
  review, is invisible to the approval gate and the assert step; the findings
  stay in the review body, their single home.
- **Auth:** a Claude subscription OAuth token as `CLAUDE_CODE_OAUTH_TOKEN`,
  generated with `claude setup-token` by whoever's quota the reviews should
  draw on (the token is tied to that account). Inside the run, `gh` is
  authenticated as `claude[bot]` via the Action's app token — confirmed from
  `claude-code-action` source (`src/entrypoints/run.ts` sets
  `GH_TOKEN`/`GITHUB_TOKEN` to the OIDC-exchanged app token "for downstream
  usage"), which is what makes formal reviews and thread resolution possible.

### Level semantics (upstream cells, v2.1.280)

`/code-review` routes each typed level to a model-specific cell. Opus 5.5 has
no entry of its own and gets the default table: `medium` is a single pass,
and `high` and `xhigh` are distinct, broader cells. So the working ladder is
`low` / `medium` / `high` / `xhigh`, with `high` the standard
release-round-1 choice. (On Opus 5, `medium` and `high` shared one cell.) The
skill embeds this so its level choices mean what it thinks they mean; recheck
it when the Action bumps Claude Code, since the table changes upstream.

### Sharing across repos

- **Public, so both owners can call it.** GitHub shares a private repo's
  workflows only with repos of the same owner, and the reviewed repos span a
  personal account and the PopoverAI org. Nothing in the skill or workflow is
  sensitive; the OAuth token stays in each caller's secrets.
- **Callers track `main`.** A change here reaches every repo on its next PR.
  With six repos and one maintainer, a bad change shows up as a red check
  anywhere and is fixed once. Tracking `main` also keeps the workflow and the
  skill in step, since the plugin marketplace is cloned from the default
  branch on every run. If a staging lane is ever needed, it is a second ref
  (e.g. `@unstable`) that one canary repo tracks.
- **The skill arrives as a plugin**, installed by the action into the
  runner's user config on each run and invoked as `/pr-review:ci-review-pr`.
  That config sits outside the checkout, so the action's restore of `.claude/`
  from the base branch does not touch it, and the skill no longer has to be
  on a repo's base branch before its first review.
- **Per-repo differences go through two escape hatches.** The
  `release_branch` input covers the one mechanical difference seen so far
  (a `production` release branch). A `.github/pr-review-notes.md` in the
  reviewed repo, read from the base branch so a PR cannot rewrite its own
  instructions, covers prose differences such as where tickets live.
- **The trust gate is always on.** It skips PRs from forks by people without
  write access; in a private repo it never fires.
- **The token is passed explicitly.** `secrets: inherit` works only within
  one org, so each caller forwards `CLAUDE_CODE_OAUTH_TOKEN`. PopoverAI repos
  can share an org secret; personal-account repos each hold a copy.

### Deliberately not now

- Cron-scheduled dynamic workflows (bug hunts, style sweeps) on Actions.
- A staging ref for the shared workflow (see above).

## Established facts (smoke test, 2026-08-21)

- The engine's finder fan-out runs inside the Action (8 finders + 6 verifiers
  observed on Fable `high`); findings reach the PR from `claude[bot]`.
  Subagent dispatch appears in the SDK stream as `task_started` events, not
  `Agent` tool-use records.
- `--model claude-fable-5` resolves under OAuth in a runner.
- `claude-opus-5-5` needs Claude Code 2.1.280 or newer; an older CLI rejects
  it with a 400 and the round posts nothing. `claude-code-action` bundles a
  fixed CLI version (`claudeCodeVersion` in `src/entrypoints/run.ts`), so a
  model bump waits until the `v1` tag carries a CLI that knows the model
  (2.1.280 arrived in v1.0.232).
- **Workflow validation:** `claude-code-action` skips (exit `success`) any run
  whose workflow file differs from the default branch. The check runs on
  Anthropic's side; the action only interprets its answer
  (`src/github/token.ts`). It compares the caller's workflow file (observed
  on the two conversion PRs), so a caller stub is unexercised by the PR that adds
  it, and a change to the shared workflow here is proven by the next PR in any
  calling repo.
- **A non-review is silent** — validation skips, quota exhaustion, and
  premature returns all exit `success` with nothing posted. The design absorbs
  this natively: nothing posted means no review and no approval; the only
  path to APPROVED is the agent approving.
- `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` and a wide `--allowedTools` are in
  the workflow as cheap insurance. The evidence for both is confounded (the
  Fable quota was near-exhausted during the smoke runs), so they are marked
  unconfirmed, kept because their cost is zero. In `-p` there is no permission
  prompt: a tool outside the allowlist is refused outright.
- `show_full_output: true` stays on — the run log is the only visibility into
  a review that goes wrong.

## Plan

The pattern was built and proven in one private repo before moving here; its
history is there. Callers are found by searching for
`uses: PopoverAI/claude-pr-review`. The repo-by-repo rollout is tracked
privately, not here.

1. ~~**Build the shared repo**~~ — reusable workflow (`workflow_call`,
   `release_branch` input, trust gate, Opus 5.5), marketplace and plugin
   carrying the skill, README with the caller stub, this doc. Done
   2026-09-23. The plugin was checked by installing it from the GitHub URL
   into a clean config, where it registers as `/pr-review:ci-review-pr`.
2. ~~**Convert one repo per owner first**~~ — a PopoverAI repo and a
   personal-account repo, which covers the untested part: the cross-owner
   call, the plugin install, and the validation check against a caller stub.
   A conversion replaces the repo's `.github/workflows/pr-review.yml` with the
   stub, deletes its `.claude/skills/ci-review-pr/`, and moves repo-specific
   prose into `.github/pr-review-notes.md`. Proven when a PR after the
   conversion gets a formal `claude[bot]` review.
   **State:** the personal-account repo is proven end to end (2026-09-23): a
   smoke-test PR after the merge got the plugin installed inside the action,
   the notes file read from the base branch, and a formal `claude[bot]`
   review on Opus 5.5. The conversion PRs' own runs had already shown the
   cross-owner call resolves and that the validation check compares the
   caller's file. The PopoverAI repo is proven the same way, with
   `release_branch` set (a release PR round has not yet run under it).
3. ~~**Convert the remaining callers**~~, including one that ran the stock
   `code-review` plugin rather than this pattern. All converted and proven
   on smoke-test PRs 2026-09-23. One first failed on that repo's own invalid
   OAuth token (401, after the plugin installed) and passed once the token
   was replaced.
