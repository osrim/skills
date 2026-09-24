# skills

Agent skills that I use every day.

| Skill | What it does |
| --- | --- |
| [`draft-commit`](skills/draft-commit/SKILL.md) | Drafts a commit message for the staged changes. |
| [`draft-pr`](skills/draft-pr/SKILL.md) | Drafts a PR description for the current branch, in 120 words or fewer. |

Both skills are read-only. They write text and do not run git writes.

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

## Requirements

Both skills load [`the-writing-whip`](https://gist.github.com/ossa-ma/dae6f9571534f3fbd1266a384be00e11). Install it separately:

```sh
npx niksi add https://gist.github.com/ossa-ma/dae6f9571534f3fbd1266a384be00e11
```

## License

[MIT](LICENSE)
