---
name: c-cpp-review
description: |
  Generic C/C++ code reviewer. Activates automatically when the working directory
  contains C or C++ source files. Provides deep regression analysis, CHANGE-N
  categorization, bidirectional rule checking, and false positive elimination.
  Does not assume any particular project framework, OS, or build system.
invocation_policy: automatic
detection_triggers:
  - "*.c"
  - "*.cpp"
  - "*.cc"
  - "*.cxx"
  - "*.h"
  - "*.hpp"
  - "*.hh"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
---

# C/C++ Review Skill

## ALWAYS LOAD FIRST

1. `technical-patterns.md` — C/C++ pitfall reference (mandatory)
2. `review-core.md` — analysis protocol (mandatory for patch/commit review)

## Capabilities

### Patch and Commit Review

When asked to review a patch, commit, or PR:

1. Load `review-core.md` and follow the complete 4-task protocol.
2. Load pattern files matched by CHANGE-N categories:
   - Locking changes → `patterns/locking.md`
   - RCU / lock-free → `patterns/rcu-and-lockless.md`
   - Object lifecycle → `patterns/ref-counting.md`
   - Memory operations → `patterns/memory-safety.md`
   - Atomic access → `patterns/atomics.md`
3. Run every potential issue through `false-positive-guide.md` before reporting.

### Crash and Stack Trace Analysis

When asked to debug a crash, warning, or stack trace:

1. Load `technical-patterns.md`.
2. Use the crash information as entry points into call-graph analysis.
3. Trace the execution path that led to the crash, verifying each step with
   actual code rather than assumptions.
4. Produce a debug report with root cause, evidence, and suggested fix.

### Pattern-Based Code Change

When asked to make a repeatable code change (rename, API update, parameter
addition, boilerplate removal):

1. Identify all call sites before modifying any.
2. Apply the change consistently — do not leave partial updates.
3. Verify the build after changes.

### Role Assignment

When operating in a multi-role context, load `roles.md` and follow the
Planner → Programmer → Reviewer sequence.

When assigned a specific role (e.g., "act as Reviewer"), read only that role's
responsibilities and follow them exclusively for this session.

## Detection Triggers (load additional context when matched)

| Trigger | Load |
|---------|------|
| `pthread_mutex`, `PTHREAD_MUTEX`, `std::mutex`, `std::lock_guard` | `patterns/locking.md` |
| `atomic_`, `std::atomic`, `__atomic_`, `_Atomic` | `patterns/atomics.md` |
| `rcu_read_lock`, `call_rcu`, `synchronize_rcu`, `rcu_assign_pointer` | `patterns/rcu-and-lockless.md` |
| `refcount_t`, `kref`, `std::shared_ptr`, `ref_get`, `ref_put` | `patterns/ref-counting.md` |
| `malloc`, `calloc`, `realloc`, `free`, `new`, `delete`, `mmap` | `patterns/memory-safety.md` |
| `volatile` used with shared data | `patterns/atomics.md` |

## Output

- Patch reviews produce plain-text findings grouped by BLOCKER / WARNING / NOTE.
- Debug sessions produce a root-cause analysis with evidence.
- Both formats are suitable for inclusion in bug trackers or code review tools.
- Never use markdown headings or bullet points in finding text intended for
  plain-text communication.
