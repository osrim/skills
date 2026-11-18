---
name: stack-implement
description: Trigger this when the user asks to implement several issues, a spec's sub-issues, or a milestone or project, as stacked PRs.
argument-hint: "<milestone, project, or issue list> [base branch]"
---

# Stack implement

Arguments: `$ARGUMENTS`

- The source: a milestone, project, or list of issues (URLs or IDs).
- The base branch for each stack's first item. Empty: the repo's default branch.

You are the orchestrator. Every issue gets its own worktree and a **builder**, which builds the issue test-first, opens its PR, and reviews it with `pr-review`. With one item in the plan, you are the builder; with more, each item gets its own subagent. Each branch starts from the branch before it, so the PRs form a **stack**: every PR's base is the previous item's branch, and the first PR's base is the base branch.

The user's request authorizes creating the branches and worktrees, pushing them, opening the PRs, linking them into GitHub stacks, and moving each issue to In Progress, with no further confirmation. Merging stays with the user.

## 1. Plan the stacks

Fetch every issue in the source, reading every page, and drop closed issues. A **spec** is an issue with sub-issues. On GitHub, each issue from `gh api repos/<owner>/<repo>/issues/<n>` or a milestone's REST issue list carries `sub_issues_summary.total`, above 0 for a spec, and `parent_issue_url` when it has a parent; read a spec's sub-issues with `gh api --paginate repos/<owner>/<repo>/issues/<n>/sub_issues`. In a tracker such as Linear, read the issue's sub-issue and parent fields. Split the source into stacks:

- **An issue list** is one stack. Replace each spec in it with its open sub-issues, in the tracker's order, until no item is a spec. A spec with no open sub-issues adds no item.
- **A milestone or project** gives one stack per spec. Drop every spec, then group the remaining issues by parent: each parent's issues form one stack, and the issues with no parent form one more stack.

Keep each issue once. Within each stack, order the issues so each one comes after every in-stack issue that blocks it, and break ties by the source's order. For each blocker outside its stack, check that it is merged into the base branch; if one is not, leave out the blocked issue and every issue in its stack that depends on it, directly or through others. When no stack has an item left, stop.

Name each item's branch in the repo's convention, falling back to `feat/<issue id in lowercase>`.

Post the plan: for each stack, a numbered list of issue, parent spec (or none), branch, and base. Under it, list each spec from the source that no planned issue has as its parent, and each issue left out with its blocker. The step is done when every open issue that is not a spec, from the source or from expanding a spec in an issue list, appears exactly once in the plan, and every in-stack blocker sits above the issues it blocks. The plan is a record, so go straight on to step 2.

## 2. Build each item, in order

Take the stacks one at a time, in the plan's order, through steps 2 and 3. Every stack's first item takes the base branch as its base. For each item, from the top:

An earlier run may have left work behind. When the item's branch has an open PR, reuse the PR and the branch's worktree (`git worktree add <path> <branch>` when it has none), then go to check 3. When the branch exists without a PR, reuse it in the same way, skip creating it in 1, and tell the builder about its earlier commits.

1. Fetch `origin`, then create the item's worktree on a new branch from `origin/<base>`: the repo's worktree tool when it has one, otherwise `git worktree add -b <branch> <path> origin/<base>`.
2. Build the item with the brief below.
   - **One item in the plan:** follow the brief yourself, in the item's worktree, so the user watches the work and can steer it. Ask the user about an ask-first decision when it comes up, and record the answer in the PR description. On a commit that fails on signing, ask the user to unlock the signer, then commit.
   - **More items:** dispatch one subagent per item, started with a fresh context window, so each item's context stays out of the next, and wait for its report. When it reports a commit that failed on signing, ask the user once to unlock the signer, then commit its staged changes yourself.
3. Check the PR: `gh pr view <branch> --json url,state,baseRefName` shows an open PR whose base is the item's base. Correct a wrong base with `gh pr edit <branch> --base <base>`. Post the PR link with the stack position (`2/7`).

The item is done when that check passes. The next item then starts, with this item's branch as its base. When an item ends without a PR, stop its stack and go to step 3 with the items that have PRs: every later item builds on this one. The next stack then starts.

### Build brief

The brief carries this item's issue alone, so the builder gives the whole of its attention to it. Include:

- The issue ID, its link, and the stack position, plus its parent spec's link when it has one.
- The worktree path and branch. All work happens there, on that branch name.
- The base branch, which is also the PR's base.
- Move the issue to In Progress before starting.
- Build the issue: call the Skill tool with `tdd`, where possible at the seams the issue or its spec agrees. Run the typecheck and single test files as you go. The build is done when one full test suite run passes after your last change: start it in the background where the harness can, and write the docs meanwhile. On a failure, re-run only the failing test until it passes, then run the full suite once more.
- Before every commit, call the Skill tool with `commit-message`. Before `gh pr create` or `gh pr edit --body`, call it with `pr-description`, passing the base.
- Commit, push the branch, and open a PR with `--base <base>`. The body links the issue. Leave the PR unmerged.
- Self review: call the Skill tool with `pr-review`, passing the PR, `draft`, and, when the issue has a parent spec, `spec=<issue URL> spec=<parent spec URL>`. Fix every finding that holds up, run the typecheck, lint, and the tests that exercise the changed files, then commit and push. CI runs the full suite on the push. The self review is done when every finding is fixed or declined with evidence.
- When a commit fails on signing, keep the changes staged and report the exact error; retry at most once.
- When an ask-first decision comes up (public API, schema, a new dependency, a lockfile shape) and the issue does not settle it, take the most conservative option and record it in the PR description and the report.
- Report the PR URL, the head commit, the checks run with their results, each self-review finding with its outcome (or why `pr-review` could not run), every conservative decision, and anything left undone.

## 3. Link the stack

Without a GitHub stack, GitHub merges each PR into the branch below it, so merging the top PR lands only on the previous item's branch. Link every open PR of the stack, bottom to top, into one GitHub stack with the base branch as its trunk:

```sh
gh stack link --base <base> <pr 1> <pr 2> ... <pr n>
```

Pass PR numbers, not branch names: `gh stack link` pushes branch arguments before it looks them up. Leave out `--base` and the trunk becomes the repo's default branch, which is wrong when the user named another base.

The step is done when `gh api repos/{owner}/{repo}/pulls/<top pr> --jq .stack` shows `base.ref` equal to the base branch and `size` equal to the number of PRs in this stack. Skip the step when the stack has one PR. When `gh stack` is not installed (`gh extension list` has no `github/gh-stack`), skip the link and put the `gh stack link` command in the report for the user to run.

## 4. Report

For each stack, give the stack number from `.stack.number`, then a table of position, issue, parent spec, branch, base, PR, and self-review findings fixed and declined, then the top PR's branch and worktree, where the dev server shows every change in that stack. Under the tables, list each conservative decision with its PR, the specs no planned issue has as its parent, and the issues left out with their blockers. If a stack stopped, name the failing item, why it failed, and the items not started. Remove the branches and worktrees this run created that hold no commits ahead of their base.
