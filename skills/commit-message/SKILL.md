---
name: commit-message
description: Writes a commit message for the staged changes, in the repository's commit style. Use when the user asks for a commit message, or before you run git commit. Read-only.
allowed-tools: Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git status:*), Bash(git branch:*)
---

# Commit message

Write a commit message for the staged changes. This skill writes the text only.

## Context

- Staged diff: !`git diff --cached`
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -20`

## Process

1. If the staged diff is empty, say so and stop.
2. If the `the-writing-whip` skill is available, load it before you write any text. Else, write plain and short, with concrete words.
3. Find the intent of the change. If the diff holds more than one concern, propose a split in one line. Then draft one message for the full diff.
4. Match the format, scopes, and casing of the recent commits. If the history has no clear style, use `type(scope): subject`.
5. If the branch name holds a ticket ID (for example `ABC-123`), add a `Refs: ABC-123` trailer.
6. Read the draft as a teammate who runs `git log` a year from now. Cut every word that the diff already says.

## Output

If a person asked for the message, put only the message in one fenced code block. If you pass it to `git commit`, use the plain text.

The subject says what changed, in the imperative mood, in 72 characters or fewer. The body is optional. Use up to 3 lines on why, only when the subject cannot carry it.

Example:

```
fix(auth): refresh the token before it expires

Requests failed when the token expired between the check and the call.
```
