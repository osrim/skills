# skills

Agent skills that I use every day.

| Skill | What it does |
| --- | --- |
| [`commit-message`](skills/commit-message/SKILL.md) | Writes a commit message for the staged changes. |
| [`pr-description`](skills/pr-description/SKILL.md) | Writes a PR description for the current branch, focused on why the change exists. |
| [`inline-review`](skills/inline-review/SKILL.md) | Reviews a GitHub PR for standards, spec, and correctness in three sub-agents, then publishes inline comments. |
| [`address-review`](skills/address-review/SKILL.md) | Works through a PR's open review comments: fixes what holds up and replies to each thread, signed with the model. |
| [`stack-implement`](skills/stack-implement/SKILL.md) | Implements a list of issues as stacked PRs, one subagent and worktree per issue, each running the separately installed `implement` skill. |

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
| [`implement`](https://github.com/mattpocock/skills/tree/main/skills/engineering/implement) | `stack-implement`, which runs it for each issue. | Yes |
| [`the-writing-whip`](https://gist.github.com/ossa-ma/dae6f9571534f3fbd1266a384be00e11) | `commit-message` and `pr-description`. Without it, they use a short plain-writing rule. | No |

## Release

Push `main`, then create a GitHub release. The command tags `main` on GitHub
and writes release notes from the commits since the last tag:

```sh
gh release create vX.Y.Z --target main --generate-notes
```

## License

[MIT](LICENSE)
