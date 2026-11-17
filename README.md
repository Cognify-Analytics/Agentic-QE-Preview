# Agentic QE Preview

Claude Code skills that write unit tests, document a codebase, run the
tests, and decide what each failure means: a mistake in the test, or a real
bug in the code. Everything runs on your machine inside Claude Code. It
never pushes, merges, or writes to GitHub.

This is the **report-only preview** of
[Agentic QE](https://github.com/Cognify-Analytics/Agentic-QE). The full
plugin adds filing issues and opening pull requests on GitHub, an opt-in
bug-fix mode, follow-up on maintainers' replies, run history with cost
tracking, and a GitHub Action that runs the pipeline on every pull request.

## Skills

| Command | What it does |
|---|---|
| `/agentic-qe-preview:generate-tests [file or folder]` | Writes unit tests for a target, matching the repo's own conventions, runs them, and reports coverage |
| `/agentic-qe-preview:build-context` | Builds or updates `.qa/context.md`, a living overview of the project: what it does, its data schema, architecture, test coverage, and the test tooling it needs |
| `/agentic-qe-preview:triage-tests` | Runs the suite and classifies every failure as a bad test or a real bug. Fixes simple mistakes in tests the pipeline wrote, and drafts a GitHub issue for each suspected bug |
| `/agentic-qe-preview:test-pipeline [file or folder]` | Runs generate-tests, build-context and triage-tests in sequence, checking each stage's output before the next one starts |

With no target, `generate-tests` and `test-pipeline` rank the whole repo
and pick what most needs tests.

## What it will and won't do

- **Never changes production code.** If a test fails because the code is
  wrong, that is reported as a suspected bug, with the evidence, and left
  for a person to decide.
- **Never edits a test a person wrote.** Only tests the pipeline itself
  generated (marked `generated_by_qa: true` in `.qa/test-inventory.json`)
  are ever fixed.
- **Never edits manifests or build files.** If the repo is missing a test
  tool (a test runner, a coverage provider), the run says exactly what to
  install and skips what needs it.
- **Drafts issues, doesn't file them.** Each suspected bug comes with a
  complete issue (title, evidence, code owners from `CODEOWNERS`) in the
  run's report. File the ones you agree with yourself.

## Supported languages

The pipeline works out a repo's language and test setup from its manifest
files and never assumes one. Every language gets the shared rules in
`rules/general/`. Four languages also have their own rules:

| Language | Test frameworks | Framework-specific rules | Coverage read from |
|---|---|---|---|
| Python | pytest | Web backends (FastAPI, Starlette, Flask, Django) | pytest-cov (Cobertura) |
| JavaScript / TypeScript | Jest, Vitest, `node:test` | React, Next.js, Node backends (Express, Fastify, NestJS, Koa, Hono) | Cobertura or LCOV |
| Java | JUnit 5 (JUnit 4 when that's all there is), AssertJ, Mockito | Spring Boot | JaCoCo (Maven, Gradle) |
| Kotlin | JUnit 5, Kotest, MockK (or Mockito via mockito-kotlin), coroutines | Spring Boot | JaCoCo or Kover |

Go, C#, Rust, Ruby, PHP, Elixir, Scala and Swift are detected and get the
general rules, their default test location and their build commands, but
no language-specific rules yet.

To see what your repo needs, open section 5, "Test tooling requirements",
of `.qa/context.md`. For each stack it lists every tool the pipeline needs,
whether the repo has it, the version that fits, and a ready-to-paste
install command.

## Install

You need [Claude Code](https://code.claude.com) installed. Then, in a
Claude Code session:

```
/plugin marketplace add Cognify-Analytics/Agentic-QE-Preview
/plugin install agentic-qe-preview@agentic-qe-preview-marketplace
```

Check it worked: type `/agentic-qe-preview:` and the four skills should be
listed.

## Run it

From the root of the repo you want to test (it must be a git repo):

```
/agentic-qe-preview:test-pipeline                  # whole repo
/agentic-qe-preview:test-pipeline src/pricing.py   # one target
```

The skills don't pre-approve anything beyond reading files, so Claude Code
asks before it writes a file or runs a command (your test command, `git`).
Choose "always allow" for the ones you're happy with, or add them to your
[permission settings](https://code.claude.com/docs/en/settings).

## What a run leaves behind

| Path | What it is | Commit it? |
|---|---|---|
| `.qa/test-inventory.json` | Every module, its tests, coverage, and which tests the pipeline wrote | Yes |
| `.qa/context.md` | The project overview from `build-context` | Yes |
| The new and fixed test files | Wherever the repo keeps its tests | Yes, after you review them |
| `.claude-pipeline/` | Scratch files for the current run, including the triage report with the drafted issues | No (added to `.gitignore` for you) |

At the end of a run, the summary lists exactly which files to commit.

## Privacy

Agentic QE Preview has no server of its own. Nothing is sent to Cognify
Analytics, and Cognify does not receive or keep any of your data.

**What it reads.** Your repository's source, tests, docs and build files,
and its `CODEOWNERS` file, which can contain GitHub usernames, team names
and email addresses.

**What it stores, and where.** Only in your own repository: `.qa/` (the
project context document and the test inventory), the test files it
writes, and `.claude-pipeline/` (scratch files, git-ignored). The drafted
issues in `.claude-pipeline/` can include owners named in `CODEOWNERS`.

**What it sends, and to whom.** Nothing. The plugin makes no network
requests of its own and never contacts GitHub. Running your test suite can
contact whatever your tests or build tool reach (for example, a package
registry while Maven or Gradle resolves dependencies).

Claude reads your code while running the skills, as it does for any
Claude Code session. That is covered by your agreement with Anthropic.

## License

MIT. See [LICENSE](LICENSE). Use of the plugin is also covered by the
[Terms of Service](TERMS.md).

Support: [business@cogniify.ai](mailto:business@cogniify.ai)
