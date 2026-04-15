# C/C++ Technical Patterns Reference

## Core Instructions

- Trace full execution flows. Gather additional context from call chains to
  fully understand behavior before drawing conclusions.
- Never make assumptions based on return types, comments, assert() calls, or
  error handling patterns — verify correctness by tracing concrete execution paths.
- Never assume that changing an assert() or abort() changes what errors or
  conditions a function can accept. They indicate what is printed or aborted, nothing else.
- Never skip steps because a bug was found in a previous step.
- Documentation and comments are sometimes incomplete, outdated, or misleading:
  - Always read the actual implementation, not just the comment.
  - Check `#ifdef`/`#else` branches — the same comment may be copy-pasted to
    both sides with different semantics.
  - If a comment says "returns X" but the code shows conditional behavior, verify
    which branch applies.
- Never report errors without checking whether the error is impossible in the
  call path you found.
  - Some paths always check a condition before dereferencing.
  - Do not recommend defensive programming unless it fixes a proven bug.

---

## Execution Context Rules

- **Signal handlers:** Must only call async-signal-safe functions. Mutex acquisition
  is not safe. Prefer `sig_atomic_t` flags and let the main loop do real work.
- **Interrupt/callback context (embedded, kernel extensions):** May not block,
  sleep, or call allocators that can block. Check what context a function runs in
  before reporting a "missing lock" for a blocking primitive.
- **Thread-local storage:** `__thread` / `thread_local` variables are per-thread.
  Not shared. Do not report them as races.

---

## Error Handling

If code checks a condition via `assert()` or a project-specific macro (CHECK,
DCHECK, VERIFY, etc.), assume that condition will never happen unless you can
provide concrete evidence via code snippets and call traces.

---

## Bounds and Validation

**Important:** Never suggest defensive bounds checks unless you can prove the
source is untrusted (user input, network data, file contents, IPC messages).
Internal invariants checked by the design do not need redundant guards.

---

## Resource Management

Every resource must have a balanced lifecycle: alloc → init → use → cleanup → free.

- All pointers have the same size on a given platform. When allocating an array
  of pointers, use `sizeof(type *)` with the correct type for clarity.
- Reference counters do not get incremented after dropping to zero. A
  `refcount_dec_and_test` (or equivalent) returning "hit zero" means the object
  is being destroyed — do not use it after that.
- Global variables and static variables are zero-initialized automatically in C/C++.
- Zeroing allocators (`calloc`, C++ value-init, `memset` to 0) spare you from
  explicitly clearing pointer fields, but verify which allocator is in use.
- When freeing or destroying a resource referenced by a struct field, set the
  field to NULL immediately after freeing — unless the struct itself is also
  freed in the same operation. This prevents use-after-free if the struct is
  reused (e.g., pooled objects).
  - Safe: `free(foo->ptr); free(foo);` — nobody finds `foo->ptr` non-NULL because
    `foo` is gone.
  - Unsafe: `free(foo->ptr); add_to_pool(foo);` — the next user of the pool
    entry may skip allocation assuming `foo->ptr` is still valid.

---

## list_head / Intrusive List APIs

- `list_add(new, head)` and similar APIs initialize `new` by writing to it, but
  `head` must have been previously initialized.
- When objects are removed from lists, verify they are returned, freed, or
  otherwise accounted for. Removed-but-not-freed objects are leaks.

---

## for Loops

```
for (init; condition; advance) { body }
```

- `condition` is checked BEFORE executing `body`.
- `advance` runs only AFTER `body` completes.

Do not assume `advance` runs when `body` exits early (break, return, goto).
Do not assume `body` runs at least once.

---

## NULL Pointer Dereference

Review these examples carefully before analyzing NULL dereferences.
Misunderstanding what constitutes a dereference causes false positives.

**Dereference types:**
```c
val = *foo;            // dereferences foo
val = foo[n];          // dereferences foo; n is just read, not dereferenced
val = foo->field;      // dereferences foo; field is just read, not dereferenced
val = *foo->ptr;       // dereferences foo AND ptr
val = (*foo)->field;   // dereferences foo, dereferences *foo, reads field
val = foo->ptr->x;     // dereferences foo AND ptr; x is just read
```

**Guards:**
```c
if (foo) val = *foo;              // safe
if (foo) val = foo->field;        // safe — foo is guarded
if (foo && foo->ptr) val = *foo->ptr;  // safe — both guarded
```

**Key points:**
1. Reading a pointer field is NOT the same as dereferencing it.
   `ptr = foo->bar` dereferences `foo`, reads `bar`, but does NOT dereference `bar`.
   The dereference happens when you later use `ptr`.
2. Check where the pointer is actually used, not just where it is read.
3. `if (foo)` protects only `foo`. It does not protect `foo->bar`.

---

## ERR_PTR vs NULL (C pattern with encoded error pointers)

In projects that use pointer-encoded errors (Linux, some embedded frameworks):
```c
foo = ERR_PTR(-ENOMEM);
if (foo)       // TRUE — non-NULL, but *foo will CRASH
if (IS_ERR(foo)) // correct way to test for error
```

Know which error-checking convention the project uses before reporting a
"missing NULL check."

---

## strscpy / strlcpy / snprintf

- `strscpy()` (and `strlcpy()`) auto-detect destination array size when the
  compiler can infer the type. Do not add manual size arguments that could
  disagree with the actual array.
- `char *s = ""`: `strlen(s)` returns 0, but `s[0]` is safe to read (the NUL).
- `memcpy(dst + (offset & mask), src, size)` — alignment validation is usually
  performed elsewhere. Do not report the arithmetic as unsafe without proof
  that no alignment check exists.

---

## Global Arrays with MAX_FOO

If code has a global array sized `MAX_FOO`, check whether it is possible to
create more than `MAX_FOO` elements. If yes, check whether the insertion path
validates the count before writing.

---

## Signed/Unsigned Integer Comparison

Comparing signed and unsigned values without explicit casts can produce
surprising results when the signed value is negative (it wraps to a large
unsigned value). Check for this in loop bounds and array index calculations.

---

## Pointer Arithmetic Overflow

Adding a large offset to a pointer can overflow on 32-bit platforms or when
using `int` rather than `size_t` / `ptrdiff_t`. Check:
- Index variables used in pointer arithmetic are `size_t` or `ptrdiff_t`,
  not `int` or `unsigned`.
- Multiplication before use (e.g., `i * stride`) cannot overflow.

---

## Uninitialized Variables

- Assigning to a variable IS initializing it.
- Passing an uninitialized variable to a function is fine if that function
  writes to it before reading it.
- Only report reading from uninitialized variables, not writing to them.
- Zero-allocating functions (`calloc`, value-init, `memset(0)`) zero all bytes.
  Do not flag missing explicit initialization when the zero value is correct.

---

## C++ Exception Safety

If the codebase uses exceptions:
- Check that resources acquired before a `throw` are released. RAII objects
  handle this; raw pointers acquired before a `throw` without RAII do not.
- `catch (...)` / `noexcept` boundaries: ensure exception cannot propagate
  through C callbacks, signal handlers, or `extern "C"` functions.
- Do not report "exception safety" issues in codebases that disable exceptions
  (`-fno-exceptions`). Verify the build flags before reporting.

---

## Volatile Misuse

`volatile` prevents compiler optimization on a variable. It does NOT provide:
- Atomicity for read-modify-write operations.
- Memory ordering guarantees between threads.
- A replacement for mutexes or atomic operations.

Report `volatile` used for inter-thread communication as a bug. The correct
tool is `_Atomic` / `std::atomic` with appropriate memory ordering, or a mutex.
