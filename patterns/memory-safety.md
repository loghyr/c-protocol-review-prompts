# Memory Safety Patterns

Load this file when CHANGE categories involve memory allocation, deallocation,
pointer arithmetic, array indexing, or string operations.

---

## 1. Allocator Mismatch

Every allocation must be freed with the matching deallocator.

| Allocated with | Must free with |
|----------------|----------------|
| `malloc` / `calloc` / `realloc` | `free` |
| `new` (scalar) | `delete` |
| `new[]` (array) | `delete[]` |
| `mmap` | `munmap` |
| `posix_memalign` / `aligned_alloc` | `free` |
| Custom pool alloc | Matching pool free |

Mismatching (e.g., `free()` on `new`-allocated memory) is undefined behavior
and commonly causes heap corruption.

**Detection:** Find every allocation and locate its deallocation. Check that
the same API family is used.

---

## 2. NULL Return from Allocation

`malloc`, `calloc`, `realloc`, and `new` (with `std::nothrow`) can return NULL.
Using a NULL pointer is undefined behavior (crash on access).

**Verification:**
- Find every allocation call.
- Check whether the return value is tested before use.
- If C++ exceptions are enabled, `new` throws `std::bad_alloc` on failure (not NULL).
  Do not flag missing NULL checks in C++ projects with exceptions enabled — instead
  check for exception handling at higher levels.

```c
ptr = malloc(n);
if (!ptr) {
    return -ENOMEM;  /* correct: checked before use */
}
```

---

## 3. Double Free

Freeing the same pointer twice is undefined behavior.

**Detection:**
- Find every `free()` / `delete` call.
- Check whether the pointer could have been freed on a prior code path.
- Pay special attention to error paths that partially undo work.

**Mitigation pattern:** Set the pointer to NULL immediately after free so that
a subsequent free (via the same variable) becomes a safe no-op (`free(NULL)` is
defined to do nothing in C).

```c
free(ptr);
ptr = NULL;  /* safe: subsequent free(ptr) is a no-op */
```

---

## 4. Use-After-Free

Accessing memory after it has been freed is undefined behavior. Common sources:

- **Dangling pointer:** Pointer is freed; another path uses the pointer variable.
- **Object returned to pool / freed and reallocated:** Code holds a raw pointer
  to a pooled object; pool returns the object to another caller.
- **Callback called after cleanup:** A callback or signal handler fires after
  the object it references has been destroyed.

**Detection:**
- Trace the lifetime of every heap object from allocation to last use.
- Check every pointer passed to callbacks, stored in global state, or registered
  with an event loop / signal mechanism for lifetime correctness.
- The object must outlive every pointer to it.

---

## 5. Buffer Overflow (Write Beyond Bounds)

Writing past the end of an allocated buffer overwrites adjacent memory.

**Common sources:**
- Off-by-one in loop bounds: `for (i = 0; i <= n; i++)` writes `n+1` elements.
- Incorrect size in `memcpy` / `strcpy` / `snprintf` destination size argument.
- Reallocation that does not account for the NUL terminator in strings.
- Reading `strlen(src)` and then writing `strlen(src) + 1` bytes (including NUL)
  without checking that the destination has `strlen(src) + 1` bytes.

```c
char buf[16];
strncpy(buf, src, 16);     /* WRONG: does not NUL-terminate if src >= 16 chars */
snprintf(buf, sizeof(buf), "%s", src);  /* CORRECT: always NUL-terminates */
```

---

## 6. Integer Overflow in Size Calculations

When computing sizes for `malloc` or `memcpy`, intermediate arithmetic can
overflow before the result is passed to the allocator. This can cause
under-allocation followed by overflow writes.

```c
/* If n > SIZE_MAX / sizeof(struct foo), the multiply overflows */
size_t sz = n * sizeof(struct foo);  /* potential overflow */
ptr = malloc(sz);                    /* allocates too little */
```

**Detection:**
- Find every multiplication or addition used in an allocation size.
- Check whether the operands are bounded such that overflow is impossible.
- Prefer `calloc(n, sizeof(T))` over `malloc(n * sizeof(T))` — `calloc` is
  required by the C standard to detect overflow.

---

## 7. Signed/Unsigned Mismatch in Array Indexing

Mixing signed and unsigned types in array indexing or pointer arithmetic can
produce unexpected behavior when signed values are negative.

```c
int i = user_input();   /* may be negative */
char *p = buf + i;      /* negative i wraps to large unsigned offset → OOB */
```

**Detection:**
- Find every array index or pointer offset that derives from user input, network
  data, or file contents.
- Confirm the value is range-checked (>= 0 AND < array size) before use.

---

## 8. `realloc` Error Handling

`realloc` returns NULL on failure but does NOT free the original pointer. A
common bug:

```c
ptr = realloc(ptr, new_size);   /* WRONG: if NULL, old ptr is leaked */
```

**Correct pattern:**
```c
tmp = realloc(ptr, new_size);
if (!tmp) {
    free(ptr);      /* or handle error without freeing, depending on semantics */
    return -ENOMEM;
}
ptr = tmp;
```

---

## 9. Stack Buffer Sizing

Stack-allocated buffers have fixed size. Using `snprintf`, `strncat`, or similar
with a size derived from runtime data risks overflow if the size is incorrect.

- Do not use `strcpy` or `sprintf` for user-derived strings.
- Do not use variable-length arrays (VLAs) with unchecked user-controlled sizes —
  large VLAs exhaust stack space.

---

## 10. Memory Freed Without Clearing Pointer (Pooled Objects)

When an object is freed or returned to a pool and the struct that contained
the pointer is reused:

```c
void unregister(struct ctx *ctx) {
    ctx->dead = 1;
    free(ctx->buf);           /* freed, but ctx->buf still non-NULL */
    pool_add(ctx);            /* ctx returned to pool */
}

void register_new(struct ctx *ctx) {
    /* pulled from pool */
    if (ctx->buf) {
        use(ctx->buf);        /* USE-AFTER-FREE: buf was freed */
    }
}
```

**Rule:** When freeing a resource referenced by a struct field, set the field
to NULL immediately after — unless the struct itself is also freed in the same
operation, making the field inaccessible.

---

## Verification Output Requirements

For any memory safety issue:
1. Show the allocation site (function, file:line).
2. Show the problematic access or deallocation (function, file:line).
3. For integer overflow: show the arithmetic expression and the values that
   could cause overflow.
4. For buffer overflow: show the buffer size and the write size, proving the
   write can exceed the buffer.
