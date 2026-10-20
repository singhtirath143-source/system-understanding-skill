# System Understanding

A read-only skill for AI coding agents. Point it at a codebase and it maps how the system works before anyone
changes anything. It saves that map as a dated report, with file-and-line evidence for every claim.

Use it when you join a project, inherit a codebase, plan a change to code you don't know well, or want an agent
to understand the system before it touches it.

## What the report covers

| # | Section | What's in it |
|---|---|---|
| 0 | Summary | What the system is, the top 3 risks, a stack table, and what was and wasn't read |
| 1 | Project structure | Frontend, backend, middleware order, environment variables (names only), build and test commands, CI |
| 2 | Auth flow | Sign-in to backend validation as numbered steps, every auth file, auth risks |
| 3 | API and data flow | Every endpoint with its auth and validation, the main user flows end to end, external services |
| 4 | Background work and cloud | Workers, queues, schedules, serverless functions (e.g. Azure Function Apps, AWS Lambda), workflows (e.g. Logic Apps, Step Functions), webhooks, infrastructure as code |
| 5 | Database | Tables and fields, where the schema is defined, access rules, mismatches between the schema and the code |
| 6 | Branding | Hardcoded names, logos, theme values, URLs |
| 7 | Feature flags | Where flags are defined and checked, and what they gate |
| 8 | Routing | Public, signed-in, admin and dynamic routes, guards, 404 handling, broken links |
| 9 | Risks | Critical and moderate, each with evidence |
| 10 | Open questions | Things only a person on the team can answer |

See [a full example report](examples/nextjs-postgres-auth-starter.md), made by running the skill on a public
open-source starter app.

## How it stays safe

- **Read only.** The agent reads files, searches and runs read-only git commands. It doesn't install packages,
  run builds or migrations, start the app or call external services. The only file it writes is the report.
- **Report, don't fix.** It names problems so a person can decide what to do. It doesn't propose fixes or write
  code.
- **No guessing.** Anything it can't find is marked **Not found**, with where it looked. Anything it infers is
  marked **Inferred**, with why.
- **No secrets.** Environment variables and keys appear by name only, never their values.
- **The live cloud is opt-in.** By default it reads only the repo. If you explicitly ask it to look at deployed
  cloud resources too, it uses only list / show / describe commands. It never creates, changes, starts or
  triggers anything, and never runs commands that return secrets.

These are instructions to the agent, not technical enforcement. For a hard guarantee, run the agent where it
must ask before editing files or running commands, and approve only the report file.

## Install

### Claude Code

Copy `SKILL.md` into a folder named `system-understanding`, in one of these places:

- **This project only:** `.claude/skills/system-understanding/SKILL.md`
- **All your projects:** `~/.claude/skills/system-understanding/SKILL.md`

Then ask "understand this codebase" or "map the architecture before we change anything", or run
`/system-understanding`.

### Other agents

`SKILL.md` is plain Markdown, so it works with any agent that accepts instructions:

- paste its contents as the prompt, followed by "Run this on this repository"; or
- save it in the repo and point the agent's instruction file (e.g. `AGENTS.md`, `.github/copilot-instructions.md`,
  `.cursor/rules/`) at it.

## Where the report goes

Writing the report is the one change the skill makes, so if you haven't said where to put it, it asks first.
It suggests `docs/system-understanding/YYYY-MM-DD-system-understanding.md` in the project, or a folder outside
the repo if nothing in it may change. Each run writes a new dated file. If an earlier report exists, the new one
includes a "Changed since last report" section.

## License

[MIT](LICENSE)
