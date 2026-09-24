---
name: pr-description
description: Writes a pull request description for the current branch, focused on why the change exists. Use when the user asks for a PR description, or before you run gh pr create. Read-only.
allowed-tools: Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git status:*), Bash(git branch:*), Bash(git rev-parse:*)
---

# PR description

Write a PR description for the current branch. This skill writes the text only.

## Context

- Current branch: !`git branch --show-current`
- Base branch: !`git rev-parse --abbrev-ref origin/HEAD 2>/dev/null || echo "main"`
- Commits on this branch: !`git log --oneline origin/HEAD..HEAD 2>/dev/null || git log --oneline main..HEAD 2>/dev/null || git log --oneline master..HEAD 2>/dev/null`
- Diff against base: !`git diff origin/HEAD...HEAD 2>/dev/null || git diff main...HEAD 2>/dev/null || git diff master...HEAD 2>/dev/null`

## Process

1. If the branch has no commits ahead of the base, say so and stop.
2. If the `the-writing-whip` skill is available, load it before you write any text. Else, write plain and short, with concrete words.
3. Find out why this change exists. If the diff and the commits do not show it, ask. Put all questions in one message. Continue when you can state the why in one sentence.
4. If the repository has a PR template, use it as a menu. Keep the sections that bear on this change and drop the rest.
5. Read the draft as the reviewer. Cut every sentence that the diff already says.

## Output

Write for a reviewer who reads the diff next. Give them what the diff cannot: the why. Spend most words on the problem, the motivation, and any tradeoff. Follow with a short what: the few changes that matter, described by intent. Aim for 120 words or fewer.

If a person asked for the description, put only the description in one fenced code block. If you pass it to `gh pr create --body`, use the plain text. Default shape, with an example:

```
## Why
Deploys failed about once a week. Two services updated in parallel and raced on a shared config file.

## What
- Deploy the two services in series.
- Give each service its own copy of the config.
```
