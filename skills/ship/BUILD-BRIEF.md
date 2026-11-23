# Build brief

Send this text with its slots filled, and nothing else. It carries one ticket, so the builder gives the whole of its attention to it. As the builder of a one-ticket plan, follow it yourself and ask the user in place of reporting.

```text
Build <ticket id> (<ticket link>), a ticket of <spec link, or "no spec">.

Read first: <notes>/tickets/<ticket id>.md, <notes>/specs/<spec id>.md,
<notes>/checks.md, and <notes>/looks/<ticket id>.md when it exists. They hold
the settled decisions and the approved look. Fetch nothing they already hold.

Worktree: <path>, on branch <ticket branch>, cut from <integration branch>.
All work happens there. <Earlier commits on the branch, when it was reused.>

1. Move the ticket to In Progress with the tracker's workflow, and state its
   title.
2. Call the Skill tool with `tdd` and build the ticket. The seams the ticket
   or its spec names are confirmed. Where they name none, choose the seams
   and list them in your report.
3. Run the covering checks from checks.md as you go, and once after your last
   change: the typecheck, the lint, and the tests that exercise the files you
   changed. The full suite belongs to CI. The checks that need a running app
   (the browser suite, a UI check) belong to the orchestrator.
4. Before every commit, call the Skill tool with `commit-message`. Commit on
   your branch. When a commit fails on signing, keep the changes staged,
   retry once, and report the exact error.
5. Merge the tip of <integration branch> into your branch, and run the
   covering checks for the files that merge touched.

You are done at step 5. The orchestrator merges your branch, pushes, and
opens the PR.

Decisions: your report is your only channel to the user. When the ticket and
its spec leave a choice open, take the most conservative option and go on.
When that choice is about schema, auth, or public API, commit what you have,
stop, and report the question with your recommendation.

You may start sub-agents, each with a fresh context window.

Report: the head commit, each covering check with its result (as evidence:
the test that failed before your change and passes after, by file and name),
each decision as "question, option taken, what the alternative changes", the
seams you chose, and anything left undone.
```
