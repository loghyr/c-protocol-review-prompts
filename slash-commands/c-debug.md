# /c-debug — C/C++ Crash and Bug Analysis

Analyze a crash, stack trace, assertion failure, or unexpected behavior in C/C++ code.

## Usage

```
/c-debug <stack trace or error description>
/c-debug <coredump analysis>
/c-debug  (with crash output already in context)
```

## What This Does

1. Loads `technical-patterns.md`.
2. Uses the crash information (stack trace, assert message, address, signal)
   as entry points into call-graph analysis.
3. Traces the execution path that led to the failure, verifying each step with
   actual code — not assumptions.
4. Identifies the root cause, not just the crash site.
5. Produces a debug report with evidence and a suggested fix.

## Input Formats

All of the following work as input:
- Signal and address: `SIGSEGV at 0x...`
- Sanitizer output: AddressSanitizer, UBSan, ThreadSanitizer, MemorySanitizer
- Assert failure: `Assertion failed: (cond), file foo.c, line N`
- Stack trace (any format: GDB, LLDB, pstack, /proc/PID/backtrace)
- Valgrind report
- Description of observed vs expected behavior

## Analysis Approach

The debugger starts at the crash site and works outward:

1. **Crash site:** Identify the exact line and instruction that failed.
2. **Proximate cause:** What pointer was NULL? What index was out of bounds?
   What lock ordering was violated?
3. **Root cause:** How did the program reach that state? Trace backward through
   the call stack, checking invariants at each level.
4. **Contributing factors:** Race condition? Missing error check? Incorrect
   refcount? Allocator mismatch?

## Output

```
Root cause: <one-sentence summary>

Evidence:
  <file:line — quoted code showing the problem>
  <Explanation of why this code produces the observed failure>

Contributing factor (if any):
  <Secondary issue that enabled the root cause>

Suggested fix:
  <Concrete code change or approach>

Confidence: high | medium | low
  <What additional information would increase confidence>
```

## When to Use

- A process crashed and you have a stack trace.
- A sanitizer reported a memory error or data race.
- A test is failing intermittently.
- An assertion fires in production.
- A system hang or deadlock needs diagnosis.

Load `patterns/locking.md` for deadlock analysis, `patterns/memory-safety.md`
for memory errors, `patterns/atomics.md` for data races.
