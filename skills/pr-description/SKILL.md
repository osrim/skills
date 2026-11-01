---
name: pr-description
description: Trigger this when the user asks for a PR description, or before you run `gh pr create` or `gh pr edit --body`.
argument-hint: "[base branch]"
allowed-tools: Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git status:*), Bash(git branch:*), Bash(git rev-parse:*), Bash(gh pr view:*)
---

# PR description

Write a PR description for the current branch. This skill writes the text only.

## Process

1. Find the base: the argument; else the open PR's base (`gh pr view --json baseRefName --jq .baseRefName`); else the default branch (`git rev-parse --abbrev-ref origin/HEAD`). In a stack, the base is the branch below this one, so the description covers this PR's commits alone.
2. Run `git fetch origin <base>`, then read the commits (`git log --oneline origin/<base>..HEAD`) and the diff (`git diff origin/<base>...HEAD`). If the branch has no commits ahead of the base, say so and stop.
3. If the `the-writing-whip` skill is available, load it before you write any text. Else, write plain and short, with concrete words.
4. Find out why this change exists. If the diff and the commits do not show it, ask. Put all questions in one message. Continue when you can state the why in one sentence.
5. If the repository has a PR template, use it as a menu. Keep the sections that bear on this change and drop the rest. When replacing an existing body, carry over the sections other skills or people added, such as recorded decisions and issue links.
6. Read the draft as the reviewer. Cut every sentence that the diff already says.

## Output

Spend most words on the why: the problem, the motivation, and any tradeoff. Follow with a short what: the few changes that matter, described by intent. Link the issue the PR resolves. Aim for 120 words or fewer.

If a person asked for the description, put only the description in one fenced code block. If you pass it to `gh pr create --body`, use the plain text. Default shape, with an example:

```
## Why
Deploys failed about once a week. Two services updated in parallel and raced on a shared config file.

## What
- Deploy the two services in series.
- Give each service its own copy of the config.
```
