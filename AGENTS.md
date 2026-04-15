# C/C++ Reviewer Skill Suite

This directory provides a generic, project-agnostic C/C++ code review system. It is
designed to be usable by any AI agent without project-specific tuning.

## Directory Layout

```
c-protocol-review-prompts/
  AGENTS.md               ← this file
  roles.md                ← Planner / Programmer / Reviewer role definitions
  review-core.md          ← Main review protocol (CHANGE-N analysis + tasks)
  technical-patterns.md   ← C/C++ pitfalls: NULL, resources, atomics, loops
  false-positive-guide.md ← FP elimination checklist (TASK POSITIVE.1)
  skills/
    c-cpp-review.md       ← Skill definition (auto-detects C/C++ projects)
  patterns/
    locking.md            ← Lock ordering, ABBA, spinlock, condvar, TOCTOU
    rcu-and-lockless.md   ← RCU, lock-free hash tables, call_rcu constraints
    ref-counting.md       ← Get/put balance, use-after-put, error unref
    memory-safety.md      ← Allocator mismatches, overflows, double-free
    atomics.md            ← C11 atomics, memory ordering, data races
    error-handling.md     ← Return value contracts, errno, cleanup paths
    async-state-transfer.md ← State machine parking, inbound RPC resume, fencing
    clock-selection.md    ← CLOCK_REALTIME vs CLOCK_MONOTONIC, abstraction rules
  slash-commands/
    c-review.md           ← /c-review  — invoke the review protocol
    c-debug.md            ← /c-debug   — analyze a crash or stack trace
    c-verify.md           ← /c-verify  — verify a specific claim or change
```

## Quick Start

**Reviewing a patch or commit:**
```
/c-review <commit-hash>
/c-review <file1.c> <file2.h>
```

**Debugging a crash:**
```
/c-debug <stack-trace or error description>
```

**Verifying a specific change:**
```
/c-verify <change description>
```

## Skill Auto-Loading

The skill in `skills/c-cpp-review.md` activates automatically when the working
directory contains C or C++ source files. It loads `technical-patterns.md` and
`review-core.md` on every activation.

## Role System

Three roles are defined in `roles.md`. They can operate independently or as a team:

| Role       | Primary Responsibility                                     |
|------------|------------------------------------------------------------|
| Planner    | Decompose work, design architecture, resolve blockers      |
| Programmer | Implement, write tests, maintain build cleanliness         |
| Reviewer   | Analyze regressions, enforce standards, flag concurrency   |

When working alone, an agent adopts all three roles and sequences them:
Plan → Implement → Review.

When multiple agents collaborate, assign one role per agent instance. The Reviewer
always runs last, on completed code.

## Review Methodology

The review protocol in `review-core.md` follows a four-task structure:

1. **Context** — gather definitions, call graphs, types for all changed code
2. **Categorize** — produce CHANGE-N categories per function (control flow, resource
   management, locking, return value changes)
3. **Analyze** — reachability gate first, then bidirectional rule checks per CHANGE
4. **Report** — eliminate false positives, produce actionable findings

Pattern files under `patterns/` are loaded on-demand when a CHANGE category matches.

## Integration with masoncl/review-prompts

This suite integrates methodology from `github.com/masoncl/review-prompts`:
- Reachability gate (verify code path is reachable before reporting)
- CHANGE-N categorization (fine-grained per function)
- Bidirectional rule analysis (forward + reverse)
- TASK POSITIVE.1 verification checklist (10-step FP elimination)
- "Debate yourself" final check

Kernel-specific details (Fixes: tags, lore threads, subsystem guides, WARN_ON/BUG_ON
semantics) are not included — see the upstream repo for kernel-specific variants.

## Adding Project-Specific Rules

To add project-specific rules without modifying this directory:

1. Create a `patterns/project-name.md` file in your project tree.
2. Reference it from your project's `AGENTS.md` or `CLAUDE.md`.
3. The core files here remain unchanged and project-agnostic.

## Philosophy

- **Never report without proof.** Suspicion is not a finding.
- **Trace full execution paths.** Never assume a bug from a return type or comment.
- **Bidirectional analysis.** Ask both "does new code follow the rule?" and "does
  this change break assumptions other code has about the rule?"
- **CHANGE-N granularity.** One category per loop, per return, per lock change —
  not one category per function.
- **Reachability first.** If the changed code cannot be reached by the stated
  workload, report that immediately and stop.
