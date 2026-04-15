# Reference Counting Patterns

Load this file when CHANGE categories involve object lifecycle management:
creation, reference acquisition (`get`/`ref`/`retain`), release (`put`/`unref`/
`release`), or destruction.

---

## 1. Get/Put Balance

Every `get()` (or `ref()`, `retain()`, `addref()`) must have exactly one
matching `put()` (or `unref()`, `release()`) on every code path, including
error paths.

**Detection:**
- Count `get` calls that are NOT balanced with a `put` on error paths.
- Count `put` calls that happen without a prior `get` (double-put / over-release).

**Common pattern:**
```c
obj_get(obj);         /* +1 ref for new user */
ret = do_work(obj);
if (ret) {
    obj_put(obj);     /* error path MUST put */
    return ret;
}
/* ... success path ... */
obj_put(obj);         /* normal cleanup */
```

**Bidirectional check:** If the patch changes when `get` or `put` is called,
verify that all other code paths that hold the same reference still release
it. A change to one acquisition site may leave another site unbalanced.

---

## 2. Use-After-Put (Use-After-Free via Reference Count)

After calling `put()` and the put triggers destruction (refcount hits zero),
the object is freed. Accessing the object after a final `put()` is use-after-free.

**Detection:**
- Find every code path where `put()` is called and then the object is accessed.
- The access may be indirect (a pointer stored before the `put` is used after).

```c
obj_put(obj);         /* may destroy obj */
log_event(obj->name); /* WRONG: obj may be freed */
```

**Verification:** Check whether `put()` always frees, or only frees at zero.
If `put()` always decrements and the caller does not own the last reference,
access after `put()` may be valid. Verify ownership.

---

## 3. Increment After Zero

Reference counters must not be incremented after they drop to zero. An object
with a zero refcount is in the process of being destroyed.

**C11 `atomic_fetch_add` with check:**
```c
int old = atomic_fetch_add(&obj->refcount, 1, memory_order_relaxed);
if (old == 0) {
    /* WRONG: incremented a dead object */
    atomic_fetch_sub(&obj->refcount, 1, memory_order_relaxed);
    return NULL;
}
```

**Correct pattern (try-get):** Increment and check atomically that the old
value was > 0. If the old value was 0, the object is dying — back out and
return NULL.

This pattern is used for hash table lookups where the object may be concurrently
deleted. Always check whether the project's `obj_get_unless_zero()` (or
equivalent) is available and used at lookup sites.

---

## 4. Reference Counting in Concurrent Lookups (Hash Table / RCU)

When objects are stored in a hash table or RCU-protected list and can be
concurrently deleted:

```
Lookup (reader, under rcu_read_lock):
  1. Find object in hash table
  2. Attempt obj_get_unless_zero() — increment only if refcount > 0
  3. If failed (zero): object is being destroyed, return NULL
  4. Drop rcu_read_lock
  5. Use object (now protected by own reference)
  6. obj_put() when done

Deletion:
  1. Remove from hash table (under write lock or with CAS)
  2. Call synchronize_rcu() or schedule call_rcu()
  3. obj_put() for the hash table's reference
  4. Object destroyed when last reference is put
```

If the patch omits step 2 in the lookup (get_unless_zero), lookups are subject
to use-after-free.

---

## 5. Ownership Semantics at API Boundaries

When a function takes or returns a reference-counted object, the ownership
transfer must be documented and consistent:

- **Borrowed reference:** Caller owns the reference; function must NOT `put`.
- **Owned reference:** Function receives ownership; function MUST `put` (or
  transfer ownership further).
- **New reference returned:** Caller receives ownership; caller MUST `put`.

If the patch changes a function's ownership semantics without updating all
callers, the result is either a double-put (crash) or a leak.

**Verification:** Check the commit message and every call site for the changed
function.

---

## 6. `std::shared_ptr` / `std::weak_ptr` Patterns (C++)

`std::shared_ptr` and `std::weak_ptr` implement reference counting with RAII.
Watch for:

- **Circular shared_ptr references:** A → B → A via `shared_ptr` → memory leak.
  Use `weak_ptr` to break cycles.
- **`shared_ptr` from raw `this`:** Do not call `std::make_shared<T>(this)` or
  `shared_ptr<T>(this)` inside a member function unless the class inherits from
  `std::enable_shared_from_this`. Creating two separate `shared_ptr` objects for
  the same raw pointer gives two independent refcounts → double-free.
- **`weak_ptr::lock()` result:** Always check that `lock()` returns a non-null
  `shared_ptr` before using it. The object may have been destroyed.
- **Storing `shared_ptr` in containers and then erasing:** Erasing from a
  container drops the `shared_ptr`; confirm no other code holds a raw pointer
  to the object that outlives the container entry.

---

## Verification Output Requirements

For any reference counting issue:
1. Show the `get`/`put` sequence with file:line for each call.
2. For use-after-put: show the put and the subsequent access with locations.
3. For get-after-zero: show the counter value that permits the increment and
   why it is possible to reach zero before the increment.
4. For ownership issues: quote the function signature and all call sites affected.
