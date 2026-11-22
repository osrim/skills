---
name: stack-implement
description: Trigger this when the user asks to implement a spec, several tickets, or a milestone or project, as stacked PRs.
argument-hint: "<spec, milestone, project, or ticket list> [base branch]"
---

# Stack implement

Arguments: `$ARGUMENTS`

- The source: a spec, a milestone, a project, or a list of tickets (URLs or IDs).
- The base branch for each stack's first ticket. Empty: the repo's default branch.

This skill is the build step of Matt Pocock's main flow, run at scale: after `to-spec` and `to-tickets`, it does what `implement` does, once per ticket in a fresh context, and lands the tickets as stacked PRs. `implement` runs only when the user invokes it, so the build brief below carries its steps, with `pr-review` in place of `code-review`.

You are the orchestrator. Every ticket gets its own worktree and a **builder**, which builds the ticket test-first, opens its PR, and reviews it with `pr-review`. With one ticket in the plan, you are the builder; with more, each ticket gets its own subagent. Each branch starts from the branch before it, so the PRs form a **stack**: every PR's base is the previous ticket's branch, and the first PR's base is the base branch.

The user's request authorizes creating the branches and worktrees, pushing them, opening the PRs, linking them into GitHub stacks, and moving each ticket to In Progress, with no further confirmation. Merging stays with the user.

## Vocabulary

The words are Matt Pocock's. Sort every issue the source brings in, in this order:

- **Wayfinder issue**: an issue labelled `wayfinder:map` (a map) or `wayfinder:<type>` (a decision ticket). It holds decisions, not build work, even when a map has sub-issues; a cleared map goes to `to-spec`.
- **Spec**: what `to-spec` publishes. An issue with sub-issues, or one whose body has `to-spec`'s `## Problem Statement` and `## User Stories` sections. A spec with no sub-issues is **unticketed**: `to-tickets` has not split it yet, so it has nothing to build.
- **Ticket**: every other issue. A sub-issue of a spec is what `to-tickets` publishes; a standalone one may come from `triage`. Its **blocking edges** name the tickets that gate it.
- **Triage state**: the ticket's triage role, read through the repo's triage-label mapping. A ticket is **buildable** when its state is `ready-for-agent` or it has none; `ready-for-human`, `needs-triage`, `needs-info`, and `wontfix` are not.

## 1. Plan the stacks

The issue tracker should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills`. Use its workflow for every issue read and write below.

Fetch every issue in the source, reading every page, and drop closed issues. Read each issue's parent and labels, and each spec's sub-issues. Leave out, wherever it appears in the source or among a spec's sub-issues, with its reason: every wayfinder issue (clear the map, then run `/to-spec`), every unticketed spec (run `/to-tickets` on it), and every ticket that is not buildable (its triage state). Split the rest into stacks:

- **A spec or a ticket list** is one stack. Replace each spec in it with its open sub-issues, in the tracker's order, until no item is a spec. A spec whose sub-issues are all closed adds no ticket.
- **A milestone or project** gives one stack per spec. Drop every spec, then group the remaining tickets by parent: each parent's tickets form one stack, and the tickets with no parent form one more stack.

Keep each ticket once. Within each stack, order the tickets so each one comes after every in-stack ticket that blocks it, and break ties by the source's order. A blocker outside its stack is cleared when its PR is merged into the base branch, or, when it has no PR (a decision or a prototype), when the tracker shows it closed as done. For each blocker that is not cleared, leave out the blocked ticket and every ticket in its stack that depends on it, directly or through others. When no stack has a ticket left, stop.

Name each ticket's branch in the repo's convention, falling back to `feat/<ticket id in lowercase>`.

Post the plan: for each stack, a numbered list of ticket, parent spec (or none), branch, and base. Under it, list each spec from the source that no planned ticket has as its parent, and each issue left out with its reason: its open blocker, its triage state, or the skill to run first. The step is done when every open issue from the source, or from expanding a spec, appears exactly once: as a planned ticket, as a spec, or as left out with its reason, and every in-stack blocker sits above the tickets it blocks. The plan is a record, so go straight on to step 2.

## 2. Build each ticket, in order

Take the stacks one at a time, in the plan's order, through steps 2 and 3. Every stack's first ticket takes the base branch as its base. For each ticket, from the top:

An earlier run may have left work behind. When the ticket's branch has an open PR, reuse the PR and the branch's worktree (`git worktree add <path> <branch>` when it has none), then go to check 3. When the branch exists without a PR, reuse it in the same way, skip creating it in 1, and tell the builder about its earlier commits.

1. Fetch `origin`, then create the ticket's worktree on a new branch from `origin/<base>`: the repo's worktree tool when it has one, otherwise `git worktree add -b <branch> <path> origin/<base>`.
2. Build the ticket with the brief below.
   - **One ticket in the plan:** follow the brief yourself, in the ticket's worktree, so the user watches the work and can steer it. Ask the user about an ask-first decision when it comes up, and record the answer in the PR description. On a commit that fails on signing, ask the user to unlock the signer, then commit.
   - **More tickets:** dispatch one subagent per ticket, started with a fresh context window, so each ticket's context stays out of the next, and wait for its report. When it reports a commit that failed on signing, ask the user once to unlock the signer, then commit its staged changes yourself.
3. Check the PR: `gh pr view <branch> --json url,state,baseRefName` shows an open PR whose base is the ticket's base. Correct a wrong base with `gh pr edit <branch> --base <base>`. Post the PR link with the stack position (`2/7`).

The ticket is done when that check passes. The next ticket then starts, with this ticket's branch as its base. When a ticket ends without a PR, stop its stack and go to step 3 with the tickets that have PRs: every later ticket builds on this one. The next stack then starts.

### Build brief

The brief carries this ticket alone, so the builder gives the whole of its attention to it. Include:

- The ticket ID, its link, and the stack position, plus its parent spec's link when it has one.
- The worktree path and branch. All work happens there, on that branch name.
- The base branch, which is also the PR's base.
- Move the ticket to In Progress before starting, with the tracker's workflow, and state its title.
- Build the ticket: call the Skill tool with `tdd`, where possible at the seams the ticket or its spec agreed. As a subagent, treat those seams as confirmed; where the ticket and its spec name none, choose the seams and list them in your report. Run the typecheck and single test files as you go. The build is done when one full test suite run passes after your last change: start it in the background where the harness can, and write the docs meanwhile. On a failure, re-run only the failing test until it passes, then run the full suite once more.
- Before every commit, call the Skill tool with `commit-message`. Before `gh pr create` or `gh pr edit --body`, call it with `pr-description`, passing the base.
- Commit, push the branch, and open a PR with `--base <base>`. The body links the ticket. Leave the PR unmerged.
- Self review: call the Skill tool with `pr-review`, passing the PR, `draft`, and, when the ticket has a parent spec, `spec=<ticket URL> spec=<parent spec URL>`. Fix every finding that holds up, run the typecheck, lint, and the tests that exercise the changed files, then commit and push. CI runs the full suite on the push. The self review is done when every finding is fixed or declined with evidence.
- When a commit fails on signing, keep the changes staged and report the exact error; retry at most once.
- When an ask-first decision comes up (public API, schema, a new dependency, a lockfile shape) and neither the ticket nor its spec settles it, take the most conservative option and record it in the PR description and the report.
- Report the PR URL, the head commit, the checks run with their results, each self-review finding with its outcome (or why `pr-review` could not run), every conservative decision, and anything left undone.

## 3. Link the stack

Without a GitHub stack, GitHub merges each PR into the branch below it, so merging the top PR lands only on the previous ticket's branch. Link every open PR of the stack, bottom to top, into one GitHub stack with the base branch as its trunk:

```sh
gh stack link --base <base> <pr 1> <pr 2> ... <pr n>
```

Pass PR numbers, not branch names: `gh stack link` pushes branch arguments before it looks them up. Leave out `--base` and the trunk becomes the repo's default branch, which is wrong when the user named another base.

The step is done when `gh api repos/{owner}/{repo}/pulls/<top pr> --jq .stack` shows `base.ref` equal to the base branch and `size` equal to the number of PRs in this stack. Skip the step when the stack has one PR. When `gh stack` is not installed (`gh extension list` has no `github/gh-stack`), skip the link and put the `gh stack link` command in the report for the user to run.

## 4. Report

For each stack, give the stack number from `.stack.number`, then a table of position, ticket, parent spec, branch, base, PR, and self-review findings fixed and declined, then the top PR's branch and worktree, where the dev server shows every change in that stack. Under the tables, list each conservative decision with its PR, the specs no planned ticket has as its parent, and the issues left out with their reasons. If a stack stopped, name the failing ticket, why it failed, and the tickets not started. Remove the branches and worktrees this run created that hold no commits ahead of their base.
