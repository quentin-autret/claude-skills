# claude-skills

Agent skills distilled from the official documentation of developer tools and frameworks.

Each skill is built from the vendor's own docs, never from model memory. Every claim in a skill links to the documentation page it comes from, and was checked against that page before publication. Each skill states the documentation version it covers.

## Skills

No skill published yet.

## Layout

```
skills/
  <technology>-docs/
    SKILL.md            overview, covered version, how to navigate the references
    references/         one file per documentation domain, loaded only when needed
    evals/evals.json    scenarios used to test the skill
    SOURCES.md          pages read, version, fetch date
```

## Using a skill

Copy a skill folder into `.claude/skills/` of your project (or `~/.claude/skills/` for all projects). Claude Code picks it up without a restart. Claude loads the `SKILL.md` when a task matches its description, then reads only the reference files the task needs.

## Versions

A skill covers the documentation version written in its `SKILL.md` and in its `metadata`. Check it against the version your project uses before relying on version-specific advice.

## License

MIT, see [LICENSE](LICENSE). The documentation each skill draws from belongs to its authors, under its own license, linked from the skill's `SOURCES.md`.
