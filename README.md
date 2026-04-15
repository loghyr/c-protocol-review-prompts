# c-protocol-review-prompts

Generic, project-agnostic C/C++ code review prompts for AI agents. Designed for
systems and protocol code: network stacks, distributed storage, kernel-adjacent
daemons, and anything else that lives close to the wire.

Derived in part from [masoncl/review-prompts](https://github.com/masoncl/review-prompts)
(kernel-focused); this suite removes kernel-specific machinery and adds protocol
and concurrency patterns for userspace systems code.

## What Is This

A set of Markdown files that teach an AI agent how to review C/C++ code rigorously:

- **Review protocol** — four-task analysis (context, categorize, analyze, report)
  with a reachability gate before every finding
- **False-positive elimination** — 10-step TASK POSITIVE.1 checklist adapted from
  the masoncl methodology; cuts noise before output is produced
- **Pattern library** — per-domain pitfall references loaded on demand (locking,
  RCU, ref-counting, memory safety, atomics, error handling, async state, clocks)
- **Role system** — Planner / Programmer / Reviewer definitions for single-agent
  or multi-agent workflows
- **Slash commands** — `/c-review`, `/c-debug`, `/c-verify` for Claude Code

## Directory Layout

```
c-protocol-review-prompts/
  AGENTS.md               ← AI agent entry point (auto-loaded by Claude Code)
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
    c-review.md           ← /c-review  — full patch or file review
    c-debug.md            ← /c-debug   — crash and stack trace analysis
    c-verify.md           ← /c-verify  — verify a specific claim or change
```

## Quick Start (Claude Code)

Clone this repo, then point your project's `AGENTS.md` or `CLAUDE.md` at it:

```
/c-review <commit-hash>
/c-review <file1.c> <file2.h>

/c-debug <stack trace or sanitizer output>

/c-verify "does bar() hold the lock before calling baz()"
```

The skill in `skills/c-cpp-review.md` activates automatically when the working
directory contains C or C++ source files.

## Using with Other AI Agents

Any agent that can read Markdown files can use this suite. The suggested load
order for a review session:

1. `technical-patterns.md` — always load first
2. `review-core.md` — load for patch or commit review
3. Pattern files — load on demand based on what the code touches (see AGENTS.md)
4. `false-positive-guide.md` — run before finalizing any finding

The files are self-contained and make no assumptions about the agent framework,
project build system, or OS.

## Adding Project-Specific Rules

Do not modify the files in this repo. Instead:

1. Create a `patterns/project-name.md` in your project tree.
2. Reference it from your project's `AGENTS.md` or `CLAUDE.md`.
3. The core files here remain unchanged and reusable across projects.

## Philosophy

- **Never report without proof.** Suspicion is not a finding.
- **Trace full execution paths.** Never assume a bug from a return type or comment.
- **Bidirectional analysis.** Ask both "does new code follow the rule?" and "does
  this change break assumptions other code has about the rule?"
- **CHANGE-N granularity.** One category per loop, per return, per lock change —
  not one category per function.
- **Reachability first.** If the changed code cannot be reached by the stated
  workload, report that immediately and stop.
- **Debate yourself.** Before reporting any finding, argue the author's side, then
  argue the reviewer's side. Report only if the reviewer's argument wins.

## Relationship to masoncl/review-prompts

This suite adapts the core methodology from masoncl/review-prompts and removes
kernel-specific details (Fixes: tags, lore threads, subsystem maintainer guides,
`WARN_ON`/`BUG_ON` semantics, `rcu_read_lock_bh`). What remains applies equally
to userspace protocol daemons and kernel modules.

Additions not in the upstream repo:
- Protocol-specific patterns (async state transfer, clock selection)
- Lock-free hash table lifecycle (urcu/liburcu patterns)
- `call_rcu` callback constraints
- Iterator-before-put requirement in hash table traversal
- Extended false-positive guide adapted for userspace (no `might_sleep`, no
  `lockdep_assert_held`)

## License

CC0 1.0 Universal — see [LICENSE](LICENSE). No rights reserved.
