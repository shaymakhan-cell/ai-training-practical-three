---
description: Implement a feature in scheduler.py from the spec in specs/current.md, verify with pytest, and only stop when the relevant tests pass. Use after /spec has produced a current spec.
---

You are implementing a feature in `scheduler.py` from `specs/current.md`. Work in explicit phases. Do **not** report done until the relevant tests pass.

`$ARGUMENTS` may name a specific test file (e.g. `test_01_availability.py`) — if present, focus verification on that file. If absent, run the full suite.

## Phase 1 — Pre-flight

Before touching any file, do all of the following and tell the user what you found:

1. Read `specs/current.md`. If it doesn't exist or is empty, stop and tell the user to run `/spec` first.
2. Read `scheduler.py`. List every function already defined there (name and signature).
3. Identify the test file(s) that will verify this work. If `$ARGUMENTS` named one, use it. Otherwise pick the lowest-numbered file in `tests/` whose name matches the feature, and run `python -m pytest <that file> -v --collect-only` so you know exactly which test names you must satisfy.
4. Read those test files end-to-end. Tests are the contract — note any assertions about return types, error messages, or edge cases that the spec doesn't explicitly state.
5. Flag any conflicts between the spec and the tests **before** writing code. Ask the user to resolve them.

## Phase 2 — Plan

Output a plan to the user before touching any file. The plan must include:

- **New functions**: name, signature with type hints, one-line purpose
- **Modified functions**: name, what changes, why
- **Helper functions**: any internal helpers you intend to introduce
- **Data flow**: a one-paragraph description of how the new code fits with what's already in `scheduler.py`
- **Edge cases the spec calls out**: bullet list, each mapped to where it will be handled

After printing the plan, proceed directly to Phase 3 — don't wait for user approval. The plan is a contract you can be held to, not a request for permission.

If the spec is too thin to plan from (e.g. missing return type, no acceptance criteria, no error handling specified), stop here, list what's missing, and ask the user to run `/spec` again.

## Phase 3 — Implement

Write the code. Follow these standards:

- **Type hints on every public function** — parameters and return.
- **Brief docstrings** — one sentence summary + Returns + Raises. Don't write essays.
- **Specific exceptions** — raise the exact exception type and message the spec specifies. Tests often assert on message strings; if the spec gives wording, use it verbatim.
- **Small pure functions** — split out helpers when a function exceeds ~30 lines or mixes parsing with logic.
- **Don't add new dependencies** — use what's already in `requirements.txt` (`openpyxl`, `ortools`, `holidays`, `pytest`) plus the standard library.
- **Never break an existing function** — read its callers and tests before changing its signature. If you must change behavior, say so explicitly in the final report.

Edit `scheduler.py` in place. Prefer the `Edit` tool over `Write` for an existing file so unchanged code stays unchanged.

## Phase 4 — Verify

1. Run the targeted test file:
   ```
   python -m pytest tests/<test_file>.py -v
   ```
   (Use `python` from the activated `.venv`. If the user is not on Windows, `python3` is the same.)
2. If any test fails, do all of the following before re-editing:
   - Read the full failure output for the first failing test.
   - State, in one sentence, what the test expects vs what the code did.
   - Identify the smallest fix that addresses **only** the failing assertion.
3. Apply the fix, re-run the same test file.
4. Repeat up to **3 fix iterations**. If tests still fail after 3 attempts, stop and report: what's failing, what you tried, and what you think is wrong with either the code or the spec. Do not keep grinding.
5. Once the targeted file is green, run the full suite (`python -m pytest tests/ -v`) to verify no regressions. If a previously-passing test now fails, fix it before reporting done.

## Phase 5 — Report

Print a tight summary in this shape:

```
✓ Built: <function name(s)>
Tests: <N> passed, <M> failed (file: <test_file>)
Full suite: <N> passed (no regressions)

Watch out for:
- <any assumption you made that wasn't in the spec>
- <any tradeoff you took, e.g. linear scan vs hashmap>
- <anything the user should verify manually>
```

If you hit the 3-iteration cap without green tests, the report shape is instead:

```
✗ Build incomplete after 3 fix attempts.
Failing: <test names>
Attempts:
  1. <what you tried, why it didn't work>
  2. <…>
  3. <…>
Hypothesis: <code bug | spec ambiguity | test issue>
Suggested next step: <run /spec to clarify X | manually inspect Y>
```

## Anti-patterns to avoid

- **Don't skip Phase 1.** Reading the existing `scheduler.py` once at the start is faster than discovering mid-implement that a function already exists.
- **Don't loosen assertions to make tests pass** — if a test expects `ValueError`, raise `ValueError`, don't catch the real bug and return `None`.
- **Don't write to multiple files at once.** Edit `scheduler.py` only, unless the spec explicitly requires a new module.
- **Don't claim done with failing tests.** "Done" means the targeted file is green AND the full suite has no new regressions.
