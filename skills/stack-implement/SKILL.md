---
name: stack-implement
description: Trigger this when the user asks to implement several issues, or a milestone or project, as stacked PRs.
argument-hint: "<milestone, project, or issue list> [base branch]"
---

# Stack implement

Arguments: `$ARGUMENTS`

- The source: a milestone, project, or list of issues (URLs or IDs).
- The base branch for the first item. Empty: the repo's default branch.

You are the orchestrator. Every issue gets its own worktree and its own subagent, which builds the issue test-first, opens its PR, and reviews it with `pr-review`. Each branch starts from the branch before it, so the PRs form a **stack**: every PR's base is the previous item's branch, and the first PR's base is the base branch.

The user's request authorizes creating the branches and worktrees, pushing them, opening the PRs, linking them into a stack on GitHub, and moving each issue to In Progress, with no further confirmation. Merging stays with the user.

The repository's `origin` decides the forge, GitHub or Gitea. Before the first PR or issue command, call the Skill tool with `forge`, and run its command for each operation in bold.

## 1. Plan the stack

Fetch every issue in the source, reading every page (**Milestone issues** for a milestone on the forge), and drop closed issues. Order the issues so each one comes after every in-stack issue that blocks it, and break ties by the source's order. For each blocker outside the stack, check that it is merged into the base branch; if one is not, stop and report it.

Name each item's branch in the repo's convention, falling back to `feat/<issue id in lowercase>`.

Post the plan as a numbered list of issue, branch, and base. The step is done when every issue in the source appears exactly once and every in-stack blocker sits above the issues it blocks. The plan is a record, so go straight on to step 2.

## 2. Build each item, in order

For each item, from the top:

An earlier run may have left work behind. When the item's branch has an open PR, reuse the PR and the branch's worktree (`git worktree add <path> <branch>` when it has none), then go to check 3. When the branch exists without a PR, reuse it in the same way, skip creating it in 1, and tell the delegate about its earlier commits.

1. Fetch `origin`, then create the item's worktree on a new branch from `origin/<base>`: the repo's worktree tool when it has one, otherwise `git worktree add -b <branch> <path> origin/<base>`.
2. Dispatch one subagent, started with a fresh context window, with the brief below, and wait for its report. When it reports a commit that failed on signing, ask the user once to unlock the signer, then commit its staged changes yourself.
3. Check the PR: **Find PR** shows an open PR whose base is the item's base. Correct a wrong base with **Set base**. Post the PR link with the stack position (`2/7`).

The item is done when that check passes. The next item then starts, with this item's branch as its base. When an item ends without a PR, stop the stack and go to step 3 with the items that have PRs: every later item builds on this one.

### Delegate brief

The brief carries this item's issue alone, so the delegate gives the whole of its attention to it. Include:

- The issue ID, its link, and the stack position.
- The worktree path and branch. All work happens there, on that branch name.
- The base branch, which is also the PR's base.
- Move the issue to In Progress before starting, where its tracker has that state.
- Build the issue: call the Skill tool with `tdd`, where possible at the seams the issue or its spec agrees. Run the typecheck and single test files as you go, and the full test suite once at the end.
- Call the Skill tool with `forge`, and run its command for each operation in bold.
- Before every commit, call the Skill tool with `commit-message`. Before **Open PR** or **Edit body**, call it with `pr-description`, passing the base.
- Commit, push the branch, and open a PR against the base with **Open PR**. The body links the issue. Leave the PR unmerged.
- Self review: call the Skill tool with `pr-review`, passing the PR and `draft`. Fix every finding that holds up, run the full test suite, then commit and push. The self review is done when every finding is fixed or declined with evidence.
- When a commit fails on signing, keep the changes staged and report the exact error; retry at most once.
- When an ask-first decision comes up (public API, schema, a new dependency, a lockfile shape) and the issue does not settle it, take the most conservative option and record it in the PR description and the report.
- Report the PR URL, the head commit, the checks run with their results, each self-review finding with its outcome (or why `pr-review` could not run), every conservative decision, and anything left undone.

## 3. Link the stack

Skip the step when the stack has one PR.

**GitHub.** Without a GitHub stack, GitHub merges each PR into the branch below it, so merging the top PR lands only on the previous item's branch. Link every open PR, bottom to top, into one stack with the base branch as its trunk:

```sh
gh stack link --base <base> <pr 1> <pr 2> ... <pr n>
```

Pass PR numbers, not branch names: `gh stack link` pushes branch arguments before it looks them up. Leave out `--base` and the trunk becomes the repo's default branch, which is wrong when the user named another base.

The step is done when `gh api repos/{owner}/{repo}/pulls/<top pr> --jq .stack` shows `base.ref` equal to the base branch and `size` equal to the number of PRs. When `gh stack` is not installed (`gh extension list` has no `github/gh-stack`), skip the link and put the `gh stack link` command in the report for the user to run.

**Gitea.** Gitea has no stacks: each PR stays based on the previous item's branch, so merging a PR lands it only on that branch. Link nothing. Put the **merge order** in the report: merge the PRs bottom first, each into the base branch. Before each merge after the first, check that the PR's base is the base branch: Gitea retargets it when the PR below merges, unless the instance turned `RETARGET_CHILDREN_ON_MERGE` off, in which case correct it with **Set base**. When the merge below squashed or rebased, rebase the PR onto the base branch first. The step is done when the merge order lists every PR, bottom first.

## 4. Report

On GitHub, give the stack number from `.stack.number`; on Gitea, give the merge order from step 3. Then give a table of position, issue, branch, base, PR, and self-review findings fixed and declined. Under it, list each conservative decision with its PR. Give the top PR's branch and worktree, where the dev server shows every change in the stack. If the stack stopped, name the failing item, why it failed, and the items not started. Remove the branches and worktrees this run created that hold no commits ahead of their base.
