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
4. Applies the **bidirectional** reachability gate before reporting any
   issue — forward (new code reachable?) and backward (old code newly
   reached?).  See `review-core.md` Task 2 Step 0.
5. Runs bidirectional rule analysis for every CHANGE.
6. Loads matched pattern files (`patterns/locking.md`, etc.).  For
   extraction refactors, backend ports, stub removals, or feature-flag
   flips, load `patterns/reachability-change.md`.
7. Eliminates false positives via TASK POSITIVE.1 and the "debate yourself" step.
8. Produces plain-text findings (BLOCKER / WARNING / NOTE).

## Smoke-Run Supplement for Reachability Changes

Static review does not catch runtime-only bugs in newly-reachable
code. For these commit shapes, pair `/c-review` with a smoke run:

- Platform or backend port (new OS, new I/O model, new architecture).
- Extraction refactor that activates previously-stubbed code.
- First-time enabling of a feature-flag default.

The minimum smoke is in `patterns/reachability-change.md` §5:

1. Binary starts and enters its main loop on the new target.
2. Simplest client round-trip succeeds.
3. Non-trivial payload round-trips intact (e.g., 16 MB + sha256 match).
4. Two concurrent clients produce independent results.
5. SIGTERM produces clean shutdown; no fd / memory leak within 5 s.

Attach smoke evidence (log excerpt, sha256 comparison, `fstat` output)
to the review.  A reviewer who signs off without smoke evidence on a
reachability-change commit is signing off on the *idea* of the
change, not its correctness.

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
