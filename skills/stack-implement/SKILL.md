---
name: stack-implement
description: Trigger this when the user asks to implement several issues, or a milestone or project, as stacked PRs.
argument-hint: "<milestone, project, or issue list> [base branch]"
---

# Stack implement

Arguments: `$ARGUMENTS`

- The source: a milestone, project, or list of issues (URLs or IDs).
- The base branch for the first item. Empty: the repo's default branch.

You are the orchestrator. Every issue gets its own worktree and its own subagent, and the subagent runs the `implement` skill. Each branch starts from the branch before it, so the PRs form a **stack**: every PR's base is the previous item's branch, and the first PR's base is the base branch.

The user's request authorizes creating the branches and worktrees, pushing them, opening the PRs, and moving each issue to In Progress, with no further confirmation. Merging stays with the user.

## 1. Plan the stack

Fetch every issue in the source, reading every page. Order the issues so each one comes after every in-stack issue that blocks it, and break ties by the source's order. For each blocker outside the stack, check that it is merged into the base branch; if one is not, stop and report it.

Name each item's branch in the repo's convention, falling back to `feat/<issue id in lowercase>`.

Post the plan as a numbered list of issue, branch, and base. The step is done when every issue in the source appears exactly once and every in-stack blocker sits above the issues it blocks. The plan is a record, so go straight on to step 2.

## 2. Build each item, in order

For each item, from the top:

1. Fetch `origin`, then create the item's worktree on a new branch from `origin/<base>`: the repo's worktree tool when it has one, otherwise `git worktree add -b <branch> <path> origin/<base>`.
2. Dispatch one subagent with the brief below, and wait for its report.
3. Check the PR: `gh pr view <branch> --json url,state,baseRefName` shows an open PR whose base is the item's base. Correct a wrong base with `gh pr edit <branch> --base <base>`. Post the PR link with the stack position (`2/7`).

The item is done when that check passes. The next item then starts, with this item's branch as its base. When an item ends without a PR, stop the stack and go to step 3: every later item builds on this one.

### Delegate brief

The brief carries this item's issue alone, so the delegate gives the whole of its attention to it. Include:

- The issue ID, its link, and the stack position.
- The worktree path and branch. All work happens there.
- The base branch, which is also the PR's base.
- Read the `implement` skill's `SKILL.md` (in the skills directory, such as `~/.claude/skills/implement/`) and follow it for this issue. (`implement` is user-invoked, so the Skill tool cannot load it.)
- Move the issue to In Progress before starting.
- After `implement` finishes, push the branch and open a PR with `--base <base>`. Write the commits and PR text with the repo's commit and PR description skills when installed. Leave the PR unmerged.
- When an ask-first decision comes up (public API, schema, a new dependency, a lockfile shape) and the issue does not settle it, take the most conservative option and record it in the PR description and the report.
- Report the PR URL, the head commit, the checks run with their results, every conservative decision, and anything left undone.

## 3. Report

Give a table of position, issue, branch, base, and PR. Under it, list each conservative decision with its PR. If the stack stopped, name the failing item, why it failed, and the items not started.
