# skills

Agent skills that I use every day.

| Skill | What it does |
| --- | --- |
| [`commit-message`](skills/commit-message/SKILL.md) | Writes a commit message for the staged changes. |
| [`pr-description`](skills/pr-description/SKILL.md) | Writes a PR description for the current branch, focused on why the change exists. |

Both skills are read-only. They write text and do not run git writes. Call them with `/commit-message` or `/pr-description`, or let the agent use them when it commits or opens a PR.

## Install

### Using [niksi](https://github.com/osrim/niksi):

```sh
npx niksi add osrim/skills
```

To pin a release, add a tag: `npx niksi add osrim/skills@v1.0.0`.

If you installed niksi with Homebrew or `npm install -g`, use `nik` in place of `npx niksi`.

### Using npx skills

```sh
npx skills add osrim/skills
```

## Optional

Both skills use [`the-writing-whip`](https://gist.github.com/ossa-ma/dae6f9571534f3fbd1266a384be00e11) when it is installed. Without it, they fall back to a short plain-writing rule. To install it:

```sh
npx niksi add https://gist.github.com/ossa-ma/dae6f9571534f3fbd1266a384be00e11
```

## License

[MIT](LICENSE)
