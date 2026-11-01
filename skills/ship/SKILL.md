---
name: ship
description: Trigger this when the user asks to ship an issue, a milestone, or a project end to end.
argument-hint: "<issue, milestone, project, or issue list> [base branch] [reviewer=<provider>/<model>[:<effort>]]"
---

# Ship

Arguments: `$ARGUMENTS`

- The source and base branch, exactly as `stack-implement` takes them. One issue is a stack of one. A list of open PRs skips to step 3, bottom of the stack first, with a worktree per PR branch and the repo's documented checks in place of the delegates' checks.
- `reviewer=`: the cross-provider reviewer, such as `codex/gpt-6.1-sol:high`. Empty: the reviewer named in the user's instructions (`AGENTS.md`, `CLAUDE.md`). With neither, ask before step 3.

You are the orchestrator. Every PR gets two rounds of review: the private self review `stack-implement` runs with `pr-review draft`, then a public `pr-review` by a model from a **different provider** than yours. `address-review` then fixes or answers every open comment on the PR: from that reviewer, from review bots such as CodeRabbit, and from humans.

The user's request authorizes everything `stack-implement`, `pr-review`, and `address-review` authorize, plus force-pushing the stack branches this run created when restacking (step 6). Merging stays with the user.

Start every sub-agent with a fresh context window and give it its PR's worktree, so its work stays in that checkout.

## 1. Implement

Call the Skill tool with `stack-implement`, passing the source and base branch. The step is done when its report is posted. Carry its table, each item's worktree path, and the checks each delegate reported through every later step. When the stack stopped, the remaining steps cover only the PRs it opened.

## 2. Settle the decisions

Collect every conservative decision and every undone item from the delegate reports. Ask the user about all of them in one grouped question set: each question names its PR, the option taken, and what the alternative changes. With none, go to step 3.

For each PR with a changed decision or a requested follow-up, dispatch one sub-agent to apply the answers in its worktree, update the PR description's decision record, commit with the commit skill when installed, and push with a plain `git push`. The step is done when every decision is confirmed or applied and pushed.

## 3. Review on another provider

Resolve the reviewer. It must run on a different provider from yours; when it matches yours, ask for another.

For each PR, start one child on the reviewer's provider, model, and effort with the harness's cross-provider delegation. When the harness has none, tell the user, give the `/pr-review <PR URL>` command to run on the reviewer, and wait for them to confirm it ran. The brief:

- The PR URL and its worktree path, which the child only reads.
- Call the Skill tool with `pr-review`, passing the PR, and publish the review.
- Report the review URL, the head SHA reviewed, and the finding count per axis.

Run the children in parallel. The step is done when every PR has a review signed `🤖` by the reviewer on its current head SHA (check with `gh api repos/<owner>/<repo>/pulls/<number>/reviews`), or its child's error is recorded. A PR without that review goes on to step 4 with one round of review fewer.

## 4. Wait for review bots

For each PR, list the **expected bots**: `[bot]` accounts with a pending check, an in-progress comment, or a review on the PR, plus bots configured in the repo (such as `.coderabbit.yaml`). A bot is **done** when it has posted a review or comment on the current head SHA, or its check has completed.

Wait until every expected bot is done, or 15 minutes have passed since step 3 finished. Use the harness's PR watch when it has one, and stop watching once done; otherwise check every minute. The step is done when every expected bot is done or the time is up; record each bot that timed out.

## 5. Address the review

For each PR, dispatch one sub-agent on your own model, in parallel. The brief:

- The PR URL and its worktree path.
- Call the Skill tool with `address-review`, passing the PR.
- When a fix needs the user's decision, leave that comment unreplied and report the question with its thread link.
- Report the counts per outcome, the pushed commits, and the open questions.

When questions come back, ask the user all of them in one grouped question set, then dispatch one sub-agent per affected PR with the answers, to fix, push, and reply as `address-review` does. The step is done when every comment on every work list has a posted reply.

## 6. Restack

A PR **needs a restack** when its base is a stack branch and `git merge-base --is-ancestor origin/<base> origin/<branch>` fails.

Fetch `origin` and record the remote head SHA of every stack branch. Then, for each PR from the bottom of the stack up, when it needs a restack, in its worktree with the local branch level with `origin/<branch>`:

1. `git rebase --onto origin/<base> <recorded SHA of base> <branch>`. The recorded SHA bounds the rebase to this PR's own commits.
2. Run the checks the delegate reported for this PR in step 1.
3. `git push --force-with-lease=<branch>:<recorded SHA of branch>`, then fetch `origin`.

On a conflict, run `git rebase --abort`. On a failed check, run `git reset --hard <recorded SHA of branch>`, which the remote still holds. Either way, stop restacking and record the PR, the conflicting files or failing check, and the PRs above it. The step is done when no PR needs a restack, or the restack stopped.

## 7. Report

Give a table of position, PR, self-review findings, reviewer findings, bot reviews seen, `address-review` outcomes (fix, decline, answer), and restack result. Under it, list what waits on the user: requested human reviewers who have not reviewed, bots that timed out, unanswered questions, and a stopped restack with its cause. Bots may review again after the restack's force-pushes; run `address-review` on a PR to answer them.
