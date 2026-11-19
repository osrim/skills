---
name: ship
description: Trigger this when the user asks to ship a spec, a ticket, a milestone, or a project end to end.
argument-hint: "<spec, ticket, milestone, project, or ticket list> [base branch] [reviewer=<provider>/<model>[:<effort>]]"
---

# Ship

Arguments: `$ARGUMENTS`

- The source and base branch, exactly as `stack-implement` takes them; it plans one or more stacks from them. A list of open PRs skips to step 3, bottom of the stack first, with a worktree per PR branch and the repo's documented checks in place of the checks step 1 reports.
- `reviewer=`: the cross-provider reviewer, such as `codex/gpt-6.1-sol:high`. Empty: the reviewer named in the user's instructions (`AGENTS.md`, `CLAUDE.md`). With neither, ask before step 3.

You are the orchestrator. Every PR gets two reviews side by side: the private self review `stack-implement` runs with `pr-review draft`, and a public `pr-review` by a model from a **different provider** than yours. `address-review` then fixes or answers every open comment on the PR: from that reviewer, from review bots such as CodeRabbit, and from humans. The reviewer then reviews the final head once more.

The user's request authorizes everything `stack-implement`, `pr-review`, and `address-review` authorize, plus force-pushing the stacks' branches when restacking (step 5). Merging stays with the user.

With one PR in the run, do the per-PR work of steps 2, 4, and 7 yourself, in its worktree, following the brief a sub-agent would get, so the user watches the work and can steer it. Ask the user each question when it comes up, in place of collecting it into a report: ask step 2's questions while the self review's reviewers run, so the review fixes and the answers land in one commit and one round of checks. The step 3 reviewer stays a child, on the reviewer's provider. With more PRs, each step dispatches sub-agents as written.

Start every sub-agent with a fresh context window and give it its PR's worktree, so its work stays in that checkout. Tell each one that commits, and do the same when you commit: before every commit, call the Skill tool with `commit-message`; before `gh pr edit --body`, call it with `pr-description`, passing the PR's base.

After each builder's one full test suite run, every fix round runs only the **covering checks**: the typecheck, lint, and the tests that exercise the changed files. CI runs the full suite on every push, and step 7 fixes what it finds.

## 1. Implement

Call the Skill tool with `stack-implement`, passing the source and base branch. Start step 3 for each PR as soon as you know it is open: when you open it yourself, or when `stack-implement` posts its link. The step is done when `stack-implement`'s report is posted. Carry its tables (each ticket's parent spec among them), each ticket's worktree path, and the checks each ticket's builder reported through every later step. When a stack stopped, the remaining steps cover only the PRs it opened.

## 2. Settle the decisions

Collect every conservative decision and every undone item from the builders' reports. Ask the user about all of them in one grouped question set: each question names its PR, the option taken, and what the alternative changes. With none, the step is done.

For each PR with a changed decision or a requested follow-up, dispatch one sub-agent to apply the answers in its worktree, run the covering checks, update the PR description's decision record, commit, and push with a plain `git push`. The step is done when every decision is confirmed or applied and pushed.

## 3. Review on another provider

Resolve the reviewer. It must run on a different provider from yours; when it matches yours, ask for another.

For each PR, start one child on the reviewer's provider, model, and effort with the harness's cross-provider delegation. The PR's **spec arguments** are `spec=<ticket URL> spec=<parent spec URL>` when its ticket has a parent spec (from step 1's tables; for a list of PRs, from the ticket each PR closes and its parent, found with the tracker's workflow), and none otherwise, so the Spec axis grades against the spec as well as the ticket. When the harness has no cross-provider delegation, tell the user, give the `/pr-review <PR URL>` command with the spec arguments to run on the reviewer, and wait for them to confirm it ran. The brief:

- The PR URL and its worktree path, which the child only reads.
- Call the Skill tool with `pr-review`, passing the PR and its spec arguments, and publish the review.
- Report the review URL, the head SHA reviewed, and the finding count per axis.

Run the children in parallel, beside steps 1 and 2. The step is done when every PR has a review signed `🤖` by the reviewer (check with `gh api repos/<owner>/<repo>/pulls/<number>/reviews`), or its child's error is recorded. A PR without that review goes on to step 4 with one review fewer.

## 4. Address the review

For each PR, dispatch one sub-agent on your own model, in parallel. The brief:

- The PR URL and its worktree path.
- Call the Skill tool with `address-review`, passing the PR. It waits for review bots still reviewing.
- When a fix needs the user's decision, leave that comment unreplied and report the question with its thread link.
- Report the counts per outcome, the pushed commits, the bots that timed out, and the open questions.

When questions come back, ask the user all of them in one grouped question set, then dispatch one sub-agent per affected PR with the answers, to fix, push, and reply as `address-review` does.

Then, for each PR whose head moved since the reviewer's review, run step 3 once more on the new head, and address its new comments the same way. Leave any later review to the user. The step is done when every comment on every work list has a posted reply, and the reviewer has reviewed each PR's latest head or its error is recorded.

## 5. Restack

A PR **needs a restack** when its base is a stack branch and `git merge-base --is-ancestor origin/<base> origin/<branch>` fails, or when `gh pr view <pr> --json mergeable` reports `CONFLICTING`.

Fetch `origin` and record the remote head SHA of the base branch and every stack branch. Then, in each stack, for each PR from the bottom up, when it needs a restack, in its worktree with the local branch level with `origin/<branch>`:

1. `git rebase --onto origin/<base> <recorded SHA of base> <branch>`. The recorded SHA bounds the rebase to this PR's own commits. Resolve conflicts in favour of the decisions the tickets, specs, and reviews settled.
2. Run the checks the builder reported for this PR in step 1.
3. `git push --force-with-lease=<branch>:<recorded SHA of branch>`, then fetch `origin`.

On a conflict you cannot settle from those decisions, run `git rebase --abort`. On a failed check, run `git reset --hard <recorded SHA of branch>`, which the remote still holds. Either way, stop restacking that stack and record the PR, the conflicting files or failing check, and the PRs above it. The step is done when no PR needs a restack, or each stack that still needs one has stopped.

## 6. Report

Report without waiting for CI. For each stack, give a table of position, PR, parent spec, self-review findings, reviewer findings, `address-review` outcomes (fix, decline, answer), restack result, and checks (passed, pending, or failed, from `gh pr checks <pr>`), and the top PR's branch and worktree, where the dev server shows every change in that stack. Then list what waits on the user: requested human reviewers who have not reviewed, bots that timed out, unanswered questions, failed checks, a stopped restack with its cause, and from `stack-implement`'s report the issues left out with their reasons, the specs no planned ticket has as its parent, stopped stacks, and any `gh stack link` command left to run. Bots may review again after the force-pushes; run `address-review` on a PR to answer them.

## 7. Watch the checks

Watch the required checks on every PR's head with the harness's PR watch, ending the turn until it reports; without one, check every minute. For each PR with a failed check, dispatch one sub-agent in its worktree to read the failing log, fix the cause in this PR, run the check locally, and push with a plain `git push`. When a fix lands below other PRs, run step 5 again. The step is done when every required check passes, or each PR has had one fix round. Then stop the watch and post a one-line result per PR, naming the checks still failing.
