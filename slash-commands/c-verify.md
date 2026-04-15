# /c-verify — C/C++ Claim and Change Verification

Verify a specific claim about code correctness, or spot-check a targeted
change without running the full patch review protocol.

## Usage

```
/c-verify <claim or question>
/c-verify "is the refcount always balanced in foo()"
/c-verify "does bar() hold the lock before calling baz()"
/c-verify <function-name> [file]
```

## What This Does

1. Loads `technical-patterns.md`.
2. Reads the relevant code sections.
3. Traces the execution paths that are relevant to the claim.
4. Applies the TASK POSITIVE.1 checklist from `false-positive-guide.md` to
   confirm or deny the claim with concrete evidence.
5. Reports: verified correct, verified incorrect, or insufficient evidence.

## Difference from /c-review

`/c-review` analyzes ALL changes in a commit or file set exhaustively.

`/c-verify` analyzes ONE specific claim or concern. Use it when:
- A code reviewer raised a concern and you want to verify it.
- You want to confirm an invariant holds before a refactor.
- You are not sure whether a specific pattern (locking, ownership, etc.) is correct.
- You want to sanity-check a single function without a full review.

## Output

```
Claim: <restated claim being verified>

Verdict: CORRECT | INCORRECT | UNCERTAIN

Evidence:
  <file:line — quoted code>
  <Explanation of what the code actually does>

If INCORRECT:
  <What would need to change to make the claim true>

If UNCERTAIN:
  <What additional information is needed to determine correctness>
```

## Examples

```
/c-verify "thread_pool_submit() is safe to call from a signal handler"
/c-verify "cache_evict() always releases the cache lock before returning"
/c-verify "the refcount in session_alloc() is never incremented past INT_MAX"
```

## Pattern Loading

`/c-verify` loads pattern files on demand based on the claim:
- Claims about locking → `patterns/locking.md`
- Claims about ownership → `patterns/ref-counting.md`
- Claims about thread safety or atomics → `patterns/atomics.md`
- Claims about memory → `patterns/memory-safety.md`
