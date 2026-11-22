---
name: batch-implement
description: Trigger this when the user asks to implement a spec, several tickets, a milestone, or a project.
argument-hint: "<spec, milestone, project, or ticket list> [base branch] [notes=<dir>]"
---

# Batch implement

Arguments: `$ARGUMENTS`

- The source: a spec, a milestone, a project, or a list of tickets (URLs or IDs).
- The base branch every integration branch starts from. Empty: the repo's default branch.
- `notes=<dir>`: the run notes directory. Empty: create one outside the repo, in the harness's scratch directory when it has one.

This skill is Matt Pocock's `implement-spec`, run for every spec in the source. Each spec's tickets land on one **integration branch** with one PR. The tickets are a task graph: the **frontier** is every ticket whose blockers are merged, and the whole frontier builds at once.

You are the orchestrator. This skill builds and opens the PRs; reviewing them is the caller's work (`ship`, or the user with `pr-review`). Every ticket gets its own worktree and a **builder**, a subagent started with a fresh context window, in the background where the harness can. The builder builds the ticket test-first and commits; you merge its branch into the integration branch, push, and open the PR. With one ticket in the plan, you are the builder, so the user watches the work and can steer it.

The user's request authorizes creating the branches and worktrees, pushing the integration branches, opening their PRs, and moving each ticket to In Progress, with no further confirmation. Merging a PR stays with the user.

## Vocabulary

The words are Matt Pocock's. Sort every issue the source brings in, in this order:

- **Wayfinder issue**: an issue labelled `wayfinder:map` (a map) or `wayfinder:<type>` (a decision ticket). It holds decisions, not build work, even when a map has sub-issues; a cleared map goes to `to-spec`.
- **Spec**: what `to-spec` publishes. An issue with sub-issues, or one whose body has `to-spec`'s `## Problem Statement` and `## User Stories` sections. A spec with no sub-issues is **unticketed**: `to-tickets` has not split it yet, so it has nothing to build.
- **Ticket**: every other issue. A sub-issue of a spec is what `to-tickets` publishes; a standalone one may come from `triage`. Its **blocking edges** name the tickets that gate it.
- **Triage state**: the ticket's triage role, read through the repo's triage-label mapping. A ticket is **buildable** when its state is `ready-for-agent` or it has none; `ready-for-human`, `needs-triage`, `needs-info`, and `wontfix` are not.
- **Visual ticket**: a ticket that changes what a screen looks like: layout, a new component, styling, or on-screen copy.

## Run rules

- **Wait by ending the turn.** After you start background work, end the turn; its completion brings you back. Every turn you take starts work, merges work, or reports a result. When the harness cannot run a subagent in the background, run it in the foreground.
- **Decide and record.** With more than one ticket in the plan, the user is away: put no question to them after step 1. When a choice comes up that the ticket and its spec leave open, take the recommended or most conservative option and append it to `<notes>/decisions.md` as the ticket, the question, the option taken, and what the alternative changes. You alone write that file, from the builders' reports.
- **Park on schema, auth, or public API.** A choice in one of those areas that the ticket and its spec leave open parks its ticket: the ticket and every ticket it blocks leave the frontier, the question goes under `## Needs your answer` in `<notes>/decisions.md` with a recommendation, and the rest of the run goes on.
- **Covering checks.** Builders and mergers run only the typecheck, the lint, and the tests that exercise the files they changed. CI runs the full suite on every push.

## 1. Plan

The issue tracker should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills`. Use its workflow for every issue read and write below.

Fetch every issue in the source, reading every page, and drop closed issues. Read each issue's parent, labels, and blocking edges, and each spec's sub-issues. Leave out, wherever it appears in the source or among a spec's sub-issues, with its reason: every wayfinder issue (clear the map, then run `/to-spec`), every unticketed spec (run `/to-tickets` on it), and every ticket that is not buildable (its triage state). Replace each remaining spec with its open sub-issues until no item is a spec, and keep each ticket once. Then split the tickets into integration branches:

- The tickets of one parent spec share one integration branch.
- Tickets with no parent spec share an integration branch only when blocking edges join them. Every other one gets its own.

A blocker outside the plan is cleared when its PR is merged into the base branch, or, when it has no PR (a decision or a prototype), when the tracker shows it closed as done. For each blocker that is not cleared, leave out the blocked ticket and every ticket that depends on it, directly or through others. When no ticket is left, stop.

Name each integration branch in the repo's convention, falling back to `feat/<spec or ticket id in lowercase>`, and each ticket branch `<integration branch>-<ticket id in lowercase>`. Ticket branches stay local.

Write the run notes, which every later brief points to in place of restating them:

- `tickets/<ticket id>.md` and `specs/<spec id>.md`: each issue's title, link, and body, as fetched.
- `checks.md`, unless the notes already hold one: the repo's exact commands for the typecheck, the lint, one test file, the full suite, the browser suite, and the dev server, taken from its docs and scripts, and whether CI runs the full suite on a push.
- `decisions.md`: empty.

For each visual ticket, get its **look** before any build: the mockup or screenshot the ticket carries, or else one subagent per ticket, started with a fresh context window, that calls the Skill tool with `prototype` and saves a mockup image under `looks/` in the notes. Show every look to the user in one message, each under its ticket ID. A visual ticket joins the frontier when the user approves its look; save what they said to `looks/<ticket id>.md`.

Post the plan: for each integration branch, its base and a list of ticket, parent spec (or none), ticket branch, blockers, and whether it waits on a look. Under it, list each spec from the source that no planned ticket has as its parent, each issue left out with its reason (its open blocker, its triage state, or the skill to run first), and the notes directory. The step is done when every open issue from the source, or from expanding a spec, appears exactly once: as a planned ticket, as a spec, or as left out with its reason. The plan is a record, so go straight on to step 2 while the looks are made and answered.

## 2. Build the frontier

An earlier run may have left work behind. Reuse an integration branch that exists, with its worktree and its PR. A ticket whose merge commit is on it (`git log --merges --grep '<ticket id>' origin/<integration branch>`) is built. A ticket branch that exists without a merge keeps its commits: reuse it and tell the builder about them.

Fetch `origin`, then give each integration branch a worktree on a new branch from `origin/<base>`: the repo's worktree tool when it has one, otherwise `git worktree add -b <branch> <path> origin/<base>`.

A ticket is on the frontier when it is not built, not parked, not waiting on a look, and every blocker it has in the plan is merged into its integration branch. Repeat until the frontier is empty and no builder is running:

1. **Start** every ticket on the frontier: create its worktree on its ticket branch from the tip of its integration branch, and dispatch a builder with the build brief. An integration branch with one ticket has no ticket branch: its builder works in the integration worktree.
2. **Wait** by ending the turn.
3. **Merge** each builder that reports a head commit, one merge at a time, in the integration worktree: `git merge --no-ff <ticket branch> -m "Merge <ticket id>: <ticket title>"`. On a conflict, dispatch a merger subagent with a fresh context window, pointed at the two tickets' notes, to resolve it in favour of what the tickets and the spec settled, run the covering checks for the conflicted files, and commit. Then push the integration branch with a plain `git push`, and remove the ticket's worktree and branch.
4. **Open the PR** after an integration branch's first merge: call the Skill tool with `pr-description`, passing the base, and run `gh pr create --draft --base <base>`.
5. **Record** each decision the builder reported, and park the ticket of each question it stopped on. A builder that ends without a head commit fails its ticket: the ticket and every ticket it blocks leave the frontier, with the reason. When it reports a commit that failed on signing, tell the user once to unlock the signer, then commit its staged changes yourself.

When the loop ends, finish each integration branch that has a merge:

1. When `checks.md` says CI does not run the full suite, run it once in the integration worktree, one integration branch at a time, and dispatch one fresh builder to fix what fails.
2. Update the PR body with `pr-description`: it closes every merged ticket, and the spec when all of its tickets are merged, and it records the decisions taken for those tickets.
3. Run `gh pr ready <pr>` and post the PR link with its count of merged and planned tickets.

The step is done when every planned ticket is merged, parked, failed, or still waiting on a look, and every integration branch with a merge has a ready PR.

### Build brief

Send this text with its slots filled, and nothing else. It carries one ticket, so the builder gives the whole of its attention to it. As the builder of a one-ticket plan, follow it yourself and ask the user in place of reporting.

```text
Build <ticket id> (<ticket link>), a ticket of <spec link, or "no spec">.

Read first: <notes>/tickets/<ticket id>.md, <notes>/specs/<spec id>.md,
<notes>/checks.md, and <notes>/looks/<ticket id>.md when it exists. They hold
the settled decisions and the approved look. Fetch nothing they already hold.

Worktree: <path>, on branch <ticket branch>, cut from <integration branch>.
All work happens there. <Earlier commits on the branch, when it was reused.>

1. Move the ticket to In Progress with the tracker's workflow, and state its
   title.
2. Call the Skill tool with `tdd` and build the ticket. The seams the ticket
   or its spec names are confirmed. Where they name none, choose the seams
   and list them in your report.
3. Run the covering checks from checks.md as you go, and once after your last
   change: the typecheck, the lint, and the tests that exercise the files you
   changed. The full suite belongs to CI. The checks that need a running app
   (the browser suite, a UI check) belong to the orchestrator.
4. Before every commit, call the Skill tool with `commit-message`. Commit on
   your branch. When a commit fails on signing, keep the changes staged,
   retry once, and report the exact error.
5. Merge the tip of <integration branch> into your branch, and run the
   covering checks for the files that merge touched.

You are done at step 5. The orchestrator merges your branch, pushes, and
opens the PR.

Decisions: your report is your only channel to the user. When the ticket and
its spec leave a choice open, take the most conservative option and go on.
When that choice is about schema, auth, or public API, commit what you have,
stop, and report the question with your recommendation.

You may start subagents, each with a fresh context window.

Report: the head commit, each covering check with its result, each decision
as "question, option taken, what the alternative changes", the seams you
chose, and anything left undone.
```

## 3. Report

For each integration branch, give its PR, its worktree, and a table of ticket, parent spec, state (merged, parked, failed, or waiting on a look), and merge commit. Then give:

- `decisions.md` in full, with `## Needs your answer` first.
- The checks that need a running app, as a checklist for the user, with the dev server command from `checks.md`.
- The specs no planned ticket has as its parent, and the issues left out with their reasons.
- Each failed ticket with its reason and the tickets it blocks.

Remove the ticket worktrees and ticket branches this run created that are merged or hold no commits.
