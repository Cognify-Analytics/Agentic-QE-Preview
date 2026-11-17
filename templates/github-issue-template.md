<!--
Template for the issues /triage-tests drafts for COMPLEX/AMBIGUOUS
failures. The preview never files them: it shows each draft in its report
so a developer can file it by hand.
Fill every {placeholder}. Omit the "Owners" line entirely if no
CODEOWNERS pattern matched, rather than leaving a placeholder or guessing
one.

Keep the `agentic-qe` label and the marker comment below exactly as
shaped (valid JSON, one line), so the issue is recognised if the repo
later moves to the full Agentic QE plugin. Fill the marker's fields from
the same values as the body.
"classification" is the error-classification row that decided COMPLEX
(e.g. "behaviour contradicts docs", "timing or race condition").
-->
Title: [QA Pipeline] {short description of the suspected defect}
Labels: agentic-qe

<!-- agentic-qe:issue {"fingerprint": "{test_name} ({test_file})", "test": "{test_name}", "file": "{test_file}", "source": "{source_file}", "stack": {"stack name" or null}, "classification": "{classification row}", "generated_by_qa": {true|false}, "origin_branch": "{branch the run was on}", "origin_sha": "{git_sha}"} -->

**Failing test:** {test_name}
**File:** {test_file}
**Detected at:** {git_sha}

**What happened**
{one or two lines: what was expected vs what actually happened, plus the
raw assertion/error line}

**Why we think this is a real bug, not a bad test**
{the specific evidence: a quote or reference from .qa/context.md, a docstring,
or an adjacent doc, contrasted with what the code actually does at
{file}:{line}. If .qa/context.md was unavailable, say so and name whatever
source you used instead (docstring, README, etc.), or say plainly that no
documented expectation was found and the assertion looks like a directly
observed behavioral gap.}

**Owners:** {@owner, @org/team, ...} (from CODEOWNERS: {matched pattern})

**We did not change any code.** This needs a human decision about which is
correct — {the documented/expected behavior} or {the current implementation}.

**What should happen next?** A maintainer decides which is correct and either fixes the code or updates the test to the confirmed behaviour.
