# False Positive Prevention Guide

It is critical this guide is fully processed in a careful, systematic way.
It is used during the false positive section of the review, where avoiding
false positives is of utmost importance. Shift all bias away from efficient
processing and focus on following these instructions as carefully as possible.

## Core Principle

**If you cannot prove an issue exists with concrete evidence, do not report it.**

**Corollary:** For deadlocks, infinite waits, crashes, and data corruption,
"concrete evidence" means proving the code path is structurally possible — not
proving it will definitely execute on every run. A `pthread_cond_wait` with no
timeout and no fallback wake condition is a deadlock bug if the wake condition
depends on external events that can stop. Do not dismiss such bugs as "unlikely
in practice."

You must follow every instruction in every section. Do not skip steps.
Complete TASK POSITIVE.1 before completing the false positive check.

---

## Common False Positive Patterns

### 0. Context Preservation

- Confirm the full commit message or patch description is still in context.
  If not, reload it before proceeding.
- Do not proceed with false positive verification without this context ready.

### 1. Defensive Programming Requests

**Never suggest** defensive checks unless you can prove:
- The input comes from an untrusted source (user input, network, file, IPC).
- An actual code path exists where invalid data reaches this point.
- The current code can demonstrably fail.

```
Bad:  "Add a NULL check here for safety."
Bad:  "This should validate the index."
Good: "User input from funcA() can reach this without validation at file:line."
```

### 1.1 Failure to Handle Errors

**Never report** failure to handle errors unless:
- You can prove the error is possible in this context.
- You have confirmed that the function arguments used do not prevent the error.

### 2. API Misuse Assumptions

**Never report** issues based on theoretical API misuse unless you can prove:
- An actual calling path exists that triggers the issue.
- The function's naming or documentation does not clearly indicate usage
  constraints that rule out the path.

### 3. Unverifiable Assumptions

Require proof that author assertions are correct. Research claims in commit
messages, comments, and code, and prove them correct.

If the author makes claims without code evidence, treat them as unverified.
Design decisions must be justified by code or documentation.

**Report unless:**
- You found specific code that proves the author correct.
- You can verify all assumptions with concrete code paths.
- The behavior is proven correct, not just claimed.

### 3.1 Comment-Based Dismissals (MANDATORY)

When dismissing an issue because a comment says the code behaves a certain way,
you MUST verify against the actual implementation:

1. Read the function body, not just the comment.
   - Output: quote the actual implementation code.
2. Check for conditional compilation (`#ifdef`/`#else`).
   - Output: which branch applies and why.
3. Verify helper function behavior — if dismissing because "function X returns Y",
   read function X's implementation.
   - Output: quote the implementation showing the claimed guarantee.
4. If you cannot verify the comment matches the implementation, report the issue.

### 4. Locking False Positives

**Before reporting** a locking issue:
- Check ALL calling functions for held locks (output: list each caller and locks held).
- Trace up 2-3 levels to find lock context (output: full lock chain from entry to site).
- Verify the actual lock requirements (output: quote lock documentation or convention).

Common mistakes:
- Missing that a caller holds the required lock.
- Not recognizing that a lock-free path protects the access.
- Assuming all shared data needs traditional locks.

### 5. Use-After-Free Confusion

Distinguish carefully:
- Use-after-free: accessing freed memory → report.
- Use-before-free: using then freeing → do not report.
- Free-after-use: normal cleanup → do not report.

Verification:
- Trace the exact sequence: alloc@loc → use@loc → free@loc → use@loc.
- Check if object ownership was transferred.

### 6. Resource Leak Misconceptions

Not a leak if:
- Ownership was transferred to another subsystem.
- Object was added to a list or queue for later processing.
- Cleanup happens in a callback or deferred work.
- It is in test code and does not affect the system.

Verify by:
- Tracing object ownership changes (output: alloc@loc → stored in X@loc → freed by Y@loc).
- Checking for async cleanup mechanisms (output: cleanup callback location, or "none found").

### 7. Order Changes

**Do not report** order changes unless you can prove:
- A race condition is introduced.
- A dependency is violated.
- An ABBA deadlock pattern emerges.
- State becomes invalid.

### 8. Races

- Identify the EXACT data structure names and definitions (output: struct name and location).
- Identify the locks or barriers that should protect them (output: lock name and definition).
- Prove the race with CODE SNIPPETS (output: two code paths that can execute concurrently).

### 8.1 Race Dismissal: Full-Path Verification (MANDATORY)

When dismissing a race because "the code detects the invalid state and aborts,"
verify the ENTIRE instruction sequence between the race window and the recovery
point. A single abort check later in the function does not make earlier
dereferences safe.

Before accepting a race dismissal, answer ALL of these:

1. What exact instruction opens the race window? (function, file:line, what state becomes stale)
2. What exact instruction closes it? (function, file:line, synchronization mechanism)
3. What is the "graceful handler" you claim makes this safe? (function, file:line, detection method)
4. List every instruction between #1 and #3 that touches the contested resource.
   Are ALL of them safe if the resource was invalidated by the racing thread?
   (output: enumerate each instruction with verdict: safe/unsafe)

If you cannot affirmatively answer #4 for every intermediate instruction,
the dismissal is invalid. Report the race.

### 9. Performance Tradeoffs

Not a regression if:
- Lower performance was an intentional tradeoff.
- The commit message explains the performance impact.
- Simplicity or maintainability was prioritized.

### 10. Intentional Compatibility Choices

Leaving a stub interface (a file, endpoint, or API that returns a constant value)
is not a regression if that value was legal before the interface was deprecated.

Only report if the resource contract has been demonstrably broken.

### 11. Subjective Issues

Subjective concerns are not bugs — they are opinions. They can still be wrong.
Check them against the commit message, nearby code, and nearby comments, and
apply the "debate yourself" step before including them.

### 12. Uninitialized Variables

- Assigning to a variable is the same as initializing it.
- Passing an uninitialized variable to a function is fine if that function
  writes to it before reading it.
- Only report reading from uninitialized variables, not writing to them.
- Zero-allocating functions zero all bytes. Do not flag missing explicit
  initialization for fields whose zero value is correct.

### 13. Implicit Guard Conditions

Before reporting a NULL dereference:
- Review the `technical-patterns.md` NULL Pointer Dereference section.
- Check EVERY NULL pointer for guard conditions in callers, not just at the
  immediate call site.

### 14. Patch Series False Positive Removal

Large changes are broken into small logical units. Do not report work in
progress that is completed later in the series.

If a potential bug is simply work that is completed in a later patch, it is
a false positive.

However: each patch must compile and must not introduce new runtime bugs.
Intermediate patches may intentionally introduce performance issues fixed
later — but the commit message or comments must explain this.

If you identify a real regression that is fixed later in the patch series,
you must still report it, AND indicate you found the fix (provide its commit
hash and subject).

---

## TASK POSITIVE.1 Verification Checklist

Complete every step below and produce the required output.
Do not skip steps. Do not claim completion without producing output.

Before reporting ANY regression, verify:

**0.** For NULL pointer dereferences:
- Review the NULL Pointer Dereference section in `technical-patterns.md`.
- Output: "reviewed" or "not applicable — not a NULL dereference issue"

**1. Can I prove this path executes?**
- Find calling code that reaches this point.
  - Output: call chain with locations ("caller@file:line → target@file:line")
- Check for conditions that block the path.
  - Output: list conditions checked and their evaluation
- Verify the code is not dead or compile-time disabled.
  - Output: enabled by config/flag, or "always enabled"

**2. Is the bad behavior structurally possible?**
- Prove the code path exists and triggering conditions are not structurally impossible.
  - Output: step-by-step execution path showing the failure
- Prove the failure mode is concrete (crash, deadlock, corruption, leak), not
  just "increases risk."
  - Output: the specific failure mode and triggering condition
- Note: A deadlock or crash that depends on runtime conditions (timing, memory
  pressure, shutdown state) is a real bug if the code has no structural prevention
  (timeout, fallback condition, bounded retry). Do not dismiss as "unlikely."

**3. Did I check the full context?**
- Examine calling functions (2-3 levels up).
  - Output: list each caller checked with a representative line
- Check initialization and cleanup paths.
  - Output: init/cleanup functions examined with locations
- Verify subsystem conventions.
  - Output: conventions found and whether code follows them

**4. Is this actually wrong?**
- Check for intentional design choice.
  - Output: quote commit message or comment if it explains intent, else "no explanation found"
- Check for documented limitation.
  - Output: quote documentation if found, else "not documented"
- Verify this is not test code where imperfection is acceptable.
  - Output: "production code" or "test code — severity adjusted"
- Confirm the bug exists today, not just if code changes later.
  - Output: current triggering path or "theoretical future issue only"
  - Note: "theoretical" means the path cannot be reached today. A bug that
    depends on runtime conditions is not theoretical.

**5. Did I check the commit message and surrounding comments?**
- Read the entire commit message.
  - Output: quote any text explaining this behavior, or "no explanation found"
- Read surrounding code comments.
  - Output: quote relevant comments, or "no relevant comments"

**6. When complex multi-step conditions are required for the bug:**
- Prove these conditions are actually simultaneously possible.
  - Output: code path showing each condition can be true at the same time

**7. Did I hallucinate a problem that does not exist?**
- Verify the bug report matches the actual code.
  - Output: quote the exact code snippet from the file with file:line
- Reread the file and confirm code matches your analysis.
  - Output: file:line and verbatim code
- Check your math (division by zero requires zero in denominator, etc.).
  - Output: arithmetic verification or "no arithmetic involved"

**8. Did I check for future fixes in the same patch series?**
- Search forward in the git range if one was provided (never search backwards).
  - Output: commits checked or "no git range provided"
- If fix found later in series:
  - Output: "found fix in [commit] — reporting as real bug with later fix" or "no fix found"

**9. If dismissing based on comments or documentation, verify the implementation:**
- Did you read the actual function implementation, not just the comment?
  - Output: quote the implementation code that proves the comment is accurate
- Does the function have `#ifdef`/`#else` branches with different behavior?
  - Output: list config options that affect behavior, state which applies
- Did you verify helper functions behave as their comments claim?
  - Output: quote helper implementation or "no helper functions involved"
- If you cannot verify implementation matches documentation, do NOT dismiss.
  - Output: "implementation verified" or "cannot verify — reporting issue"

**10. Debate yourself**

Do these two steps in order:

**10.1** Pretend you are the author. Think hard and try to prove the review is incorrect.
- Check for hallucinations or invented information.
- For NULL safety, ask as the author:
  - Did the reviewer search for similar code in the same subsystem accessing this pointer?
  - If reporting a missing NULL check, did they explain why other code HAS that check?
  - Did they verify lifecycle dependencies or just analyze syntactically?
  - Is there semantic coupling between a guard condition and pointer validity they missed?
- For locking, ask as the author:
  - Did the reviewer check what locks my caller holds?
  - Is there a lock held higher in the call chain they missed?
- For resource leaks, ask as the author:
  - Did the reviewer trace ownership transfer?
  - Did they check for async cleanup mechanisms?
- For all issues, ask as the author:
  - Did they check if this is intentional based on commit message or comments?
  - Did they verify the conditions for the bug are actually possible?
  - Are they confusing a structurally possible bug with a defensive programming suggestion?
- Output: strongest argument against reporting this bug.
- IMPORTANT: "unlikely in practice" is NOT a valid argument against a deadlock,
  crash, or data corruption. Only "structurally impossible" is valid.

**10.2** Now pretend you are the reviewer. Think hard and address the author's arguments.
- Address each author argument with code evidence.
- Output: code evidence refuting the author, or "cannot refute with code — likely false positive"

---

## Mandatory Validation

- If any Output requirement above is blank or skipped, repeat that step.
- If you cannot produce code evidence for your conclusion, the bug is likely a false positive.

---

## Patch Series

- You may only look forward in git history (toward newer commits), never backward.
- Never invent other methods to look forward in git history.
- If the prompt included a git commit range, look forward through that range for
  later patches that resolve the bug found.

---

## Special Cases

### Test Code
- Memory leaks in test programs → usually acceptable.
- File descriptor leaks in tests → usually acceptable.
- Unless it crashes or hangs the system → always report.

### Assertions and Abort Calls
- Removing `assert()`, `WARN_ON()`, `BUG_ON()`, `abort()` → not a regression.
- Removing `static_assert()` / `BUILD_BUG_ON()` → not a regression.
- Unless removing a critical runtime check that catches real corruption → report.

### Reverts
- Focus on new issues introduced by the revert, not the original bug.
- Do not re-report the problem the revert was meant to fix.

---

## Final Filter

Before adding any finding to the report, answer all four:

1. **Do I have proof, not just suspicion?** Code snippets showing all components
   required to trigger the bug count as proof — ONLY if the conditions are proven
   possible. Existing `assert()` calls do NOT count as proof.
2. **Would an expert see this as a real issue?**
3. **Is this worth the developer's time?**
4. **Am I suggesting defensive programming, or reporting a concrete bug?**
   - Defensive: "add a NULL check here for safety" → discard.
   - Concrete: "this pthread_cond_wait has no timeout and no fallback wake" → report.

If you cannot answer yes to all four, investigate further or discard.

---

## Remember

- False positives waste everyone's time.
- Missed bugs also waste everyone's time — a deadlock in production is worse
  than a false positive in review.
- Developers are experts — but experts miss bugs too, especially subtle
  interactions between subsystems (thread pools, event loops, shutdown ordering).
- Real bugs have real proof — proof means the code path exists and has no
  structural prevention, not that the bug will fire on every execution.
