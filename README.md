# tibi-skills

Agent skills from the National Library of Norway, shared for reuse by other organisations.

## Skills

| Skill | What it does |
| --- | --- |
| [`human-code-review`](skills/human-code-review/SKILL.md) | Quizzes the developer on the actual diff before merge, to confirm they understand the change. Flags code that is hard to understand and proposes clearer code. |

## Install

With the [`skills`](https://www.skills.sh/) CLI:

```bash
npx skills add NationalLibraryOfNorway/tibi-skills
```

Or manually: copy `skills/<name>/` into your agent's skills directory (for example `.claude/skills/` or `.agents/skills/`).

## Language

The skills are written in English and reply in the language the developer uses.

## Contributing

Issues and pull requests are welcome. They are read on a best-effort basis, with no response time guaranteed.

## Licence

[MIT](LICENSE)
