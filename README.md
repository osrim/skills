# skills

Agent skills that I use every day.

## PR workflow

| Skill | What it does |
| --- | --- |
| [`ship`](skills/ship/SKILL.md) | Runs the whole flow below for a spec, a ticket, a milestone, or a project: implement, review on another provider beside the self review, settle open decisions with you, address every comment once review bots finish, have the reviewer check the final head, restack, report, and fix failing checks. |
| [`stack-implement`](skills/stack-implement/SKILL.md) | Runs Matt Pocock's `implement` once per ticket and lands the tickets as stacked PRs: one stack for a spec or a ticket list, one per spec for a milestone or project, and one worktree per ticket, built in the main thread when there is one ticket and by one subagent per ticket otherwise. Each builder builds test-first, opens the PR, and self-reviews it with `pr-review draft`. Leaves out unticketed specs, wayfinder maps, and tickets not triaged for an agent, and names the skill to run first. |
| [`pr-review`](skills/pr-review/SKILL.md) | Reviews a GitHub PR for standards, spec, and correctness in three sub-agents, then publishes inline comments. Based on Matt Pocock's [`code-review`](https://github.com/mattpocock/skills/blob/main/skills/engineering/code-review/SKILL.md). |
| [`address-review`](skills/address-review/SKILL.md) | Works through a PR's open review comments from models, bots, and people: fixes what holds up and replies to each thread, signed with the model. |

### How they fit

`ship` covers the build and review steps of Matt Pocock's main flow: his
`implement` and `code-review` for each ticket, plus a cross-provider review,
review comments, and CI. It uses his words: a spec is what `to-spec`
publishes, and a ticket is what `to-tickets` splits it into.

```text
grill-with-docs → to-spec → to-tickets → ship
```

```text
ship
├─ stack-implement   per ticket: tdd, open PR, pr-review draft (self review)
├─ pr-review         public review by a model from another provider, against
│                    the ticket and its parent spec, beside the self review
│                    and once more on the final head
└─ address-review    fixes or answers pr-review, review bots, and people
```

Every commit goes through `commit-message`, and every PR body through
`pr-description`.

Each skill also runs on its own. Name your cross-provider reviewer in your
`AGENTS.md` or `CLAUDE.md`, or pass `reviewer=` to `ship`.

## Writing

| Skill | What it does |
| --- | --- |
| [`commit-message`](skills/commit-message/SKILL.md) | Writes a commit message for the staged changes. |
| [`pr-description`](skills/pr-description/SKILL.md) | Writes a PR description for the current branch, focused on why the change exists. |

## Install

Install with [niksi](https://github.com/osrim/niksi):

```sh
npx niksi add osrim/skills
```

To pin a release, add a tag: `npx niksi add osrim/skills@v0.2.0`. If you
installed niksi with Homebrew or `npm install -g`, use `nik` in place of
`npx niksi`.

Or install with `npx skills`:

```sh
npx skills add osrim/skills
```

## Dependencies

Some skills here run skills from other sources. Install each dependency
alongside the skills that use it:

| Dependency | Used by | Required |
| --- | --- | --- |
| [`setup-matt-pocock-skills`](https://github.com/mattpocock/skills/tree/main/skills/engineering/setup-matt-pocock-skills) | `stack-implement`, `pr-review`, and `ship`, which read and update issues with the tracker workflow it configures. | Yes |
| [`tdd`](https://github.com/mattpocock/skills/tree/main/skills/engineering/tdd) | `stack-implement`, which builds each ticket with it, and `ship`, which runs `stack-implement`. | Yes |
| [`gh stack`](https://github.com/github/gh-stack) | `stack-implement`, which links the PRs into a GitHub stack that merges into the base branch, and `ship`, which runs `stack-implement`. Without it, `stack-implement` reports the link command for you to run. | No |

## Release

Push `main`, then create a GitHub release. The command tags `main` on GitHub
and writes release notes from the commits since the last tag:

```sh
gh release create vX.Y.Z --target main --generate-notes
```

## License

[MIT](LICENSE)
