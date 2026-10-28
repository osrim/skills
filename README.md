# skills

Agent skills that I use every day.

| Skill | What it does |
| --- | --- |
| [`commit-message`](skills/commit-message/SKILL.md) | Writes a commit message for the staged changes. |
| [`pr-description`](skills/pr-description/SKILL.md) | Writes a PR description for the current branch, focused on why the change exists. |
| [`code-review-inline`](skills/code-review-inline/SKILL.md) | Reviews a GitHub PR against the repository's standards and its spec, then publishes inline comments. |

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

## Optional: the-writing-whip

The `commit-message` and `pr-description` skills use
[`the-writing-whip`](https://gist.github.com/ossa-ma/dae6f9571534f3fbd1266a384be00e11)
when it's installed. Without it, they use a short plain-writing rule. To
install it:

```sh
npx niksi add https://gist.github.com/ossa-ma/dae6f9571534f3fbd1266a384be00e11
```

## License

[MIT](LICENSE)
