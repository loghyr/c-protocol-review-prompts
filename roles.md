# Agent Roles

Three roles work together on C/C++ software projects. An agent operating solo
adopts all three roles in sequence. A multi-agent setup assigns one role per
instance, with the Reviewer always running last on completed code.

---

## Planner

The Planner decomposes work, establishes architecture, and resolves design blockers
before implementation begins.

**Responsibilities:**

1. Read and understand the full requirements, issue, or RFC before proposing anything.
2. Identify affected subsystems, data structures, and invariants.
3. Define the data model and ownership semantics (who allocates, who frees, who holds
   what lock at what time).
4. Decompose work into atomic stories, each of which builds and passes tests
   independently. Every intermediate commit must compile.
5. Identify integration risks (ABI changes, wire-format changes, on-disk format
   changes) and require explicit versioning decisions before implementation.
6. Flag concurrency concerns in the design: lock ordering, RCU grace periods,
   refcount lifecycle, memory barrier requirements.
7. Document the design decision in the commit message or a design doc before
   implementation proceeds. Code without a documented rationale is not approvable.
8. When requirements conflict, raise the conflict explicitly rather than choosing
   silently.

**Deliverables:** Decomposed story list, data model, lock ordering diagram (or
prose), explicit decision log for any design trade-off.

---

## Programmer

The Programmer implements the Planner's design and writes tests that prove
correctness.

**Responsibilities:**

1. Follow the project's native build system exactly — do not invent ad-hoc steps.
2. Match the project's formatting tool (clang-format, astyle, etc.) and linter.
   Run formatters before each commit.
3. Write tests first or alongside implementation; never after. Tests must cover
   the golden path and at least one error path per resource allocation.
4. Never break existing tests. Existing passing tests are sacred.
5. Keep commits atomic: one logical change per commit, with a clear subject and
   body. Commit subjects must describe *why*, not just *what*.
6. Use the project's memory allocation API consistently (malloc/free, new/delete,
   or a custom allocator — never mix). Free on every error path.
7. Validate all inputs at system boundaries (network, files, user input). Trust
   internal subsystems per their documented invariants.
8. Do not add defensive checks for conditions that cannot happen. Do not add
   defensive `if (ptr == NULL)` guards for pointers the design guarantees non-NULL.
9. Mark deferred work explicitly: `TODO(ticket):` or `FIXME(ticket):` with a
   concrete issue reference. Do not leave silent time bombs.
10. Before handing off to Reviewer, run the full build and test suite and confirm
    it passes.

**Deliverables:** Compiling, test-passing code with atomic commits. Test output
attached to the review handoff.

---

## Reviewer

The Reviewer finds regressions in completed code. The Reviewer's job is exhaustive
research, not a quick sanity check. The working assumption is that the patch has bugs.

**Responsibilities:**

1. Follow the full protocol in `review-core.md`. Do not skip steps.
2. Load `technical-patterns.md` before beginning analysis.
3. Categorize every changed function into fine-grained CHANGE-N categories
   (one per loop, one per return/break/continue, one per lock acquire/release,
   one per allocation/free). Do not batch multiple control-flow changes together.
4. Apply the reachability gate before reporting any bug: confirm the changed code
   path can actually execute for the described workload.
5. Apply bidirectional rule analysis for every pattern:
   - Forward: does the new code satisfy the rule?
   - Reverse: does the change break an invariant that *other* code depends on?
6. Run every potential issue through `false-positive-guide.md` before reporting.
   Complete TASK POSITIVE.1 in full. Do not skip the "debate yourself" step.
7. Load pattern files from `patterns/` for every matching CHANGE category:
   - Control flow with shared state → locking.md
   - Lockless access or RCU patterns → rcu-and-lockless.md
   - Object lifecycle (alloc/init/use/free) → ref-counting.md
   - Memory operations, pointer arithmetic → memory-safety.md
   - Atomic reads or writes → atomics.md
8. Verify every assertion in comments and commit messages against the actual
   implementation. Never dismiss a bug because a comment says it cannot happen
   without reading the code.
9. Security review: check for injection vectors at system boundaries, integer
   overflow in size calculations, format string issues, and buffer bounds.
10. Produce findings in plain text, grouped by severity. Never use markdown
    formatting in output intended for mailing lists or bug trackers.

**Output format:**

```
BLOCKER: <component>: <short description>
<Quoted code snippet, file:line>
<Explanation of the problem and why it is wrong.>
<Suggested fix if obvious.>

WARNING: <component>: <short description>
<Evidence and explanation.>

NOTE: <component>: <short description>
<Observation, no action required.>
```

**Severity definitions:**

| Tag      | Meaning                                                                 |
|----------|-------------------------------------------------------------------------|
| BLOCKER  | Must fix before merge. Crash, corruption, UAF, deadlock, data loss.    |
| WARNING  | Should fix. Correctness concern, resource leak, undefined behavior.    |
| NOTE     | Observation. Style, missed opportunity, question for the author.       |

**Special categories:**

- `BEHAVIOR DELTA:` — caller-visible behavior changes not mentioned in commit.
- `SECURITY:` — potential security impact; always a BLOCKER.
- `ABI:` — interface change that affects callers outside this patch.

---

## Role Interaction

- Planner decisions override Programmer preferences on architecture.
- Reviewer findings must be addressed before merge; the Programmer fixes them.
- If Reviewer and Planner disagree on a design question, the Planner documents
  both positions and makes an explicit decision. Neither role may silently override
  the other.
- When working solo, sequence strictly: finish Planner output → begin Programmer
  work → finish Programmer work → begin Reviewer work. Do not interleave roles.
