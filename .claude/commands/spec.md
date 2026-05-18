---
description: Interview the user about a feature they want to build, then write a complete technical spec to specs/current.md. Use before implementing any new function or module.
---

You are about to write a technical specification for a piece of code the user wants to build. A great spec is the difference between "build something roughly like this" and "build exactly this — no clarifying questions needed."

Your output will be `specs/current.md`. Read `specs/example.md` first so you match its structure and level of detail.

## Goal

By the end of this conversation, `specs/current.md` should be precise enough that a fresh Claude session could implement it correctly without re-asking any of the questions below.

## How to interview

This is a **focused interview**, not free-form chat. Ask questions in **batches** so the user can answer several at once. Only follow up on what's still unclear.

`$ARGUMENTS` may contain a one-line description of what to spec. If empty or vague, ask first: *"In one or two sentences, what are you trying to build?"*

After each round, write a tight bullet summary of what you have so far and ask for corrections before moving on.

### Round 1 — Intent & inputs

Ask, in a single message:

1. What's the function signature? Name, parameters, types.
2. Where does the input data live and what format is it in? (File path, in-memory object, network call, etc.)
3. Show me a small concrete example of the input — a few rows, a sample object, the typical file structure.

### Round 2 — Output & guarantees

Ask, in a single message:

1. What does the function return? Exact type and shape.
2. What ordering or sorting guarantees apply? (Sorted by what key? Insertion order? Unspecified?)
3. What happens on empty input? On a missing or unreadable input source?

### Round 3 — Rules and edge cases

Probe specifically for things the user probably hasn't thought about. Use targeted prompts, not "anything else?":

1. What invariants must always hold on the output? (Uniqueness, non-empty, all elements of type X, etc.)
2. How should the function behave with: blank/whitespace values, duplicates, case differences, Unicode, unexpected types?
3. Are any inputs *errors* (raise an exception) vs *ignored* (skip and continue)? State which for each scenario.

### Round 4 — Engineering

Ask in one message:

1. Performance constraints — expected input size, must it stream, memory limits?
2. Required or banned libraries?
3. Exact error message text (for any exceptions raised). Tests often check message strings.

## Handling vague answers

When the user says "handle errors gracefully" or "make it robust," do **not** accept it. Ask concretely:

> "When X occurs, should the function: return `None`, raise `ValueError`, log a warning and skip the row, or something else?"

If after a follow-up the user still isn't sure, **propose a default and call it out as an explicit assumption** in the final spec. Never bury an assumption.

## Write the spec

Once the interview is complete, write `specs/current.md` with these sections in this order:

1. **Title** — `# Specification: <function or module name>`
2. **Overview** — one paragraph in plain English
3. **Inputs** — table of parameters (`name | type | description`), plus a paragraph on the data/file format if relevant
4. **Outputs** — return type, shape, and any ordering guarantees
5. **Rules** — numbered list of testable invariants
6. **Acceptance Criteria** — `AC1`, `AC2`, … each stated as *"Given X, returns Y"* or *"Given X, raises Y"*
7. **Test Cases** — three or more concrete pytest-style test functions using realistic data (use `data/basic.xlsx` when relevant)
8. **Edge Cases** — a table of `(input, expected behavior)` rows
9. **Engineering Notes** — library choices, performance notes, anything subtle

Match the structure and depth of `specs/example.md`. If the user listed any assumptions you proposed during the interview, surface them under **Engineering Notes** with a leading "**Assumption**:" so they're easy to find.

## After writing

1. Read the spec back to the user in a tight bullet summary — don't paste the whole file.
2. Ask: *"Anything missing or wrong?"*
3. Iterate, but cap at two revision rounds. If anything is still loose after that, add a **Known Gaps** section at the bottom listing what's unresolved, and stop.

## Quality bar

A good spec from `/spec` makes a follow-up round unnecessary. You should be able to hand `specs/current.md` to a fresh Claude session, say "implement this," and get a correct implementation without it asking the questions above.
