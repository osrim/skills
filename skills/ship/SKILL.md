---
name: ship
description: Trigger this when the user asks to implement or ship a spec, a ticket, a milestone, or a project.
argument-hint: "<spec, ticket, milestone, project, ticket list, or PR list> [base branch] [stack] [reviewer=<provider>/<model>[:<effort>][,<next>...]]"
---

# Ship

Arguments: `$ARGUMENTS`

- The source: a spec, a milestone, a project, a list of tickets (URLs or IDs), or a list of open PRs. A PR list skips step 2: write the run notes yourself (`checks.md`, and each PR's tickets and spec) and give each PR branch a worktree.
- The base branch every integration branch starts from. Empty: the repo's default branch.
- `stack`: one PR per ticket, each based on the one below it, so the user can review and merge a spec layer by layer. Without it, each spec gets one PR.
- `reviewer=`: the cross-family reviewers, in the order to try them. Empty: the reviewer and the fallbacks the user's instructions name (`AGENTS.md`, `CLAUDE.md`).

You are the **orchestrator**. Each spec's tickets build in parallel and land on one **integration branch**, with one PR, or with `stack`, one PR per ticket. Every PR then gets a public `pr-review` by a model from a **different family** than the one that built it (Claude, GPT, Gemini: the provider that serves the model does not count). `address-review` fixes or answers every open comment, from that reviewer, from review bots, and from humans. The reviewer then verifies the fixes.

The user's request authorizes creating branches and worktrees, pushing, opening PRs, moving tickets to In Progress, and everything `pr-review` and `address-review` authorize. Merging stays with the user.

Start every sub-agent with a fresh context window, in its PR's worktree, with the brief the step names. Before every commit, call the Skill tool with `commit-message`; before `gh pr create` or `gh pr edit --body`, call it with `pr-description`, passing the PR's base and its tickets' evidence files.

With one ticket in the run, you do the work yourself, following the brief a sub-agent would get, so the user watches it and can steer it, and you ask each question as it comes up.

## Run rules

- **Wait by ending the turn.** After you start background work or a PR watch, end the turn; its completion brings you back. Every turn you take starts work, acts on a result, or reports one. When the harness cannot run a sub-agent in the background, run it in the foreground.
- **Decide and record.** The user is away after step 1: put no question to them. When a choice comes up that the tickets and specs leave open, take the recommended or most conservative option and append it to `<notes>/decisions.md` as the PR or ticket, the question, the option taken, and what the alternative changes. You alone write that file, from the sub-agents' reports.
- **Park on schema, auth, or public API.** A choice in one of those areas that the tickets and specs leave open goes under `## Needs your answer` in `decisions.md` with a recommendation. Its ticket and every ticket it blocks leave the frontier, or its comment stays unreplied, and the rest of the run goes on.
- **Covering checks.** Builders and fix rounds run only the typecheck, the lint, and the tests that exercise the changed files. CI runs the full suite on every push, and step 9 fixes what it finds.
- **Fall back.** When a provider reports a usage limit, start the remaining sub-agents on the model fallback the user's instructions name, and check that the reviewer is still a different family from the builder. When a reviewer fails, retry it once, then move to the next reviewer on the list.

## 1. Preflight

Run each check while the user is still here, and stop with the failing check when one fails:

- **Tracker:** read one issue of the source through the tracker's workflow.
- **GitHub:** `gh auth status` passes.
- **Reviewer:** resolve the reviewer list. Drop each entry of your own model family, then start a child on the first entry with the harness's cross-provider delegation, briefed to reply with its model name. Take the first entry that answers. With no entry left, ask the user for a reviewer. This also proves that sub-agents start.
- **PR watch:** record whether the harness has one.
- **App access:** the checks that need a running app are yours when you can open the dev server's pages signed in, with a browser tool and a session or a documented test login. Otherwise their owner is `manual, user`.

Create the run notes directory outside the repo, in the harness's scratch directory when it has one, and write `preflight.md` with the reviewer, the PR watch, and the owner of the app checks. The step is done when every check has passed.

## 2. Build

Follow [BUILD.md](BUILD.md): it plans the integration branches, shows the user the visual tickets' looks, builds the frontier, and opens the PRs. Start step 3 for a PR as soon as it is ready. The step is done when BUILD.md's completion criterion holds.

## 3. Review on another family

For each ready PR, start a reviewer with [REVIEW-BRIEF.md](REVIEW-BRIEF.md). With `stack`, a layer's spec sources are its own ticket and its spec. Run the reviewers in parallel. The step is done when every PR has a review signed `🤖` by the reviewer (check with `gh api repos/<owner>/<repo>/pulls/<number>/reviews`), or every reviewer on the list has failed for it. A PR with no cross-family review gets one sub-agent on your own model with the same brief, and step 8 flags it.

## 4. Address the review

For each PR, once its review is in, dispatch a sub-agent with [ADDRESS-BRIEF.md](ADDRESS-BRIEF.md), in parallel. Record the decisions and questions it reports as the run rules say. The step is done when every comment on every work list has a posted reply or sits under `## Needs your answer`.

## 5. Verify the fixes

For each PR whose head moved since the reviewer's review, start the reviewer with [REVIEW-BRIEF.md](REVIEW-BRIEF.md) and its verify line. `pr-review` checks the fix diff alone, and reviews in full only after a P0 or P1 finding or a fix outside the reviewed files. When it publishes new findings, run step 4 once more for that PR. Leave any later review to the user. The step is done when the reviewer has verified each moved PR, or its error is recorded.

## 6. Level with the base

Bring each PR level with its base, in its worktree, with a merge and a plain `git push`:

- Without `stack`, level each PR that `gh pr view <pr> --json mergeable` reports as `CONFLICTING`, by merging `origin/<base>`.
- With `stack`, go bottom-up through each stack and level every layer whose base moved since its last merge, by merging the branch below it (`origin/<base>` for the first layer), so the fixes on lower layers reach the layers above.

Dispatch one sub-agent per conflict to resolve it in favour of what the tickets, specs, and reviews settled, and run the covering checks for the conflicted files. A conflict those decisions do not settle is aborted with `git merge --abort` and recorded with its files; with `stack`, the layers above it wait. The step is done when every PR is level, or each conflict is recorded.

## 7. Check the running app

For each PR, at its final head, in its worktree, one PR at a time (with `stack`, only the top layer, which holds every ticket):

1. Start the dev server from `checks.md` on a port of its own, in the background, and leave it running for the user.
2. When the app checks are yours, run the browser suite once and the repo's UI check on the screens the PR's tickets change, and record each failure with its ticket. Dispatch one sub-agent to fix the failures on the PR (with `stack`, on the lowest layer that causes each one), run the covering checks, and push. With `stack`, run step 6 again.
3. When their owner is `manual, user`, write the checklist: each screen the tickets change, and what to confirm on it.

The step is done when every PR has a dev server URL, or the exact command when the server would not start, and either a result for the app checks or a checklist.

## 8. Report

Report without waiting for CI. For each integration branch, give its PRs (with `stack`, in merge order), their worktrees, the dev server URL, and a table of ticket, parent spec, state (merged, parked, failed, or waiting on a look), and merge commit. For each PR, give the reviewer and its findings per priority, the `address-review` outcomes (fix, decline, answer), the verification result, and the checks (passed, pending, or failed, from `gh pr checks <pr>`). Then list what waits on the user:

- The manual checklist from step 7.
- Requested human reviewers who have not reviewed, bots that timed out, failed checks, and recorded conflicts.
- PRs with no cross-family review.
- The tickets still waiting on a look, each failed ticket with its reason and the tickets it blocks, the issues left out with their reasons, and the specs no planned ticket has as its parent.

End with `decisions.md` in full, `## Needs your answer` first. Running `ship` again on the same source builds the tickets the user has since answered or approved.

## 9. Watch the checks

Watch the required checks on every PR's head with the harness's PR watch. Without one, read `gh pr checks <pr>` once, report it, and stop. For each PR with a failed check, dispatch one sub-agent in its worktree to read the failing log, fix the cause, run the check locally, and push with a plain `git push`; with `stack`, run step 6 after the fixes. The step is done when every required check passes, or each PR has had one fix round. Then stop the watch and post a one-line result per PR, naming the checks still failing, followed by `decisions.md` in full.
