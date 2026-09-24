---
name: draft-pr
description: Draft a pull request description for the current branch against the base branch. Read-only. Never creates, edits, or pushes a PR.
disable-model-invocation: true
allowed-tools: Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git status:*), Bash(git branch:*), Bash(git rev-parse:*)
---

# Draft PR

Draft a PR description for the current branch. The user opens the PR. You only write the text.

## Context

- Current branch: !`git branch --show-current`
- Base branch: !`git rev-parse --abbrev-ref origin/HEAD 2>/dev/null || echo "main"`
- Commits on this branch: !`git log --oneline origin/HEAD..HEAD 2>/dev/null || git log --oneline main..HEAD 2>/dev/null || git log --oneline master..HEAD 2>/dev/null`
- Diff against base: !`git diff origin/HEAD..HEAD 2>/dev/null || git diff main..HEAD 2>/dev/null || git diff master..HEAD 2>/dev/null`

## Process

1. If the branch has no commits ahead of the base, say so and stop.
2. Load the `the-writing-whip` skill. Apply it to every word you write.
3. Find the motivation, the test method, and the linked issue. Ask about each one that the diff and the commits do not show. Ask all questions in one message.
4. If the repository has a PR template (for example `.github/PULL_REQUEST_TEMPLATE.md`), fill it. Else use the format below.

## Output

Only the description, in one fenced code block.

```
## Summary
<1-2 sentences: the problem and the fix.>

## Changes
- <3-5 bullets. Name files, functions, and behaviors.>
```

Caps:

- ≤120 words in total, including a repository template.
- Only the `Summary` and `Changes` headings, unless the repository template or the user asks for more.
- Leave out a section that has no content.
