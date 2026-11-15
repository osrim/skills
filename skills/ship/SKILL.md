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

The user's request authorizes everything `stack-implement`, `pr-review`, and `address-review` authorize, plus force-pushing the stack's branches when restacking (step 5). Merging stays with the user.

Start every sub-agent with a fresh context window and give it its PR's worktree, so its work stays in that checkout. Tell each one that commits: before every commit, call the Skill tool with `commit-message`; before `gh pr edit --body`, call it with `pr-description`, passing the PR's base.

## 1. Implement

Call the Skill tool with `stack-implement`, passing the source and base branch. The step is done when its report is posted. Carry its table, each item's worktree path, and the checks each delegate reported through every later step. When the stack stopped, the remaining steps cover only the PRs it opened.

## 2. Settle the decisions

Collect every conservative decision and every undone item from the delegate reports. Ask the user about all of them in one grouped question set: each question names its PR, the option taken, and what the alternative changes. With none, go to step 3.

For each PR with a changed decision or a requested follow-up, dispatch one sub-agent to apply the answers in its worktree, update the PR description's decision record, commit, and push with a plain `git push`. The step is done when every decision is confirmed or applied and pushed.

## 3. Review on another provider

Resolve the reviewer. It must run on a different provider from yours; when it matches yours, ask for another.

For each PR, start one child on the reviewer's provider, model, and effort with the harness's cross-provider delegation. When the harness has none, tell the user, give the `/pr-review <PR URL>` command to run on the reviewer, and wait for them to confirm it ran. The brief:

- The PR URL and its worktree path, which the child only reads.
- Call the Skill tool with `pr-review`, passing the PR, and publish the review.
- Report the review URL, the head SHA reviewed, and the finding count per axis.

Run the children in parallel. The step is done when every PR has a review signed `🤖` by the reviewer on its current head SHA (check with `gh api repos/<owner>/<repo>/pulls/<number>/reviews`), or its child's error is recorded. A PR without that review goes on to step 4 with one round of review fewer.

## 4. Address the review

For each PR, dispatch one sub-agent on your own model, in parallel. The brief:

- The PR URL and its worktree path.
- Call the Skill tool with `address-review`, passing the PR. It waits for review bots still reviewing.
- When a fix needs the user's decision, leave that comment unreplied and report the question with its thread link.
- Report the counts per outcome, the pushed commits, the bots that timed out, and the open questions.

When questions come back, ask the user all of them in one grouped question set, then dispatch one sub-agent per affected PR with the answers, to fix, push, and reply as `address-review` does. The step is done when every comment on every work list has a posted reply.

## 5. Restack

A PR **needs a restack** when its base is a stack branch and `git merge-base --is-ancestor origin/<base> origin/<branch>` fails, or when `gh pr view <pr> --json mergeable` reports `CONFLICTING`.

Fetch `origin` and record the remote head SHA of the base branch and every stack branch. Then, for each PR from the bottom of the stack up, when it needs a restack, in its worktree with the local branch level with `origin/<branch>`:

1. `git rebase --onto origin/<base> <recorded SHA of base> <branch>`. The recorded SHA bounds the rebase to this PR's own commits. Resolve conflicts in favour of the decisions the issues and reviews settled.
2. Run the checks the delegate reported for this PR in step 1.
3. `git push --force-with-lease=<branch>:<recorded SHA of branch>`, then fetch `origin`.

On a conflict you cannot settle from those decisions, run `git rebase --abort`. On a failed check, run `git reset --hard <recorded SHA of branch>`, which the remote still holds. Either way, stop restacking and record the PR, the conflicting files or failing check, and the PRs above it. The step is done when no PR needs a restack, or the restack stopped.

## 6. Get the checks green

Wait for the required checks on every PR's head, using the harness's PR watch when it has one, otherwise checking every minute. For each PR with a failed check, dispatch one sub-agent in its worktree to read the failing log, fix the cause in this PR, run the check locally, and push with a plain `git push`. When a fix lands below other PRs, run step 5 again. The step is done when every required check passes, or each PR has had one fix round; record the checks still failing.

## 7. Report

Give a table of position, PR, self-review findings, reviewer findings, `address-review` outcomes (fix, decline, answer), restack result, and checks. Under it, give the top PR's branch and worktree, where the dev server shows every change in the stack. Then list what waits on the user: requested human reviewers who have not reviewed, bots that timed out, unanswered questions, checks still failing, and a stopped restack with its cause. Bots may review again after the force-pushes; run `address-review` on a PR to answer them.
