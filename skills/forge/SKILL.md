---
name: forge
description: Trigger this when the user asks to run a PR command on GitHub or Gitea, or before you run `gh` or `tea` on a pull request or issue.
---

# Forge

The **forge** hosts the repository's PRs, reviews, and checks. Skills name each PR command as an operation in bold, such as **Find PR**; run the command in your forge's column of the table below.

## Pick the forge

Take the host from `git remote get-url origin`:

- `github.com`: **GitHub**. Run `gh`.
- A host that `tea login list` has a login for: **Gitea**. Run `tea` 0.16 or later, and `tea api` for what `tea` has no subcommand for.
- Neither: stop and report the remote.

`tea` reads options only before the positional arguments: write `tea <command> <options> <arguments>`, as every command below does.

Both CLIs read the owner and repository from `origin`, and both fill `{owner}` and `{repo}` in an API endpoint. Quote an endpoint that holds `?` or `&`.

## Logins

On GitHub, every command runs as the active `gh` account.

On Gitea, the **review operations** (**Submit review**, **Approve**, **Request changes**) run as the **review login**: the `tea` login the user's instructions name for reviews, such as `review-bot`, passed as `--login <review login>`. Every other operation, replies and comments included, runs as the default login. When the instructions name no review login, reviews use the default login too.

## Operations

`<n>` is the PR number. Read every page of a list: `gh api --paginate`, or `?page=<p>&limit=50` on a Gitea endpoint.

| Operation | GitHub | Gitea |
| --- | --- | --- |
| **Find PR** for a branch | `gh pr view <branch> --json number,url,state,baseRefName` | `tea pr list --state open --fields index,head,base,url --output json`, then the entry whose `head` is the branch |
| **PR state** | `gh pr view <n> --json body,baseRefName,baseRefOid,headRefOid,commits,files,mergeable`; `mergeable` is `CONFLICTING` on a conflict | `tea api 'repos/{owner}/{repo}/pulls/<n>'` gives `body`, `base.ref`, `base.sha`, `head.sha`, and `mergeable`, which is `false` on a conflict; commits and files at `.../pulls/<n>/commits` and `.../pulls/<n>/files` |
| **Open PR** | `gh pr create --base <base> --head <branch> --title <title> --body-file <file>` | `tea pr create --base <base> --head <branch> --title <title> --description-file <file>` |
| **Edit body** | `gh pr edit <n> --body-file <file>` | `tea pr edit --description-file <file> <n>` |
| **Set base** | `gh pr edit <n> --base <base>` | `tea api -X PATCH -f base=<base> 'repos/{owner}/{repo}/pulls/<n>'` |
| **Checks** on the PR head | `gh pr checks <n> --required` | `tea api 'repos/{owner}/{repo}/commits/<head sha>/status'`: `state` is `success`, `pending`, or `failure`, with one entry per check in `statuses` |
| **Check log** | `gh run view <run id> --log-failed` | `tea actions runs list --branch <branch>`, then `tea actions runs logs <run id>` |
| **Reviews** | `gh api 'repos/{owner}/{repo}/pulls/<n>/reviews'` | `tea api 'repos/{owner}/{repo}/pulls/<n>/reviews'` |
| **Review threads** | `gh api graphql` on `pullRequest.reviewThreads`: `isResolved`, `isOutdated`, `path`, `line`, and each comment's `databaseId`, author, and body | `tea pr review-comments --fields id,path,line,body,reviewer,resolver,created --output json <n>`. A thread is the comments on one `path` and `line`, oldest first, and stays unresolved while its first comment's `resolver` is empty. Gitea reports no outdated state: compare the code at the head. |
| **Review comments** of one review | `gh api 'repos/{owner}/{repo}/pulls/<n>/reviews/<review id>/comments'`; older responses give `position` and `diff_hunk` in place of `line` | `tea api 'repos/{owner}/{repo}/pulls/<n>/reviews/<review id>/comments'`; `position` is the head line and `original_position` the base line |
| **Submit review** | `gh api 'repos/{owner}/{repo}/pulls/<n>/reviews' --method POST --input <file>` | `tea api --login <review login> -d @<file> 'repos/{owner}/{repo}/pulls/<n>/reviews'` |
| **Reply** in a thread | `gh api 'repos/{owner}/{repo}/pulls/<n>/comments/<first comment id>/replies' --method POST -f body=<body>` | `tea pr reply <n> <first comment id> "<body>"` |
| **Comment** on the PR | `gh pr comment <n> --body-file <file>` | `tea comment <n> "<body>"` |
| **Approve** | `gh pr review <n> --approve --body <body>` | `tea pr approve --login <review login> <n> "<body>"` |
| **Request changes** | `gh pr review <n> --request-changes --body <body>` | `tea pr reject --login <review login> <n> "<findings>"` |
| **Merge** | `gh pr merge <n> --squash` (or `--merge`, `--rebase`) | `tea pr merge --style squash <n>` (or `merge`, `rebase`) |
| **Milestone issues** | `gh issue list --milestone <milestone> --state open --json number,title,url,body` | `tea issues list --milestones <milestone> --state open --output json` |

## Review payload

**Submit review** posts one JSON file: `event: COMMENT`, the pinned `commit_id`, a `body`, and one `comments` item per anchored finding, each with `path` and `body`. The anchor differs:

- **GitHub:** `line` and `side`: the head line with `side: RIGHT` for an added line, the base line with `side: LEFT` for a removed one. A range adds `start_line` and `start_side`.
- **Gitea:** `new_position` for an added line (the head line number) or `old_position` for a removed one (the base line number). Gitea anchors a single line: anchor a range at its last line and name the range in the body.
