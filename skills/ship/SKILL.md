---
name: ship
description: Trigger this when the user asks to ship a spec, a ticket, a milestone, or a project end to end.
argument-hint: "<spec, ticket, milestone, project, or ticket list> [base branch] [reviewer=<provider>/<model>[:<effort>][,<next>...]]"
---

# Ship

Arguments: `$ARGUMENTS`

- The source and base branch, exactly as `stack-implement` takes them. A list of open PRs skips step 2: write the run notes yourself (the repo's check commands, and each PR's tickets and spec), and use a worktree per PR branch.
- `reviewer=`: the cross-family reviewers, in the order to try them, such as `copilot/gpt-6.2-sol,codex/gpt-6.1-sol:high`. Empty: the reviewer and the fallbacks named in the user's instructions (`AGENTS.md`, `CLAUDE.md`).

You are the orchestrator. `stack-implement` lands each spec's tickets on one integration branch with one PR. Every PR then gets a public `pr-review` by a model from a **different family** than the one that built it (Claude, GPT, Gemini: the provider that serves the model does not count). `address-review` fixes or answers every open comment on the PR: from that reviewer, from review bots such as CodeRabbit, and from humans. The reviewer then verifies the fixes.

The user's request authorizes everything `stack-implement`, `pr-review`, and `address-review` authorize. Merging stays with the user.

With one ticket in the run, do the work of steps 4 and 9 yourself, in the PR's worktree, following the brief a sub-agent would get, so the user watches the work and can steer it, and ask the user each question when it comes up. With more tickets, each step dispatches sub-agents as written, and the run rules below hold.

Start every sub-agent with a fresh context window and give it its PR's worktree, so its work stays in that checkout. Tell each one that commits, and do the same when you commit: before every commit, call the Skill tool with `commit-message`; before `gh pr edit --body`, call it with `pr-description`, passing the PR's base.

## Run rules

- **Wait by ending the turn.** After you start background work or a PR watch, end the turn; its completion brings you back. Every turn you take starts work, acts on a result, or reports one. When the harness cannot run a child in the background, run it in the foreground.
- **Decide and record.** The user is away after step 1: put no question to them. When a choice comes up that the tickets and specs leave open, take the recommended option and append it to `<notes>/decisions.md` as the PR or ticket, the question, the option taken, and what the alternative changes. You alone write that file, from the sub-agents' reports.
- **Park on schema, auth, or public API.** A choice in one of those areas that the tickets and specs leave open goes under `## Needs your answer` in `<notes>/decisions.md` with a recommendation. Its comment stays unreplied or its ticket stays parked, and the rest of the run goes on.
- **Show the decisions.** The step 8 report and the step 9 result each end with `decisions.md` in full, so the list is always in the latest message.
- **Covering checks.** Every fix round runs only the typecheck, the lint, and the tests that exercise the changed files. CI runs the full suite on every push, and step 9 fixes what it finds.
- **Fall back.** When a provider reports a usage limit, start the remaining children on the model fallback the user's instructions name, and check that the reviewer is still a different family from the builder. When a reviewer child fails, retry it once, then move to the next reviewer on the list.

## 1. Preflight

Run each check while the user is still here, and stop with the failing check when one fails:

- **Tracker:** read one issue of the source through the tracker's workflow.
- **GitHub:** `gh auth status` passes.
- **Reviewer:** resolve the reviewer list. Drop each entry of your own model family, then start a child on the first entry with the harness's cross-provider delegation, briefed to reply with its model name. Take the first entry that answers. With no entry left, ask the user for a reviewer. This also proves that sub-agents start.
- **PR watch:** record whether the harness has one.
- **App access:** the checks that need a running app are yours when you can open the dev server's pages signed in, with a browser tool and a session or a documented test login. Otherwise their owner is `manual, user`.

Create the run notes directory outside the repo, in the harness's scratch directory when it has one, and write `preflight.md` with the reviewer, the PR watch, and the owner of the app checks. The step is done when every check has passed.

## 2. Implement

Call the Skill tool with `stack-implement`, passing the source, the base branch, `notes=<dir>`, and `no-review`. It shows the user the visual tickets' looks, builds the frontier, and posts each PR when it is ready. Start step 3 for a PR as soon as it is posted. The step is done when `stack-implement`'s report is posted. Carry its tables and each PR's worktree path through every later step.

## 3. Review on another family

For each ready PR, start one child on the reviewer's provider, model, and effort with the harness's cross-provider delegation. The brief:

- The PR URL and its worktree path, which the child only reads.
- Call the Skill tool with `pr-review`, passing the PR and one `spec=<notes file>` for the PR's spec and for each ticket merged into it, and publish the review. Those files hold the issues' text: make no tracker calls for them.
- The app checks run once per PR in step 7, owned by the owner in `preflight.md`. Drop a finding that says they are missing.
- Report the review URL, the head SHA reviewed, and the finding count per axis and per priority.

Run the children in parallel. The step is done when every PR has a review signed `🤖` by the reviewer (check with `gh api repos/<owner>/<repo>/pulls/<number>/reviews`), or every reviewer on the list has failed for it. A PR with no cross-family review gets one sub-agent on your own model that runs `pr-review` the same way, and step 8 flags it.

## 4. Address the review

For each PR, once its review is in, dispatch one sub-agent on your own model, in parallel. The brief:

- The PR URL and its worktree path.
- Call the Skill tool with `address-review`, passing the PR. It waits for review bots still reviewing.
- When a fix needs a decision that belongs to the user, take the recommended option and report it as a decision. When that decision is about schema, auth, or public API, leave the comment unreplied and report the question with its thread link and a recommendation.
- Report the counts per outcome, the pushed commits, the files the fixes touched, the bots that timed out, the decisions, and the open questions.

Record the decisions and the questions as the run rules say. The step is done when every comment on every work list has a posted reply or sits under `## Needs your answer`.

## 5. Verify the fixes

For each PR whose head moved since the reviewer's review, start one child on the reviewer as in step 3, briefed to call the Skill tool with `pr-review`, passing the PR, the same `spec=` files, and `verify=<head SHA it reviewed>`. `pr-review` checks the fix diff alone, and reviews in full only after a P0 or P1 finding or a fix outside the reviewed files. When it publishes new findings, run step 4 once more for that PR. Leave any later review to the user. The step is done when the reviewer has verified each moved PR, or its error is recorded.

## 6. Level with the base

For each PR that `gh pr view <pr> --json mergeable` reports as `CONFLICTING`, dispatch one sub-agent in its worktree to merge `origin/<base>` into the PR branch, resolve the conflicts in favour of what the tickets, specs, and reviews settled, run the covering checks for the conflicted files, and push with a plain `git push`. A conflict those decisions do not settle is aborted with `git merge --abort` and recorded with its files. The step is done when no PR conflicts, or each conflict is recorded.

## 7. Check the running app

For each PR, at its final head, in its worktree, one PR at a time:

1. Start the dev server from `checks.md` on a port of its own, in the background, and leave it running for the user.
2. When the app checks are yours, run the browser suite once and the repo's UI check on the screens the PR's tickets change, and record each failure with its ticket. Dispatch one sub-agent to fix the failures, run the covering checks, and push.
3. When their owner is `manual, user`, write the checklist: each screen the tickets change, and what to confirm on it.

The step is done when every PR has a dev server URL, or the exact command when the server would not start, and either a result for the app checks or a checklist.

## 8. Report

Report without waiting for CI. For each PR, give its link, its worktree, the dev server URL, and a table of ticket, parent spec, and state (merged, parked, failed, or waiting on a look), then the reviewer and its findings per priority, the `address-review` outcomes (fix, decline, answer), the verification result, and the checks (passed, pending, or failed, from `gh pr checks <pr>`). Then list what waits on the user:

- The manual checklist from step 7.
- Requested human reviewers who have not reviewed, bots that timed out, failed checks, and recorded conflicts.
- PRs with no cross-family review.
- From `stack-implement`'s report: the tickets still waiting on a look, the failed tickets, the issues left out with their reasons, and the specs no planned ticket has as its parent.

End with `decisions.md` in full, `## Needs your answer` first. Running `ship` again on the same source builds the tickets the user has since answered or approved.

## 9. Watch the checks

Watch the required checks on every PR's head with the harness's PR watch. Without one, read `gh pr checks <pr>` once, report it, and stop. For each PR with a failed check, dispatch one sub-agent in its worktree to read the failing log, fix the cause, run the check locally, and push with a plain `git push`. The step is done when every required check passes, or each PR has had one fix round. Then stop the watch and post a one-line result per PR, naming the checks still failing, followed by `decisions.md`.
