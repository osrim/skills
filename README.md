# skills

Agent skills that I use every day.

## PR workflow

| Skill | What it does |
| --- | --- |
| [`ship`](skills/ship/SKILL.md) | Runs the whole flow below for an issue, a milestone, or a project: implement, settle open decisions with you, review on another provider, address every comment once review bots finish, restack, and fix failing checks. |
| [`stack-implement`](skills/stack-implement/SKILL.md) | Implements a list of issues as stacked PRs, one subagent and worktree per issue. Each subagent builds test-first, opens the PR, and self-reviews it with `pr-review draft`. |
| [`pr-review`](skills/pr-review/SKILL.md) | Reviews a GitHub or Gitea PR for standards, spec, and correctness in three sub-agents, then publishes inline comments. Based on Matt Pocock's [`code-review`](https://github.com/mattpocock/skills/blob/main/skills/engineering/code-review/SKILL.md). |
| [`address-review`](skills/address-review/SKILL.md) | Works through a PR's open review comments from models, bots, and people: fixes what holds up and replies to each thread, signed with the model. |
| [`forge`](skills/forge/SKILL.md) | Picks the forge from the `origin` remote and gives the command for each PR operation: `gh` on GitHub, `tea` on Gitea. Signs Gitea reviews with the review login your instructions name. |

### How they fit

```text
ship
├─ stack-implement   per issue: tdd, open PR, pr-review draft (self review)
├─ pr-review         public review by a model from another provider
└─ address-review    fixes or answers pr-review, review bots, and people
```

Every commit goes through `commit-message`, every PR body through
`pr-description`, and every PR command through `forge`.

Each skill also runs on its own. Name your cross-provider reviewer in your
`AGENTS.md` or `CLAUDE.md`, or pass `reviewer=` to `ship`. On Gitea, name
the `tea` login that signs reviews there too, such as `review-bot`.

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
| [`tdd`](https://github.com/mattpocock/skills/tree/main/skills/engineering/tdd) | `stack-implement`, which builds each issue with it, and `ship`, which runs `stack-implement`. | Yes |
| [`gh`](https://cli.github.com) on GitHub, [`tea`](https://gitea.com/gitea/tea) 0.16 or later on Gitea | `forge`, which every skill in the PR workflow runs for its PR commands. | Yes |
| [`gh stack`](https://github.com/github/gh-stack) | On GitHub, `stack-implement`, which links the PRs into a GitHub stack that merges into the base branch, and `ship`, which runs `stack-implement`. Without it, `stack-implement` reports the link command for you to run. | No |
| [`the-writing-whip`](https://gist.github.com/ossa-ma/dae6f9571534f3fbd1266a384be00e11) | `commit-message` and `pr-description`. Without it, they use a short plain-writing rule. | No |

## Release

Push `main`, then create a GitHub release. The command tags `main` on GitHub
and writes release notes from the commits since the last tag:

```sh
gh release create vX.Y.Z --target main --generate-notes
```

## License

[MIT](LICENSE)
