---
name: test-pipeline
description: Use when the user asks to run the full QA pipeline, audit test health end to end, or set up testing for a repository.
allowed-tools: Read, Grep, Glob
---

# Test Pipeline Skill

You run `/generate-tests`, `/build-context`, and `/triage-tests` in
sequence, checking after each one that it actually produced valid output
before letting the next stage rely on it. You run in the developer's main
session so they can watch each stage happen.

This is the report-only preview of Agentic QE. It never creates a branch,
commits, pushes, or writes to GitHub: suspected bugs are drafted as issues
for the developer to file. The full Agentic QE plugin
(https://github.com/Cognify-Analytics/Agentic-QE) adds live issues and
PRs, bug-fix mode, issue follow-up, run history and CI. If the developer
asks for any of those (`--live`, `--fix-bugs`, `--clickup-branches`), say
in one line that this preview doesn't have it, and carry on report-only.

**Arguments, if given:** $ARGUMENTS — a file or folder is forwarded to
`/generate-tests` as its target. With no target, it scans the whole repo
per its own Step 5 ranking.

---

## Before starting

1. **Check this is a git repo.** Run `git rev-parse --is-inside-work-tree`.
   If it fails, **stop immediately** and tell the user: this pipeline needs
   git for the SHA and branch stamped into every output file, and for
   telling the tests it wrote apart from files already committed. Suggest
   `git init` and an initial commit, then stop. Do not create any other
   file yet.
2. Create `.qa/` if it doesn't exist.
3. Create `.claude-pipeline/` if it doesn't exist.
4. **Write `.claude-pipeline/run.json`** (schema below) **before invoking
   any stage**, since that's how `/generate-tests` knows it is running
   inside the pipeline and skips its approval prompt. Take `sha` from
   `git rev-parse --short HEAD` and `branch` from
   `git rev-parse --abbrev-ref HEAD`.
5. Ensure `.gitignore` has a `.claude-pipeline/` line (add it if missing;
   create `.gitignore` if the repo has none). A one-liner is enough:
   `grep -qxF '.claude-pipeline/' .gitignore 2>/dev/null || echo '.claude-pipeline/' >> .gitignore`
6. Ensure `.gitignore` does **not** ignore `.qa/` — that folder is meant to
   be committed. If you find a `.qa` or `.qa/` line in `.gitignore`, remove
   it and tell the user why: their test inventory and context doc would
   otherwise never make it into version control and the next run would
   look like a first run to every teammate.

**Expected — `.claude-pipeline/run.json`:**

```json
{
  "sha": "a1b2c3d",
  "branch": "feature/refunds",
  "timestamp": "2026-08-21T14:02:00Z",
  "mode": "pipeline"
}
```

`timestamp` is the real current UTC time, to the second. Take it from
`date -u +%Y-%m-%dT%H:%M:%SZ` when you write the file. Never write today's
date with `T00:00:00Z` or any other made-up time.

---

## Stage 1 — `/generate-tests`

Invoke it with the Skill tool (forwarding any target from `$ARGUMENTS`).

**Check before continuing:**
- Does `.claude-pipeline/01-tests.json` exist, and is its `"status"` not
  `"failed"`?
- Has `.qa/test-inventory.json` actually been updated (its
  `last_updated_sha` matches this run's `sha` from `run.json`)?

**If either check fails: stop the whole pipeline right here.** Do not run
Stage 2 or 3. Report exactly which check failed and show whatever
`01-tests.json` does contain (even a `"status": "partial"` file has useful
detail). This check matters most of all — a silent failure here would hand
Stage 2 nothing to document and Stage 3 nothing real to triage.

**Then re-check every skip that blames the environment.** Some skips say a
package is missing or a module can't be imported. Look for them in
`01-tests.json`'s `modules_skipped` entries, and in `tooling_gaps` entries
with `"detected_by": "generate-tests"` that left a module or stack untested.
For each one, run a single command yourself, in that stack's folder:
- for an import claim: `python -c "import <module>"`;
- for a package claim: `python -m pip show <package>` or `npm ls <package>`.

Do this even when the entry has `evidence`. One cheap command per claim is
how a wrong skip gets caught before it costs a whole stack its tests.
- **The command fails as claimed:** the skip stands.
- **It succeeds, because the module imports or the package is installed:**
  the skip was wrong. Say so on screen, naming the claim and the command
  that disproved it. Then invoke `/generate-tests` once more, with the
  wrongly skipped module's path as its target, or the stack's folder if the
  whole stack was skipped. It merges into this run's `01-tests.json` and
  inventory (its Step 14). Run the checks above again afterwards.
  **At most one such re-run per pipeline run.** Anything still wrong after
  it goes in `summary.md` under "## Needs a human".

## Stage 2 — `/build-context`

Invoke it with the Skill tool. No target to forward — it always looks at
the whole repo.

**Check before continuing:** does `.qa/context.md` exist and have content
(not just a header with empty sections)? Note: if `/build-context` correctly
determined nothing needed to change and reported "context is current," the
existing file having content still satisfies this check — that's success,
not failure.

**If the check fails: stop.** Do not run Stage 3 against a missing or
empty context document — `/triage-tests` explicitly depends on it to judge
whether a failure is a real bug.

## Stage 3 — `/triage-tests`

Invoke it with the Skill tool.

**Check before continuing:** does `.claude-pipeline/03-triage.json` exist?

**If the check fails: stop and report.** Don't write a summary claiming
triage happened if it didn't produce output.

---

## If you stop early at a stage gate

Before stopping at any gate above, write `.claude-pipeline/stages.json`
(the stages that ran, and `"not_reached"` for the rest) and
`.claude-pipeline/summary.md`, saying where and why the run stopped. Fill
in whatever is genuinely known and leave the rest out rather than guessing
at numbers no stage produced. Then report what failed on screen and stop.

---

## Finish

1. **Write `.claude-pipeline/stages.json` first** (shape below) — before
   the summary, so it survives even if a later step here fails.
2. Read `run.json`, `01-tests.json`, `.qa/context.md`, and `03-triage.json`.
3. Write `.claude-pipeline/summary.md` (shape below), and show it on
   screen.
4. **Remind the developer to commit** `.qa/context.md`,
   `.qa/test-inventory.json`, every test file `01-tests.json` listed under
   `files_created`/`files_modified`, and every test file fixed in triage
   (each `fixes_applied` entry's test, with its file taken from the matching
   `classifications` entry). This pipeline never commits on its own —
   commits and merges are for people.
5. **Point out the drafted issues.** If `03-triage.json`'s `issues_created`
   is not empty, say how many suspected bugs were found, that none were
   filed, and that each one's `title` and `drafted_body` in
   `.claude-pipeline/03-triage.json` are ready to paste into a new GitHub
   issue.

**Expected — `.claude-pipeline/stages.json`:**

```json
{
  "generate_tests": "ran",
  "build_context": "ran",
  "triage_tests": "ran"
}
```

Each value is `"ran"` or `"not_reached"` (the pipeline stopped before it).

**Expected — `.claude-pipeline/summary.md`:**

```markdown
# QA Pipeline Run — {project name}
Started {timestamp} | SHA {sha} | branch {branch} | report-only

## Tests
{files created/modified from 01-tests.json, coverage before -> after}
{tests_flagged_as_possible_bugs count, if any, called out by name}
{one line per coverage_shortfall entry: "<module> stopped at <coverage>%: <what is uncovered> (<reason>)"}
{one line per modules_skipped entry that has evidence: "<module> not tested: <reason> (<evidence>)"; and, if Stage 1's skip re-check disproved a claim, say which one and that /generate-tests was re-run}

## Context
{"context is current" or which sections were updated}

## Failures
{totals from 03-triage.json: passed/failed, SIMPLE fixed count, COMPLEX/AMBIGUOUS count}

## Needs a human
{if the inventory has any tooling_gaps: one line per gap — "<stack>: install <tool> <version> (<what it broke>)" — then "Install steps (PowerShell and bash): .qa/context.md, section 5"}
{one line per issue in issues_created: its title and owners (or "no CODEOWNERS match"), then "Drafted, not filed — full text in .claude-pipeline/03-triage.json"}
{one line per entry in caps_hit}

## Changes made
{the test files fixed in the working tree (fixes_applied), and "no source code changed"}

## Please commit
{the exact file list from Finish step 4 above}
```

**Multi-stack repos** (`01-tests.json`/`03-triage.json` have a `stacks`
shape — see `/generate-tests`): report coverage and triage totals **per
stack** in `summary.md`'s "## Tests" and "## Failures" sections. Never
collapse two stacks into one combined number.

---

## Troubleshooting

**Not a git repo.** Stop before creating anything (see Before starting,
step 1). This is the one failure mode that blocks the pipeline before it
even starts, rather than between stages.

**Stage 1 fails (no test framework detected, or it stops for any other
reason).** Report it and stop. Stage 2 and 3 must never run against a
repo that Stage 1 gave up on.

**Stage 2 says "context is current" every time even after real changes.**
That's a `/build-context` bug, not something to work around here — report
it, don't force a rebuild yourself by deleting `.qa/context.md`.

**A stage's own skill invocation errors out entirely** (crashes rather than
writing a `"status": "failed"`/partial file). Treat it the same as a failed
check — stop, report what's known, do not guess at what the missing stage
would have produced.
