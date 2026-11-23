# skills

Agent skills that I use every day.

## PR workflow

| Skill | What it does |
| --- | --- |
| [`deliver`](skills/deliver/SKILL.md) | Runs the whole flow below for a spec, a ticket, a milestone, or a project, without stopping for questions: preflight, build every unblocked ticket at once in its own worktree (one PR per spec, or with `stack`, one stacked PR per ticket), review on another model family, address every comment once review bots finish, have the reviewer verify the fixes, check the running app, report with the decisions it took, and fix failing checks. |
| [`pr-review`](skills/pr-review/SKILL.md) | Reviews a GitHub PR for standards, spec, and correctness in three sub-agents, then publishes inline comments. Based on Matt Pocock's [`code-review`](https://github.com/mattpocock/skills/blob/main/skills/engineering/code-review/SKILL.md). |
| [`address-review`](skills/address-review/SKILL.md) | Works through a PR's open review comments from models, bots, and people: fixes what holds up and replies to each thread, signed with the model. |

### How they fit

`deliver` covers the build and review steps of Matt Pocock's main flow: his
`implement-spec` for each spec, plus a cross-family review, review comments,
and CI. With `stack`, each ticket gets its own PR on top of the one below it,
so you review and merge a spec layer by layer. It uses his words: a spec is
what `to-spec` publishes, and a ticket is what `to-tickets` splits it into.

```text
grill-with-docs → to-spec → to-tickets → deliver
```

```text
deliver
├─ prototype         a mockup of each visual ticket, shown before its build
├─ tdd               per ticket, in parallel along the blocking edges
├─ pr-review         public review by a model from another family, against
│                    the spec and its tickets, then a check of the fixes
└─ address-review    fixes or answers pr-review, review bots, and people
```

Every commit goes through `commit-message`, and every PR body through
`pr-description`.

Each skill also runs on its own. Name your cross-family reviewer and its
fallbacks in your `AGENTS.md` or `CLAUDE.md`, or pass `reviewer=` to `deliver`.

## Writing

| Skill | What it does |
| --- | --- |
| [`commit-message`](skills/commit-message/SKILL.md) | Writes a commit message for the staged changes. |
| [`pr-description`](skills/pr-description/SKILL.md) | Writes a PR description for the current branch: why the change exists, the evidence that it works, and its merge danger. Lists the whole stack on a stacked PR. |

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
| [`setup-matt-pocock-skills`](https://github.com/mattpocock/skills/tree/main/skills/engineering/setup-matt-pocock-skills) | `pr-review` and `deliver`, which read and update issues with the tracker workflow it configures. | Yes |
| [`tdd`](https://github.com/mattpocock/skills/tree/main/skills/engineering/tdd) | `deliver`, which builds each ticket with it. | Yes |
| [`prototype`](https://github.com/mattpocock/skills/tree/main/skills/engineering/prototype) | `deliver`, which mocks up a visual ticket that carries no mockup. | For visual tickets |

## Release

Push `main`, then create a GitHub release. The command tags `main` on GitHub
and writes release notes from the commits since the last tag:

```sh
gh release create vX.Y.Z --target main --generate-notes
```

## License

[MIT](LICENSE)
