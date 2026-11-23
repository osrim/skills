---
name: pr-description
description: Trigger this when the user asks for a PR description, or before you run `gh pr create` or `gh pr edit --body`.
argument-hint: "[base branch] [evidence files...]"
allowed-tools: Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git status:*), Bash(git branch:*), Bash(git rev-parse:*), Bash(git fetch:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(gh pr checks:*)
---

# PR description

Write a PR description for the current branch: why the change exists, the **evidence** that it works, and its **merge danger**. This skill writes the text only.

## Process

1. Find the base: the argument; else the open PR's base (`gh pr view --json baseRefName --jq .baseRefName`); else the default branch (`git rev-parse --abbrev-ref origin/HEAD`). The PR is **stacked** when its base is another open PR's head branch (`gh pr list --head <base>`); the description then covers this layer's commits alone.
2. Run `git fetch origin <base>`, then read the commits (`git log --oneline origin/<base>..HEAD`) and the diff (`git diff origin/<base>...HEAD`). If the branch has no commits ahead of the base, say so and stop.
3. Find out why this change exists. If the diff and the commits do not show it, ask. Put all questions in one message. Continue when you can state the why in one sentence.
4. Gather the evidence, from what actually ran: the evidence files or check results the caller passed, the tests the diff adds or changes, and `gh pr checks`. Record a before only when a source shows it.
5. Judge the merge danger against the rules below.
6. For a stacked PR, list the whole stack, bottom-up: walk down with `gh pr list --head <base>` and up with `gh pr list --base <current branch>`.
7. If the repository has a PR template, use it as a menu. Keep the sections that bear on this change and drop the rest. When replacing an existing body, carry over the sections other skills or people added, such as recorded decisions and issue links.
8. Write plain statements in concrete words, and keep one noun for each thing. Punctuate with commas, colons, parentheses, or periods. Then read the draft as the reviewer and cut every sentence that the diff already says.

## Template

Spend most words on the why. Aim for 180 words or fewer, not counting a visual or the stack list. Example:

```
## Stack
Merge bottom-up. This PR is layer 2.
- #86
- #87 (this PR)
- #88

## Why
Deploys failed about once a week. Two services updated in parallel and raced on a shared config file. Closes #81.

## What
- Deploy the two services in series.
- Give each service its own copy of the config.

## Evidence
- **Before:** `deploy.test.ts` "parallel deploys keep both configs" failed: the second write replaced the first.
- **After:** the same test passes.

## Merge danger
**Door:** two-way. Reverting restores the shared file, which both services still read.
**Blast radius:** deploys of the API and the worker.
```

- **Stack:** only for a stacked PR. One PR number per bullet, so GitHub shows each title. Mark this PR.
- **Why:** the problem, the motivation, and any tradeoff. Link the issue the PR resolves.
- **What:** the few changes that matter, by intent, in up to three bullets. When the shape of the change matters more than its intent (a refactor, moved files, a new call path), use one small visual in place of the bullets: a call tree, a file tree, or a shaped `diff` block.
- **Evidence:** a before and an after. A screenshot when the change is visual and the caller passed one. Otherwise the test that failed and now passes, by file and name, or the command output that changed. With nothing that ran, write `None:` and the reason.
- **Merge danger:**
  - **Door:** one-way when a revert cannot undo the change: a schema migration, data deleted or rewritten, a public API removed or changed, anything sent outward (emails, webhooks, a published package), or a new data format once the first record is written. Everything else is two-way. A one-way door names what makes it one-way and how the PR limits the damage. The repo's agent docs may name more one-way changes.
  - **Blast radius:** what breaks if the change is wrong, in a few words.
  - For a stacked PR, judge this layer merged alone on top of the layers below it: each layer must be safe to merge on its own. When it is not, say so.
- **Decisions:** carried over from an existing body, when present.

## Output

If a person asked for the description, put only the description in one fenced code block. If you pass it to `gh pr create --body` or `gh pr edit --body`, use the plain text.
