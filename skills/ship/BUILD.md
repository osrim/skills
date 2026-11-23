# Build

Step 2 of `ship`: Matt Pocock's `implement-spec`, run for every spec in the source. Each spec's tickets land on one **integration branch**. The tickets are a task graph: the **frontier** is every ticket whose blockers are merged, and the whole frontier builds at once.

Every ticket gets its own worktree and a **builder**: a sub-agent that builds it test-first and commits, in the background where the harness can. You merge its branch into the integration branch and push. With one ticket in the plan, you are the builder, so the user watches the work and can steer it.

## Vocabulary

The words are Matt Pocock's. Sort every issue the source brings in, in this order:

- **Wayfinder issue**: an issue labelled `wayfinder:map` (a map) or `wayfinder:<type>` (a decision ticket). It holds decisions, not build work, even when a map has sub-issues; a cleared map goes to `to-spec`.
- **Spec**: what `to-spec` publishes. An issue with sub-issues, or one whose body has `to-spec`'s `## Problem Statement` and `## User Stories` sections. A spec with no sub-issues is **unticketed**: `to-tickets` has not split it yet, so it has nothing to build.
- **Ticket**: every other issue. A sub-issue of a spec is what `to-tickets` publishes; a standalone one may come from `triage`. Its **blocking edges** name the tickets that gate it.
- **Triage state**: the ticket's triage role, read through the repo's triage-label mapping. A ticket is **buildable** when its state is `ready-for-agent` or it has none; `ready-for-human`, `needs-triage`, `needs-info`, and `wontfix` are not.
- **Visual ticket**: a ticket that changes what a screen looks like: layout, a new component, styling, or on-screen copy.
- **Layer**: with `stack`, one ticket's PR, based on the layer below it. The layers of one integration branch form its **stack**, in merge order.

## 1. Plan

The issue tracker should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills`. Use its workflow for every issue read and write below.

Fetch every issue in the source, reading every page, and drop closed issues. Read each issue's parent, labels, and blocking edges, and each spec's sub-issues. Leave out, wherever it appears in the source or among a spec's sub-issues, with its reason: every wayfinder issue (clear the map, then run `/to-spec`), every unticketed spec (run `/to-tickets` on it), and every ticket that is not buildable (its triage state). Replace each remaining spec with its open sub-issues until no item is a spec, and keep each ticket once. Then split the tickets into integration branches:

- The tickets of one parent spec share one integration branch.
- Tickets with no parent spec share an integration branch only when blocking edges join them. Every other one gets its own.

A blocker outside the plan is cleared when its PR is merged into the base branch, or, when it has no PR (a decision or a prototype), when the tracker shows it closed as done. For each blocker that is not cleared, leave out the blocked ticket and every ticket that depends on it, directly or through others. When no ticket is left, stop.

Name each integration branch in the repo's convention, falling back to `feat/<spec or ticket id in lowercase>`, and each ticket branch `<integration branch>-<ticket id in lowercase>`. Ticket branches stay local until a merge pushes them as layers.

Write the run notes, which every later brief points to in place of restating them:

- `tickets/<ticket id>.md` and `specs/<spec id>.md`: each issue's title, link, and body, as fetched.
- `checks.md`, unless the notes already hold one: the repo's exact commands for the typecheck, the lint, one test file, the full suite, the browser suite, and the dev server, taken from its docs and scripts, and whether CI runs the full suite on a push.
- `decisions.md`: empty.

For each visual ticket, get its **look** before any build: the mockup or screenshot the ticket carries, or else one sub-agent per ticket, started with a fresh context window, that calls the Skill tool with `prototype` and saves a mockup image under `looks/` in the notes. Show every look to the user in one message, each under its ticket ID. A visual ticket joins the frontier when the user approves its look; save what they said to `looks/<ticket id>.md`.

Post the plan: for each integration branch, its base and a list of ticket, parent spec (or none), ticket branch, blockers, and whether it waits on a look. Under it, list each spec from the source that no planned ticket has as its parent, each issue left out with its reason (its open blocker, its triage state, or the skill to run first), and the notes directory. The step is done when every open issue from the source, or from expanding a spec, appears exactly once: as a planned ticket, as a spec, or as left out with its reason. The plan is a record, so go straight on to step 2 while the looks are made and answered.

## 2. Build the frontier

An earlier run may have left work behind. Reuse an integration branch that exists, with its worktree and its PRs. A ticket whose merge commit is on it (`git log --merges --grep '<ticket id>' origin/<integration branch>`) is built; with `stack`, a ticket whose layer branch exists on `origin` is built, and the integration branch restarts from the top layer. A ticket branch that exists without a merge keeps its commits: reuse it and tell the builder about them.

Fetch `origin`, then give each integration branch a worktree on a new branch from `origin/<base>`: the repo's worktree tool when it has one, otherwise `git worktree add -b <branch> <path> origin/<base>`.

A ticket is on the frontier when it is not built, not parked, not waiting on a look, and every blocker it has in the plan is merged into its integration branch. Repeat until the frontier is empty and no builder is running:

1. **Start** every ticket on the frontier: create its worktree on its ticket branch from the tip of its integration branch, and dispatch a builder with a fresh context window and [BUILD-BRIEF.md](BUILD-BRIEF.md). An integration branch with one ticket has no ticket branch: its builder works in the integration worktree.
2. **Wait** by ending the turn.
3. **Merge** each builder that reports a head commit, one merge at a time, after saving its checks to `evidence/<ticket id>.md` in the notes, in the integration worktree: `git merge --no-ff <ticket branch> -m "Merge <ticket id>: <ticket title>"`. On a conflict, dispatch a merger sub-agent with a fresh context window, pointed at the two tickets' notes, to resolve it in favour of what the tickets and the spec settled, run the covering checks for the conflicted files, and commit. Then push with a plain `git push`: the integration branch, or with `stack`, the merge commit as the ticket's layer (`git push origin HEAD:<ticket branch>`). Remove the ticket's worktree and its local branch.
4. **Open the PR** with `gh pr create --draft`, after calling the Skill tool with `pr-description`, passing the PR's base and its tickets' evidence files:
   - One PR per integration branch, after its first merge, with `--base <base>`.
   - With `stack`, one PR per layer, after its merge, with `--head <ticket branch> --base <layer below>`, or `--base <base>` for the first layer. Its diff is exactly its ticket.
5. **Record** each decision the builder reported, and park the ticket of each question it stopped on. A builder that ends without a head commit fails its ticket: the ticket and every ticket it blocks leave the frontier, with the reason. When it reports a commit that failed on signing, tell the user once to unlock the signer, then commit its staged changes yourself.

Merge order is stack order, and it respects the blocking edges, since a ticket only starts after its blockers merge.

When the loop ends, finish each integration branch that has a merge:

1. When `checks.md` says CI does not run the full suite, run it once in the integration worktree, one integration branch at a time, and dispatch one fresh builder to fix what fails. With `stack`, the builder fixes each failure on the lowest layer that causes it.
2. Update every PR body with `pr-description`: each closes its merged tickets, the PR that merges last closes the spec when all of its tickets are merged, and each records the decisions taken for its tickets. With `stack`, every layer's body lists the whole stack.
3. Run `gh pr ready <pr>` on each PR, and post each PR link with its count of merged and planned tickets.
4. With `stack`, give each layer a worktree on its branch for the later steps (`git worktree add -B <ticket branch> <path> origin/<ticket branch>`).

The step is done when every planned ticket is merged, parked, failed, or still waiting on a look, and every integration branch with a merge has its ready PR or ready stack. Remove the ticket worktrees and ticket branches this run created that hold no commits.
