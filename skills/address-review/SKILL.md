---
name: address-review
description: Trigger this when the user asks to check a PR's review comments, or to fix and answer them.
argument-hint: "[pr-url|number] [draft]"
---

# Address PR review comments

Arguments: `$ARGUMENTS`

- PR URL or number. Empty: the PR for the current branch (`gh pr view`).
- `draft`: make the fixes locally and return the replies. Commit, push, and post nothing.

Without `draft`, the user's request authorizes committing and pushing fixes to the PR branch and posting replies, with no further confirmation.

## Signature

The signature opens every reply this skill posts and every `pr-review` review body:

```text
🤖 <your model display name>

<reply>
```

Step 1 reads a thread whose latest comment is signed as already answered, so a reply without the signature gets answered again on the next run.

## 1. Collect the work list

Check out the PR's head branch, level with its remote. Use an isolated checkout when the working tree has uncommitted changes. Fetch, reading every page:

- **Threads:** unresolved review threads from GraphQL `pullRequest.reviewThreads` (`isResolved`, `isOutdated`, `path`, `line`, and each comment's `databaseId`, author, and body).
- **Review bodies:** reviews whose body carries requested changes or findings with no line anchor: the unanchored findings in a `pr-review` body, and findings folded into `<details>` blocks, such as CodeRabbit's nitpick and outside-diff comments.

A review bot is still reviewing when it has a pending check or an in-progress comment on the head SHA. Wait until each one has posted or its check has completed, up to 15 minutes, using the harness's PR watch when it has one, otherwise checking every minute, and record each bot that timed out. Then collect.

Skip a thread whose latest comment opens with the signature (an earlier answer), and a review body that a later signed PR comment already answers. Everything else is the work list. With an empty work list, report that and stop.

## 2. Decide each comment

Read the commented code at the PR head and the code it connects to. Check each claim yourself: the reviewer's priority and confidence are not evidence. Give every comment on the work list exactly one outcome. Comments that raise the same issue share one fix, and each gets its own reply.

- **Fix:** the comment holds up. Make the smallest change that resolves it, in the repo's standards. An outdated thread that a later commit already fixed is a Fix with that commit.
- **Decline:** the comment is wrong, out of scope, or conflicts with the spec or a repo rule. Record the evidence: code path, rule, spec line, or check output.
- **Answer:** the comment asks a question. Record the answer and its evidence.

When the right fix needs a decision that belongs to the user (behavior change, public API, schema, security), ask, grouping the questions, and keep working on the rest. The step is done when every comment on the work list has one outcome.

## 3. Verify and push

Run the narrowest checks that cover the fixes, plus the checks the repo documents for this kind of change. Fix failures your changes cause; report the rest.

With `draft`, leave the fixes uncommitted and go to step 4.

Commit the fixes, calling the Skill tool with `commit-message` before each commit, and record which commit fixes which comment. Push to the PR branch with a plain `git push`. If the commit fails on signing, keep the fixes staged, report the exact error once, and stop before replying. If the push is rejected, stop and report: the remote moved, and integrating it is the user's call. The step is done when every Fix maps to a pushed commit.

## 4. Reply

Every reply opens with the signature.

- **Fix:** `Fixed in <short sha>: <what changed>.`
- **Decline:** why, citing the evidence.
- **Answer:** the answer, citing the evidence.

With `draft`, return each reply with its thread link and stop here.

1. Reply to each thread on its first comment: `gh api repos/<owner>/<repo>/pulls/<number>/comments/<first comment databaseId>/replies --method POST -f body=<body>`.
2. Answer review-body findings in one PR comment (`gh pr comment`), quoting each finding above its reply.
3. Leave threads unresolved; the reviewer resolves them.
4. Fetch each thread and check that its reply landed once, with the right body. After a timeout, list the thread's comments before a retry, so a retry never duplicates.

On failure, report what was posted and what remains, and keep the prepared replies.

## 5. Report

Give the count per outcome, then each comment's link, outcome, and a one-line summary. Add the checks you ran and their results, the pushed commits, and anything unposted or waiting on the user.
