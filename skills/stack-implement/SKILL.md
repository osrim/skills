---
name: stack-implement
description: Trigger this when the user asks to implement several issues, or a milestone or project, as stacked PRs.
argument-hint: "<milestone, project, or issue list> [base branch]"
---

# Stack implement

Arguments: `$ARGUMENTS`

- The source: a milestone, project, or list of issues (URLs or IDs).
- The base branch for the first item. Empty: the repo's default branch.

You are the orchestrator. Every issue gets its own worktree and its own subagent, which builds the issue test-first, opens its PR, and reviews it with `pr-review`. Each branch starts from the branch before it, so the PRs form a **stack**: every PR's base is the previous item's branch, and the first PR's base is the base branch. Step 3 links the PRs into a GitHub stack whose trunk is the base branch. Without that link, GitHub merges each PR into the branch below it, and merging the top PR lands only on the previous item's branch.

The user's request authorizes creating the branches and worktrees, pushing them, opening the PRs, linking them into a GitHub stack, and moving each issue to In Progress, with no further confirmation. Merging stays with the user.

## 1. Plan the stack

Fetch every issue in the source, reading every page. Order the issues so each one comes after every in-stack issue that blocks it, and break ties by the source's order. For each blocker outside the stack, check that it is merged into the base branch; if one is not, stop and report it.

Name each item's branch in the repo's convention, falling back to `feat/<issue id in lowercase>`.

Post the plan as a numbered list of issue, branch, and base. The step is done when every issue in the source appears exactly once and every in-stack blocker sits above the issues it blocks. The plan is a record, so go straight on to step 2.

## 2. Build each item, in order

For each item, from the top:

1. Fetch `origin`, then create the item's worktree on a new branch from `origin/<base>`: the repo's worktree tool when it has one, otherwise `git worktree add -b <branch> <path> origin/<base>`.
2. Dispatch one subagent, started with a fresh context window, with the brief below, and wait for its report.
3. Check the PR: `gh pr view <branch> --json url,state,baseRefName` shows an open PR whose base is the item's base. Correct a wrong base with `gh pr edit <branch> --base <base>`. Post the PR link with the stack position (`2/7`).

The item is done when that check passes. The next item then starts, with this item's branch as its base. When an item ends without a PR, stop the stack and go to step 3 with the items that have PRs: every later item builds on this one.

### Delegate brief

The brief carries this item's issue alone, so the delegate gives the whole of its attention to it. Include:

- The issue ID, its link, and the stack position.
- The worktree path and branch. All work happens there.
- The base branch, which is also the PR's base.
- Move the issue to In Progress before starting.
- Build the issue: call the Skill tool with `tdd`, where possible at the seams the issue or its spec agrees. Run the typecheck and single test files as you go, and the full test suite once at the end.
- Commit, push the branch, and open a PR with `--base <base>`. Write the commits with the commit skill and the PR body with the PR description skill, given the base, when installed. Leave the PR unmerged.
- Self review: call the Skill tool with `pr-review`, passing the PR and `draft`. Fix every finding that holds up, run the full test suite, then commit and push. The self review is done when every finding is fixed or declined with evidence.
- When an ask-first decision comes up (public API, schema, a new dependency, a lockfile shape) and the issue does not settle it, take the most conservative option and record it in the PR description and the report.
- Report the PR URL, the head commit, the checks run with their results, each self-review finding with its outcome (or why `pr-review` could not run), every conservative decision, and anything left undone.

## 3. Link the stack

Link every open PR, bottom to top, into one GitHub stack with the base branch as its trunk:

```sh
gh stack link --base <base> <pr 1> <pr 2> ... <pr n>
```

Pass PR numbers, not branch names: `gh stack link` pushes branch arguments before it looks them up. Leave out `--base` and the trunk becomes the repo's default branch, which is wrong when the user named another base.

The step is done when `gh api repos/{owner}/{repo}/pulls/<top pr> --jq .stack` shows `base.ref` equal to the base branch and `size` equal to the number of PRs. Skip the step when the stack has one PR. When `gh stack` is not installed (`gh extension list` has no `github/gh-stack`), skip the link and put the `gh stack link` command in the report for the user to run.

## 4. Report

Give the stack number from `.stack.number`, then a table of position, issue, branch, base, PR, and self-review findings fixed and declined. Under it, list each conservative decision with its PR. If the stack stopped, name the failing item, why it failed, and the items not started.
