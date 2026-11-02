# claude-skills

Agent skills distilled from the official documentation of developer tools and frameworks.

Each skill is built from the vendor's own docs, never from model memory. Every claim in a skill links to the documentation page it comes from, and was checked against that page before publication. Each skill states the documentation version it covers.

## Skills

| Skill | Covers |
| :-- | :-- |
| [bullmq-docs](skills/bullmq-docs/) | BullMQ 6.x |
| [clamav-docs](skills/clamav-docs/) | ClamAV 1.5 |
| [eslint-docs](skills/eslint-docs/) | ESLint 10 |
| [grafana-cloud-docs](skills/grafana-cloud-docs/) | Grafana Cloud |
| [javascript-docs](skills/javascript-docs/) | JavaScript (MDN) |
| [jest-docs](skills/jest-docs/) | Jest 30.5 |
| [nestjs-docs](skills/nestjs-docs/) | NestJS 12 |
| [next-intl-docs](skills/next-intl-docs/) | next-intl 4 |
| [nextjs-docs](skills/nextjs-docs/) | Next.js 16.4 (App Router) |
| [nodejs-docs](skills/nodejs-docs/) | Node.js 26 |
| [opentelemetry-docs](skills/opentelemetry-docs/) | OpenTelemetry |
| [playwright-docs](skills/playwright-docs/) | Playwright Test 1.64 |
| [postgresql-docs](skills/postgresql-docs/) | PostgreSQL 18 |
| [prisma-docs](skills/prisma-docs/) | Prisma ORM 7 |
| [react-docs](skills/react-docs/) | React 19.3 |
| [redis-docs](skills/redis-docs/) | Redis Open Source 8.10 |
| [rust-std-docs](skills/rust-std-docs/) | Rust standard library 1.99 |
| [semgrep-docs](skills/semgrep-docs/) | Semgrep |
| [socketio-docs](skills/socketio-docs/) | Socket.IO 4.8 |
| [tailwindcss-docs](skills/tailwindcss-docs/) | Tailwind CSS 4 |
| [tauri-docs](skills/tauri-docs/) | Tauri 2 |
| [tokio-docs](skills/tokio-docs/) | tokio 1.54 |
| [typescript-docs](skills/typescript-docs/) | TypeScript 6.0 |
| [typesense-docs](skills/typesense-docs/) | Typesense 30.2 |
| [vite-docs](skills/vite-docs/) | Vite 8 |
| [vitest-docs](skills/vitest-docs/) | Vitest 5 |

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
