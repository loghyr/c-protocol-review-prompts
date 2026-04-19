# C/C++ Patch Analysis Protocol

This is deep regression analysis, not a quick sanity check.
The working assumption is that the patch has bugs, including in its comments
and commit message. Every change, comment, and assertion must be proven correct —
otherwise report as a regression.

If given a git range, print a numbered list of commits at the start of output,
oldest first: `#. <hash> <subject>`. Mark the commit under analysis with `*`.

Analyze only the instructed commit; consider the series only when looking forward
for fixes to regressions found.

---

## Analysis Philosophy

- New APIs are checked for consistency and ease of use.
- Any deviation from C/C++ best practices is reported as a regression.
- Verify every comment against the actual implementation. Comments can be wrong.
- Check `#ifdef`/`#else` branches — the same comment may describe both sides
  with different semantics.

---

## FILE LOADING (ALWAYS LOAD FIRST)

1. `technical-patterns.md` — core C/C++ pitfall reference
2. `false-positive-guide.md` — FP elimination checklist
3. Pattern files from `patterns/` matched by CHANGE category (see Task 1B)

---

## PATTERN DETECTION (check BEFORE Task 0)

Scan the diff for:
- Mutex, spinlock, rwlock, condvar operations → load `patterns/locking.md`
- RCU, seqlock, lock-free data structures → load `patterns/rcu-and-lockless.md`
- Refcount increment/decrement, object lifecycle → load `patterns/ref-counting.md`
- malloc/free, new/delete, pointer arithmetic, array indexing → load `patterns/memory-safety.md`
- atomic_t, stdatomic, __atomic builtins, volatile for concurrency → load `patterns/atomics.md`
- System calls, I/O (read/write/open/close/fsync), library calls with error returns → load `patterns/error-handling.md`
- clock_gettime, timeouts, timers, leases, sleep, pthread_cond_timedwait → load `patterns/clock-selection.md`
- task_pause/resume, coroutine suspend, async state transfer → load `patterns/async-state-transfer.md`
- Extraction refactors (moving code across TUs), backend/platform ports, stub removal, enabling a previously-disabled feature flag, or removing `__attribute__((unused))` / `static` / `#if 0` that had been hiding code from the linker → load `patterns/reachability-change.md`

---

## Task 0: CONTEXT MANAGEMENT

Discard non-essential details after each task to manage context limits.
Keep all context for Task 4 reporting if regressions were found.

1. Plan the initial context-gathering phase:
   - Read the full diff line-by-line before making any tool calls.
   - Understand the commit's purpose before analyzing any hunk.
   - Never just read the commit message and jump ahead.
   - If you spot suspect bugs while reading, record them in a todo list,
     but do not begin full analysis until Task 2.
   - Document the commit's intent before analyzing patterns.

2. Classify the kinds of changes introduced by the diff.

3. Plan the entire context-gathering phase. Unless running out of context,
   load all required context once and only once.

---

## RESEARCH TASKS

### TASK 1: Context Gathering

**Goal:** Build complete understanding of changed code.

1. Identify all changed functions and types.
2. Get full definitions for every identified item — never work from diff
   fragments alone. Always prefer full context over partial hunks.
3. Trace call relationships: at least one level up and one level down.
4. Always trace cleanup paths and error handling separately.
5. If a function was deleted, check its callers to confirm none remain.

### TASK 1B: Categorize Changes

Do not start analyzing any change until context gathering and categorization
are complete. Do not skip steps even if you think you spotted a bug already.

For each modified function, create separate CHANGE-N categories for:

- **Control flow:** one category PER loop, one PER changed return/break/continue.
  Inner and outer loops get separate categories — never combine them.
- **Return value / condition changes:** changes that alter what callers see.
  These often have side effects elsewhere in the call stack.
- **Resource management (allocation/free):** one category per allocation site,
  one per free site.
- **Resource management (initialization):** object construction, field init.
- **Locking:** one category per lock acquire/release/upgrade.

Add each category to your todo list. Call them CHANGE-1, CHANGE-2, etc.

### TASK 1C: Print CHANGE Categories

Output each category:

```
CHANGE-N: <short description>, <representative line of code>
```

---

### Task 2: Analyze Changes for Regressions

**Step 0 — Reachability Gate (MANDATORY, before all other Task 2 work):**

The gate is bidirectional. Apply both forward and backward checks.

**Forward — is the NEW code reachable?**  Verify the changed code
paths are reachable by the workloads or consumers described in the
commit message. Check:
- Feature flags and compile-time options that might disable the path.
- Protocol constraints that prevent execution.
- Init ordering that might mean the code never runs in practice.

If the code path cannot execute for the stated use case, report this
immediately — it is a show-stopper that supersedes detailed analysis.

**Backward — does the change make any OLD code reachable for the first
time?**  This is critical for extraction refactors, backend/platform
ports, stub-removal commits, and feature-flag flips. Ask:
- Does this commit add a caller for a function that previously had
  none in the active build?
- Does this commit remove a `-ENOSYS` stub, an `#if 0`, a
  `__attribute__((unused))`, or a `static` that had been hiding code?
- Does this commit activate a compile-time-guarded region that the
  CI has never exercised?

If yes, the pre-existing code in that path is in scope for review.
It hasn't been exercised by any test suite before this commit; its
bugs have been dormant. Treat it as if it were new code.

Load `patterns/reachability-change.md` and apply its checklist (§6).
Record the latent code in scope explicitly:

```
REACHABILITY: confirmed
NEWLY-REACHABLE CODE (in scope):
  - lib/io/backend_kqueue.c:io_request_accept_op — called from main() with ci=NULL
  - lib/io/backend_kqueue.c:kqueue_arm_heartbeat_timer — called from io_schedule_heartbeat
```
or
```
REACHABILITY: blocked — <reason>
```

**Step 1 — Callstack Analysis:**

For non-trivial changes, trace the full call stack for each CHANGE:
- Verify every comment matches actual behavior.
- Verify commit message claims are accurate.
- Question all design decisions.
- Check naming conventions and usability of new APIs.
- Check against C/C++ best practices.

**Step 2 — Bidirectional Rule Analysis:**

For every pattern rule that applies to a CHANGE:

- **Forward:** Does the new code satisfy the rule?
- **Reverse:** Does the change break an invariant that OTHER code depends on?
  If the patch changes how a lock is held, or changes a function's return
  semantics, check what callers expect — the bug may be in the callers, caused
  by the patch.

**Step 3 — Named Function Cross-Reference:**

If any pattern or guide names specific functions ("call X before Y",
"must hold lock Z when calling W"), check whether the patch touches the
invariant those rules document. If so, analyze all callsites.

---

### TASK 3: Verification

**Goal:** Eliminate false positives, confirm regressions.

1. If no regressions found: mark complete, proceed to Task 4.
2. If regressions found:
   - Load `false-positive-guide.md`.
   - Apply every check from the guide.
   - Run TASK POSITIVE.1 in full for each potential issue.
   - Only mark complete after all verification done.

---

### TASK 4: Reporting

**Goal:** Produce clear, actionable findings.

If no regressions found:
- Check: were there any issues flagged during analysis?
- If none: provide a summary and note any context limitations.

If regressions found:
- Clear context not related to the regressions.
- Format findings as plain text suitable for a code review or mailing list.
  Do not use markdown headings, bold, or bullet characters in the findings
  text itself — plain prose and code blocks only.
- Use the severity format from `roles.md` (BLOCKER / WARNING / NOTE).
- Never write `REGRESSION:` in all-caps as a label. Describe the issue
  in plain language.
- Never include issues identified as false positives in the report.

**Per-finding format:**

```
<severity>: <component>: <short subject line>

<Quoted code snippet, with file and line number.>

<Prose explanation: what is wrong, why it matters, what can go wrong.>

<Suggested fix if obvious and correct.>
```

---

### MANDATORY COMPLETION VERIFICATION

After writing the report, verify:
- No markdown formatting leaked into the findings text.
- No all-caps analysis tags (`REGRESSION:`, `BUG:`, `CRITICAL:`) that violate
  plain-text conventions.
- Every finding has a code snippet with a file:line reference.
- Every finding went through TASK POSITIVE.1 and the "debate yourself" step.

---

## OUTPUT FORMAT

Always conclude with:

```
FINAL REGRESSIONS FOUND: <number>
False positives eliminated: <number>
Assisted-by: <agent name>:<model version>
```
