---
name: draft-commit
description: Draft a commit message for the staged changes in the repository's commit style. Read-only. Never stages files or runs git commit.
disable-model-invocation: true
allowed-tools: Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git status:*)
---

# Draft commit

Draft a commit message for the staged changes. The user commits. You only write the text.

## Context

- Staged diff: !`git diff --cached`
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -20`

## Process

1. If the staged diff is empty, say so and stop.
2. Load the `the-writing-whip` skill. Apply it to every word you write.
3. Find the intent and the scope of the change. If the diff holds more than one concern, propose a split in one line. Then draft one message for the full diff.
4. Match the format, scopes, and casing of the recent commits. If the history has no clear style, use `type(scope): subject`.
5. If the branch name holds a ticket ID (for example `ABC-123`), add a `Refs: ABC-123` trailer.

## Output

Only the message, in one fenced code block.

Caps:

- Subject: ≤72 characters, imperative mood, no trailing period.
- Body: optional, ≤3 lines. Write it only when the subject cannot say why the change exists. State the why, not the what.
