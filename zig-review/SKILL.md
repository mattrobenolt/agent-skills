---
name: zig-review
description: >
  Comprehensive Zig code review across all quality dimensions. Triggers on /zig-review.
  Runs parallel subagents — each using an existing skill — for Tiger Style compliance,
  Zig 0.15 idiom correctness, deslop, simplification opportunities, and API design quality.
  Always operates on the full branch diff against main. Report-only: never makes changes.
---

# Zig Review

## Step 1: Get the diff

```bash
git diff main...HEAD
```

Fall back to `git diff HEAD` if that returns nothing.

## Step 2: Launch five parallel review agents

Spawn all five in a **single message** so they run concurrently. Each agent receives the full diff. All agents are **report-only** — they must not make changes.

**Tiger Style agent:**
> Use the tiger-style skill to review the following diff. Report violations only — do not make any changes. For each finding include the file and approximate line, what rule is violated, and whether it is a must-fix or a suggestion.
>
> [full diff]

**Zig idioms agent:**
> Use the zig skill to review the following diff for Zig 0.15 correctness issues and outdated patterns. Report findings only — do not make any changes. For each finding include the file and approximate line, what the issue is, and the correct 0.15 pattern.
>
> [full diff]

**Deslop agent:**
> Use the deslop skill to review the following diff for AI-generated comment slop. Report findings only — do not remove any comments. For each finding include the file and approximate line, the comment text, and why it is slop.
>
> [full diff]

**Simplify agent:**
> Use the simplify skill to review the following diff for simplification opportunities. Report opportunities only — do not make any changes. For each finding include the file and approximate line and a concise description of what could be simplified and why.
>
> [full diff]

**API design agent:**
> Review the following diff for public API and interface design quality. Report findings only — do not make any changes.
>
> Focus on:
> - **Clarity**: Are public function names, parameter names, and types self-documenting? Would a caller understand usage without reading the implementation?
> - **Ergonomics**: Is the call site pleasant? Are common cases easy and advanced cases possible?
> - **Minimalism**: Is the public surface as small as it can be? Are internals leaking that should be private?
> - **Consistency**: Is naming and shape consistent with the rest of the public API and Zig stdlib conventions?
> - **Error contracts**: Are error types informative? Can callers distinguish and handle errors meaningfully?
> - **Ownership semantics**: Is it clear who owns memory at the call boundary? Are lifetimes obvious from the types?
> - **Composability**: Does the API compose well with Zig idioms (allocators, error unions, optionals)?
>
> [full diff]

## Step 3: Aggregate results

Once all agents return, **filter before reporting**. Drop any finding that:

- Has no concrete, unambiguous fix ("consider whether...", "you could also...")
- Is already enforced by the compiler or type system
- Is a naming quibble where the existing name is already clear
- Would generate disagreement between two reasonable engineers
- Requires context the reviewer doesn't have
- Would itself generate new findings if acted on (too vague to be actionable)

If you applied every surviving finding and re-ran the review, it should come back clean.

Then produce the report:

```
## Tiger Style
[must-fix findings, then suggestions — or "No issues found."]

## Zig Idioms
[findings — or "No issues found."]

## Deslop
[findings — or "No issues found."]

## Simplify
[findings — or "No issues found."]

## API Design
[findings — or "No issues found."]

---
**Summary:** X must-fix, Y suggestions across Z categories.
```

Keep findings concise — one line per issue where possible, with file and line context.
