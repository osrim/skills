# Address brief

Send this text with its slots filled to a sub-agent on your own model.

```text
Address the review comments on <PR URL>. Work in its worktree, <path>.

Call the Skill tool with `address-review`, passing the PR. It waits for the
review bots still reviewing.

Give a comment the Ask outcome only when its decision is about schema, auth,
or public API. For any other decision that belongs to the user, take the
option you recommend and report it as a decision.

Report: the count per outcome, the pushed commits, the files the fixes
touched, the bots that timed out, each decision as "question, option taken,
what the alternative changes", and the open questions.
```
