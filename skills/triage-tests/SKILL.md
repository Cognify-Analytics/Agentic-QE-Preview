---
name: triage-tests
description: Use when the user asks to run tests and triage, classify, or fix failures, or to draft issues for failing tests.
allowed-tools: Read, Grep, Glob
---

# Triage Tests Skill

You run the test suite, classify every failure as SIMPLE (a bad test) or
COMPLEX (a suspected real bug), fix only what is safe to fix, and hand
everything else to a named person. You never guess, and you never let a real
bug disappear behind a quietly "fixed" test.

This preview is **report-only**: SIMPLE fixes stay in the working tree, and
every suspected bug is drafted as an issue in full and reported, never
filed. The full Agentic QE plugin
(https://github.com/Cognify-Analytics/Agentic-QE) adds live GitHub issues
and PRs, bug-fix mode and CI.

**Arguments, if given:** $ARGUMENTS — if they ask for any mode from the full
plugin (live GitHub writes, bug-fix mode, branch naming), say on screen that
it isn't available in this preview and carry on report-only. The same holds
when you are invoked by `/test-pipeline` rather than directly.

## The one rule that matters most

**A real bug labelled SIMPLE and quietly fixed is the failure this whole
project exists to prevent.** Every other rule below exists in service of
that one. When in doubt, escalate — never guess your way to SIMPLE.

## Ground rules (non-negotiable)

1. Never change production source code. A fix that would require a source
   change is automatically COMPLEX.
2. Never edit a test that is not `generated_by_qa: true` for its module in
   `.qa/test-inventory.json`. A human-written test that fails goes straight
   to COMPLEX, no matter what the failure looks like.
3. Never create a branch, commit, push, merge, or write to GitHub — no
   issues, comments, labels or PRs. Fixes stay in the working tree for the
   developer to review and commit.
4. A suspected bug gets a drafted issue, never a `// might be a bug` comment
   left in the code or the test.
5. Every limit hit (2 fix attempts, 10 issues, 30 turns) is a hard stop and
   gets recorded in `caps_hit`, not silently absorbed.

---

## Instructions

### Step 1 — Read the context

Read `.qa/context.md` if it exists. You cannot judge whether a failure is a
bug unless you know what the code is supposed to do. If it's missing, warn
on screen and continue with less confidence (see
`${CLAUDE_PLUGIN_ROOT}/rules/error-classification.md`'s "Applying this without context.md"
section) — do not stop. If `context_sha` in the file differs from the
current git SHA (or, if this repo has no git history yet, from
`.qa/test-inventory.json`'s `last_updated_sha`), warn that the context may be
stale and carry on with less confidence.

### Step 2 — Get the test command(s)

Read `.qa/test-inventory.json`. **Check for a `stacks` object first** — if
present, this is a multi-stack repo (see `/generate-tests`' multi-stack
variant of this file): read every stack's `test_command` under
`stacks.<name>.test_command`, there is no single top-level `test_command`
to read in this shape. If `stacks` is absent, read the single top-level
`test_command` as before.

If `.qa/test-inventory.json` is missing entirely, fall back to
`.claude-pipeline/01-tests.json` (same `stacks`-or-flat check applies
there too). **Do not guess the test command** — if neither file has it (for
any stack, in a multi-stack repo), stop and ask the user.

**Run every stack, always.** Even if `/generate-tests` only touched one
stack this run, run *every* stack's full suite here — a bug in a stack
nobody touched is still a bug, and skipping it would silently shrink
triage's coverage every run.

### Step 3 — Run the tests

**Redirect the output to a file; never let the full run stream into your
context.** A passing suite's output can run to thousands of lines, and
every one of them would be tokens spent to learn "nothing failed."

Run the test command with its output redirected to
`.claude-pipeline/raw-test-output.txt` and capture the exit code, e.g.
`<test command> > .claude-pipeline/raw-test-output.txt 2>&1; echo "exit=$?"`.
Then read **only the tail** (the last ~15 lines) to get the totals line.

**Multi-stack:** run each stack's test command separately (in that stack's
own subtree/environment — they are typically incompatible runtimes), and
save each stack's raw output to its own
`.claude-pipeline/raw-test-output-<stack>.txt` (e.g.
`raw-test-output-backend.txt`, `raw-test-output-frontend.txt`) instead of
one combined file — two frameworks' output interleaved in one file is not
reliably parseable. Capture one exit code per stack.

**A stack that failed before any test ran is a tooling problem, not a
test failure.** Recognise it from the tail: no test results at all, plus
an error from the tooling itself, for example
- `Cannot find dependency '@vitest/coverage-v8'`
- `Test environment jest-environment-jsdom cannot be found`
- `Preset ts-jest not found`
- `ModuleNotFoundError: No module named 'pytest_cov'` / `unrecognized arguments: --cov`
- `No tests found` / `Tests run: 0` caused by configuration
- a Maven/Gradle build failure outside test sources

Then:
1. **If the error is about coverage only**, re-run the same command once
   with its coverage flags removed (`--coverage…`, `--cov…`, the JaCoCo
   goals or init script). If that runs, triage its results normally;
   triage only needs pass/fail, not coverage.
2. Record the gap in `03-triage.json`'s `tooling_gaps`. Also add it to
   `.qa/test-inventory.json`'s `tooling_gaps` (same entry shape as
   `/generate-tests` writes, `"detected_by": "triage-tests"`), replacing
   only your own earlier entries for this stack. That's how it reaches
   section 5 of `.qa/context.md`, where the developer is told what to
   install.
3. If the stack still couldn't run, classify nothing and draft no issues
   for it. A missing package is not a code bug. Put one line in
   `escalated` naming the stack and the missing tool, and carry on with
   the other stacks.

**If every stack exited 0 (a green suite): stop here.** There are no
failures to classify, so Steps 4 through 7 have nothing to act on. Record
the totals from the tail you already read, skip to Step 8, and write
`03-triage.json` with empty `classifications`/`fixes_applied`/
`issues_created`. Do **not** read the raw output file — it is kept on disk
so a disputed run is still arguable after the fact, not for you to load.

### Step 4 — Extract every failure

*(Only reached when at least one stack failed — see Step 3.)*

**Pull out failures with targeted searches, not by reading the whole
file.** Grep the raw output for the framework's failure markers to get the
list, then read only the surrounding block for each one — reading a
5,000-line log to find 3 failures costs far more than 3 targeted reads.

| Framework | Failure markers to grep |
|---|---|
| pytest | `FAILED `, `ERROR ` (short summary at the end) |
| Jest | `● ` (one block per failing test: `● describe › test name`), `FAIL ` (per file) |
| Vitest | `FAIL `, `×` / `✗`, `AssertionError` |
| node:test | `not ok `, `✖` |
| Maven Surefire | `<<< FAILURE!`, `<<< ERROR!`, and the `[ERROR] Failures:` / `[ERROR] Errors:` summary |
| Gradle | `> … FAILED` lines (`OrderServiceTest > createOrder_validRequest_returnsOrderId() FAILED`) |
| Go | `--- FAIL:` |

**JVM builds also write structured results**, which are more reliable than
the console: `target/surefire-reports/*.txt` / `TEST-*.xml` (Maven) and
`build/test-results/test/TEST-*.xml` (Gradle), one file per test class,
with the full assertion message and stack trace of each failure. Grep
those for `<failure`/`<error` and read only the failing class's file. A
**compile error** in a test source (`COMPILATION ERROR`, `Compilation
failed`, `e: file://…`) means no test ran at all in that module: record
it as one failure against the test file that failed to compile, and
classify it like any other.

For each failing test, pull out: test name, file, error type, message, and
stack trace/traceback. **Multi-stack:** also record which stack each
failure came from (the file path's stack, per Step 2) — Step 8's output
schema carries a `"stack"` field per failure for exactly this.

### Step 5 — Read the rules and classify

Read `${CLAUDE_PLUGIN_ROOT}/rules/error-classification.md`. For each failure:

1. **Check provenance first.** Look up the failure's test file in
   `.qa/test-inventory.json`'s `modules`. If the module's `generated_by_qa`
   is `false`, the classification is **COMPLEX**, full stop — do not go on
   to inspect the error type. (If the inventory doesn't cover this file at
   all, treat it the same as `generated_by_qa: false` — you have no basis
   to believe this pipeline owns it.)
2. Otherwise, match the failure against the SIMPLE/COMPLEX tables in
   `${CLAUDE_PLUGIN_ROOT}/rules/error-classification.md`.
3. If it doesn't cleanly match, it's AMBIGUOUS — which is COMPLEX. Never
   guess.

Record every classification with a one-line, specific reason (not "looks
wrong" — cite the assertion, the docstring, the doc line, or the exact
provenance check that decided it).

### Step 6 — SIMPLE: fix the test

For each SIMPLE failure:

1. Fix it **in the working tree only**: no branch, no commit. Usually the
   file is new in this run — not tracked at the commit the run started from
   (`git ls-files --error-unmatch <file>` fails), the normal case for a file
   `/generate-tests` just wrote — and the fix travels with the rest of the
   generated tests in the developer's own commit. A fix to a test file that
   is already tracked (generated by an earlier run and committed) stays in
   the working tree too, as an uncommitted change: name it on screen so the
   developer reviews it before committing.
2. Make the smallest possible fix **to the test file** — never to production
   source. This is fixing a mistake in how the test was written (a typo,
   a wrong import, a wrong fixture keyword), not re-deriving what the test
   should assert.
3. Re-run just that test, with the single-test command for its framework
   (`${CLAUDE_PLUGIN_ROOT}/rules/general/test-execution-verification.md`
   lists them — `npx vitest run <file>`, `./mvnw -q test -Dtest=<Class>#<method> -Dsurefire.failIfNoSpecifiedTests=false`,
   `./gradlew test --tests '<fqcn>.<method>'`, …). Never a watch mode.
4. If it passes: keep the fix.
5. If it still fails: undo the change, relabel the failure COMPLEX, and move
   on. Maximum **2 attempts** per failure before relabeling.

If, partway through, you realize the "fix" would require touching production
code to make the test pass — stop immediately, undo any partial edit, and
relabel COMPLEX. Do not rationalize a source change as a test fix.

If Step 6 produced no fixes, say so and move on. An empty run is a normal
outcome, not a failure.

### Step 7 — COMPLEX/AMBIGUOUS: find the owner and draft the issue

1. **Find the owners of the failing file's source module by reading
   CODEOWNERS yourself.** Never look owners up on GitHub.
   - Use the first of these that exists: `CODEOWNERS` at the repo root,
     `.github/CODEOWNERS`, `docs/CODEOWNERS`.
   - Skip blank lines and `#` comments. Every other line is a pattern
     followed by its owners (`@user`, `@org/team`, or an email).
   - Match the source module's repo-relative path (forward slashes) against
     every pattern, gitignore-style: a leading `/` anchors to the repo
     root; a pattern ending in `/` or `/**` matches everything under that
     directory; `*` and other globs match as usual (`*.py` matches any
     `.py` file).
   - **The last matching pattern wins** — GitHub's own CODEOWNERS
     precedence. Its owners are the owners; earlier matches are discarded,
     not merged. Note the pattern that matched: the issue's Owners line
     names it.
   - No CODEOWNERS file, or no pattern matches, means `owners: []` — a
     normal outcome, not an error; draft the issue without an owner rather
     than guessing one.
2. **Build a stable fingerprint for this failure**: `{test name} ({file})` —
   e.g. `calculate_discount_premiumTier_returnsDocumentedRate
   (tests/test_pricing.py)`. Put it verbatim in the issue title (prefixed by
   the template's own title convention, if any). This is what makes a
   re-run idempotent — a run's issue for the same failure must always
   produce the same title.
3. **Existing issues can't be checked in the preview** — it never reads
   GitHub. Say once on screen that a drafted issue may duplicate one
   already open or already decided, and that the fingerprinted title is
   what to search for before filing it.
4. Draft a new issue from `${CLAUDE_PLUGIN_ROOT}/templates/github-issue-template.md`,
   filling in the fingerprinted title, the failure output, the relevant
   excerpt of `.qa/context.md` (or whatever you used in its absence, per
   Step 1), and the resolved owners if any. Keep its `agentic-qe:issue`
   marker exactly as the template shapes it.
5. Do not call the GitHub MCP server or any GitHub CLI — write the
   fully-drafted issue content into `03-triage.json`'s `issues_created` and
   show it on screen, clearly marked `[REPORT-ONLY — not filed]`. Record
   `owners` as the would-be assignees.

**Cap: 10 issues per run.** Beyond that, stop drafting individual issues and
combine everything past the cap into one summary issue draft. Record this in
`caps_hit`.

### Step 8 — Write the result

Write `.claude-pipeline/03-triage.json` (schema below). This is scratch —
gitignored, not committed.

---

## Expected output — `.claude-pipeline/03-triage.json`

```json
{
  "status": "ok",
  "mode": "report-only",
  "git_sha": "a1b2c3d",
  "context_sha": "a1b2c3d",
  "context_stale": false,
  "totals": { "tests": 84, "passed": 79, "failed": 5 },
  "classifications": [
    { "test": "test_create_order_valid",
      "file": "tests/test_orders.py",
      "type": "SIMPLE",
      "reason": "ImportError — OrderRequest imported from wrong module" },
    { "test": "test_apply_discount_premium_tier",
      "file": "tests/test_pricing.py",
      "type": "COMPLEX",
      "reason": "Premium tier discount returns 10%, docs/pricing.md says 15%. Looks like a real bug in src/pricing.py." }
  ],
  "fixes_applied": [
    { "test": "test_create_order_valid",
      "change": "corrected import path to src.models.order",
      "attempts": 1,
      "where": "working-tree" }
  ],
  "issues_created": [
    { "url": null,
      "test": "test_apply_discount_premium_tier",
      "title": "[QA Pipeline] Premium tier discount below documented rate — test_apply_discount_premium_tier (tests/test_pricing.py)",
      "owners": ["@alice", "@acme/payments"],
      "reason": "suspected real bug in discount calculation",
      "report_only": true,
      "drafted_body": "... full issue body from the template ..." }
  ],
  "tooling_gaps": [],
  "escalated": [],
  "caps_hit": []
}
```

`fixes_applied[].where` is always `"working-tree"` in the preview (Step 6):
every fix is an uncommitted change for the developer to commit.

`tooling_gaps` uses the same entry shape as `/generate-tests`' inventory
field (`stack`, `tool`, `needed_for`, `version`, `evidence`, `effect`,
`detected_by`, `detected_at_sha`). It is `[]` when every stack started
normally (see Step 3).

`mode` is always `"report-only"`. Every issue's `url` is `null` and each
carries `"report_only": true` with the drafted content (`title` and
`drafted_body`) fully present — a human reading this file must be able to
file it by hand exactly as drafted.

`owners` is the raw CODEOWNERS result from Step 7, exactly as CODEOWNERS
writes it, and `[]` when there is no CODEOWNERS file or no pattern matched.

**Multi-stack variant:** `totals` stays the sum across every stack; add a
`totals_by_stack` object alongside it, keyed by stack name, each value the
same `{tests, passed, failed}` shape. Every entry in `classifications`,
`fixes_applied` and `issues_created` carries a `"stack"` field naming which
stack it came from, e.g.
`{ "test": "test_require_user_access...", "file": "backend/tests/test_auth.py", "stack": "backend", "type": "COMPLEX", "reason": "..." }`.
Nothing else in the schema changes.

---

## Troubleshooting

**`.qa/context.md` missing.** Warn, continue with reduced confidence — see
`${CLAUDE_PLUGIN_ROOT}/rules/error-classification.md`. Do not stop.

**`.qa/test-inventory.json` missing (so you can't check `generated_by_qa`).**
Treat every failing test as if it were human-written (COMPLEX-only) until
proven otherwise — the safe default when provenance is unknown. Say so in
the run's notes.

**Test command unknown and can't be found anywhere.** Stop and ask the user.

**No CODEOWNERS file, or no pattern matches the file.** Not an error. Draft
the issue with `owners: []` and leave the template's Owners line out.

**More than 10 COMPLEX/AMBIGUOUS failures.** Draft the first 10
individually, then one combined summary issue for the rest, and record the
cap in `caps_hit`.

**A SIMPLE fix's second attempt also fails.** Undo it, relabel COMPLEX,
move on — don't try a third time.

---

## Rules reference

- `${CLAUDE_PLUGIN_ROOT}/rules/error-classification.md` — SIMPLE / COMPLEX / AMBIGUOUS and the
  human-written-test override
- `${CLAUDE_PLUGIN_ROOT}/templates/github-issue-template.md` — issue body shape
