---
name: inline-review
description: Trigger this when the user asks for an inline review of a PR.
argument-hint: "[pr-url|number] [draft] [spec=<path|url|issue>]"
---

# Code review with inline comments

Arguments: `$ARGUMENTS`

- PR URL or number. Empty: the PR for the current branch (`gh pr view`).
- `draft`: return the comments and publish nothing.
- `spec=<path|url|issue>`: the spec source. It replaces spec discovery.

Without `draft`, the user's request authorizes publishing. Publish after verification, with no further confirmation.

## 1. Pin the review

Fetch the PR's description, base and head SHAs, commits, changed files, checks, and existing reviews and comments. Review the exact head; use an isolated checkout when the working tree has uncommitted changes. Pin one comparison for both reviewers: `git diff <base>...<head>` and `git log <base>..<head> --oneline`. Confirm the diff is non-empty.

## 2. Gather sources

**Spec:** the `spec=` argument; else issues referenced in the PR description and commits, linked PRs, and matching files under `docs/`, `specs/` or `.scratch/`. Record the source. With no spec, ask for one while the Standards review runs; if none arrives, skip the Spec axis.

**Standards:** applicable `AGENTS.md`, `CLAUDE.md`, contribution guides, and the docs they reference. Record each path and rule.

## 3. Run two independent reviewers

Spawn both in parallel with the pinned SHAs, diff command, commit list, changed files, and their sources. Reviewers report; only you publish. Each report stays under 400 words and gives every finding a path, line or range, priority, evidence, and requested fix.

- **Standards reviewer:** every documented-standard breach, citing source and rule, plus Fowler code smells in the introduced code (Mysterious Name, Duplicated Code, Feature Envy, Data Clumps, Primitive Obsession, Repeated Switches, Shotgun Surgery, Divergent Change, Speculative Generality, Message Chains, Middle Man, Refused Bequest), quoting the hunk. Repo standards override smells, and smells are judgement calls. Scope: issues the diff introduces that tooling does not enforce.
- **Spec reviewer:** missing or partial requirements, unrequested scope, and incorrect implementations, each quoting the spec. Follow callers, config, and deployment behavior where they decide whether a requirement works.

## 4. Verify

For each finding, confirm the cited rule or spec, the code path, the affected case, and the consequence, using the narrowest check or existing CI evidence. Deployment findings need evidence from the service lifecycle, pipeline, or plan; a rollout prerequisite the PR already states is not an omission. Settle unsupported claims with the reviewer or drop them. Keep the two axes separate and unranked against each other.

Priority by impact: P0 blocker, P1 urgent correctness or operational failure, P2 normal fix, P3 minor improvement or documented convention. A smell alone is P2 or lower.

## 5. Write comments

Match the two bodies in [the reference review](references/example-review.md):

```text
[P2] Imperative title naming the fix

Standards: <source and rule>. <changed code and the concrete breach>. <requested fix>.
```

Spec comments start `Spec:`, quote the requirement, show the actual behavior, state the consequence, and ask for the smallest sufficient fix. Smell comments start `Standards: Possible <smell name>.` The first line is plain text. Each comment stands alone on its hunk and names repository paths and symbols.

## 6. Publish

With `draft`, return the comments with their anchors and stop here.

1. Drop findings that existing inline comments already cover (read every page). Re-read the head SHA; if it moved, review the new diff and revalidate the anchors.
2. Anchor each finding to the smallest changed range: head line numbers with `side: RIGHT` for added lines, `side: LEFT` for removed lines. A finding with no valid anchor goes in the review body, marked as such.
3. Submit one review: `event: COMMENT`, the pinned `commit_id`, and one `comments` item per finding (`path`, `line`, `side`, `body`, plus `start_line` and `start_side` for a range). Use a GitHub API tool, or write the payload to a JSON file and run `gh api repos/<owner>/<repo>/pulls/<number>/reviews --method POST --input <file>`. Review body: `🤖 <your model display name>`, then notes on unanchored findings or a skipped axis.
4. Fetch the review's comments and check each body, path, anchor, and URL. Older API responses give `position` and `diff_hunk` in place of `line`. After a timeout, list existing reviews before a retry, so a retry never duplicates.

With no new findings, publish nothing and say so. On failure, report what was published and what remains, and keep the prepared comments.

## 7. Report

Give the comment count, then `## Standards` and `## Spec`. List each finding with its link and a one-line summary, and end each axis with its finding count and worst issue. A skipped axis states why. Add the checks you ran and their limits.
