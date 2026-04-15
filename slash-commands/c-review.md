# /c-review — C/C++ Code Review

Perform a deep regression analysis on a C/C++ commit, patch, or set of files.

## Usage

```
/c-review <commit-hash>
/c-review <commit-hash>..<commit-hash>
/c-review <file1.c> [file2.h ...]
/c-review  (with a patch already in context)
```

## What This Does

1. Loads `technical-patterns.md` and `false-positive-guide.md`.
2. Reads the full diff line-by-line.
3. Produces CHANGE-N categories (one per loop, return, lock, allocation).
4. Applies the reachability gate before reporting any issue.
5. Runs bidirectional rule analysis for every CHANGE.
6. Loads matched pattern files (`patterns/locking.md`, etc.).
7. Eliminates false positives via TASK POSITIVE.1 and the "debate yourself" step.
8. Produces plain-text findings (BLOCKER / WARNING / NOTE).

## Arguments

- **Single commit hash:** Review only that commit. Read the full diff.
- **Range (A..B):** Print numbered commit list; analyze only the commit specified
  or the latest if none specified. Consider the range when looking forward for fixes.
- **File list:** Treat listed files as the changed set. Review all changes in them.
- **No argument:** Look for a patch, diff, or PR description in the current context.

## Output

Plain text, suitable for a bug tracker or code review tool:

```
BLOCKER: <component>: <short subject>
<file:line, quoted code>
<Explanation and suggested fix.>

WARNING: <component>: <short subject>
<Evidence.>

NOTE: <component>: <short subject>
<Observation.>

FINAL REGRESSIONS FOUND: N
False positives eliminated: M
```

## When to Use

- Before merging a branch.
- When asked to review a patch from a collaborator.
- When analyzing a commit for correctness after a reported bug.

The Reviewer role in `roles.md` provides the complete responsibility list.
This command invokes the Reviewer role automatically.
