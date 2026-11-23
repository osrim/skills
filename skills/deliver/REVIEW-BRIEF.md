# Review brief

Send this text with its slots filled to a child on the reviewer's provider, model, and effort, started with the harness's cross-provider delegation. Step 3 sends it without the verify line; step 5 sends it with that line.

```text
Review <PR URL>. Its worktree, <path>, is yours to read; change nothing in it.

Call the Skill tool with `pr-review`, passing the PR and these spec sources:
<one spec=<notes file> for the PR's spec and for each ticket merged into it>.
<Step 5 only: pass verify=<head SHA of the earlier review> as well.>
Those files hold the issues' text: make no tracker calls for them. Publish
the review.

The checks that need a running app run once per PR, later, owned by
<owner from preflight.md>. Drop a finding that says they are missing.

Report: the review URL, the head SHA reviewed, and the finding count per axis
and per priority.
```
