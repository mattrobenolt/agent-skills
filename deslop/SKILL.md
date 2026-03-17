---
name: deslop
description: Remove AI-generated slop from code changes. Use when the user says "deslop", "remove slop", "clean up AI comments", or wants to strip unnecessary AI-added comments from a branch's diff. Works with any language, optimized for Go and Zig codebases.
---

# Deslop

Check the diff against main and remove all AI-generated comment slop introduced in this branch.

## What to remove

- Comments that explain obvious code (`// initialize the variable`, `// return the result`)
- Section-divider comments a human wouldn't write (`// --- Helper Functions ---`)
- Redundant comments that restate the function/variable name (`// processItems processes items`)
- Comments that narrate control flow (`// check if error`, `// loop through items`)
- TODO/FIXME comments added by AI that weren't requested
- Doc comments on unexported or obvious functions that don't need them
- Any comment style inconsistent with the rest of the file

## What to keep

- Comments explaining *why*, not *what*
- Comments matching the existing style and density of the file
- License headers
- Compiler directives and build tags
- Comments the user wrote (present before this branch)

## Process

1. Get the diff: `git diff main`
2. For each changed file, read the full file to understand existing comment style and density
3. Remove slop comments from the branch's changes only — do not touch pre-existing code
4. Report a 1-3 sentence summary of what was changed
