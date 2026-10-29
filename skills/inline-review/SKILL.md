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

Fetch the PR's description, base and head SHAs, commits, changed files, checks, and existing reviews and comments. Review the exact head; use an isolated checkout when the working tree has uncommitted changes. Pin one comparison for every reviewer: `git diff <base>...<head>` and `git log <base>..<head> --oneline`. Confirm the diff is non-empty.

## 2. Gather sources

**Spec:** the `spec=` argument; else issues referenced in the PR description and commits, linked PRs, and matching files under `docs/`, `specs/` or `.scratch/`. Record the source. With no spec, ask for one while the Standards review runs; if none arrives, skip the Spec axis.

**Standards:** applicable `AGENTS.md`, `CLAUDE.md`, contribution guides, and the docs they reference. Record each path and rule.

## 3. Run three independent reviewers

Each reviewer is its own sub-agent, so it reads the code fresh and its findings stay independent of yours and of the others. Spawn the three in parallel with the pinned SHAs, diff command, commit list, changed files, and their sources. For several PRs, spawn three per PR. When you cannot spawn sub-agents, tell the user and stop before reviewing. Reviewers report; only you publish. Each report stays under 400 words and gives every finding a path, line or range, priority, evidence, and requested fix. The step is done when every reviewer has returned its report.

- **Standards reviewer:** every documented-standard breach, citing source and rule, plus Fowler code smells in the introduced code (Mysterious Name, Duplicated Code, Feature Envy, Data Clumps, Primitive Obsession, Repeated Switches, Shotgun Surgery, Divergent Change, Speculative Generality, Message Chains, Middle Man, Refused Bequest), quoting the hunk. Repo standards override smells, and smells are judgement calls. Scope: issues the diff introduces that tooling does not enforce.
- **Spec reviewer:** missing or partial requirements, unrequested scope, and incorrect implementations, each quoting the spec. Follow callers, config, and deployment behavior where they decide whether a requirement works.
- **Correctness reviewer:** reads the changed code and the code it connects to (callers, callees, tests), as the developer who gets paged when it breaks. It looks for:
  - **Bug:** a wrong result or crash on an edge case: empty, zero, last item, escaped or encoded forms, a caller the change breaks, a fix applied in one caller while the shared function stays broken.
  - **Risk:** missing checks on untrusted input, injection, leaked secrets, swallowed errors, data loss, copied third-party code or data shipped without the notice its license requires.
  - **Load:** work that grows faster than the input: regex backtracking, rescans from every offset, a query or allocation per item, an input-sized array spread into a call's arguments. Judge against the expected load in the repo, and against hostile input wherever the code reads untrusted data.
  - **Missing test:** risky new logic (a parser, a security check, a bug fix) with no test that fails when it breaks.

  Every finding names a concrete case: this input or situation gives this wrong result. No case, no finding.

## 4. Verify

For each finding, confirm the cited rule or spec, the code path, the affected case, and the consequence, using the narrowest check or existing CI evidence. Reproduce a Correctness case with a test or a one-off script when that is cheap. Deployment findings need evidence from the service lifecycle, pipeline, or plan; a rollout prerequisite the PR already states is not an omission. Settle unsupported claims with the reviewer or drop them. Keep the three axes separate and unranked against each other.

Priority by impact: P0 blocker, P1 urgent correctness or operational failure, P2 normal fix, P3 minor improvement or documented convention. A smell alone is P2 or lower.

## 5. Write comments

Match the bodies in [the reference review](references/example-review.md):

```text
[P2] Imperative title naming the fix

Standards: <source and rule>. <changed code and the concrete breach>. <requested fix>.
```

Spec comments start `Spec:`, quote the requirement, show the actual behavior, state the consequence, and ask for the smallest sufficient fix. Correctness comments start `Correctness:`, give the concrete case and its wrong result, and ask for the smallest fix, preferring one that deletes code. Smell comments start `Standards: Possible <smell name>.` The first line is plain text. Each comment stands alone on its hunk and names repository paths and symbols.

## 6. Publish

With `draft`, return the comments with their anchors and stop here.

1. Drop findings that existing inline comments already cover (read every page). Re-read the head SHA; if it moved, review the new diff and revalidate the anchors.
2. Anchor each finding to the smallest changed range: head line numbers with `side: RIGHT` for added lines, `side: LEFT` for removed lines. A finding with no valid anchor goes in the review body, marked as such.
3. Submit one review: `event: COMMENT`, the pinned `commit_id`, and one `comments` item per finding (`path`, `line`, `side`, `body`, plus `start_line` and `start_side` for a range). Use a GitHub API tool, or write the payload to a JSON file and run `gh api repos/<owner>/<repo>/pulls/<number>/reviews --method POST --input <file>`. Review body: `🤖 <your model display name>`, then notes on unanchored findings or a skipped axis.
4. Fetch the review's comments and check each body, path, anchor, and URL. Older API responses give `position` and `diff_hunk` in place of `line`. After a timeout, list existing reviews before a retry, so a retry never duplicates.

With no new findings, submit the review with no `comments` and this body:

```text
🤖 <your model display name>

No findings on <short head sha>.
```

Write `No new findings` when existing comments already cover every finding, and name a skipped axis in place of its review. Skip the review when an earlier one on the same head SHA already reports no findings.

On failure, report what was published and what remains, and keep the prepared comments.

## 7. Report

Give the comment count, then `## Standards`, `## Spec`, and `## Correctness`. List each finding with its link and a one-line summary, and end each axis with its finding count and worst issue. A skipped axis states why. Add the checks you ran and their limits.
