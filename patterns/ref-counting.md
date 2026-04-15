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

### Lock-free hash table (e.g., liburcu cds_lfht) full lifecycle

Objects in a lock-free hash table with per-object refcounting require careful
ordering at every lifecycle stage:

**Creation (the creation ref)**
```c
/* 1. Allocate and initialize ALL fields before hashing */
obj = calloc(1, sizeof(*obj));
urcu_ref_init(&obj->ref);     /* ref = 1: the creation ref */
populate_fields(obj);

/* 2. Hash after initialization, under rcu_read_lock */
rcu_read_lock();
cds_lfht_add(ht, hash, &obj->ht_node);
rcu_read_unlock();
```
The creation ref (count = 1) represents "this object is alive in the table."
It must remain held as long as the object should be findable.

**Lookup (take a find ref)**
```c
rcu_read_lock();
node = cds_lfht_lookup(ht, hash, match_fn, key, &iter);
if (node) {
    obj = caa_container_of(node, struct obj, ht_node);
    if (!urcu_ref_get_unless_zero(&obj->ref))
        obj = NULL;  /* racing destruction: zero refcount */
}
rcu_read_unlock();
/* use obj, then: */
if (obj)
    obj_put(obj);   /* drop find ref */
```

**Destruction ordering (removal BEFORE free)**
The release callback (invoked when refcount hits zero) MUST remove the
object from the hash table BEFORE scheduling the free via `call_rcu`.
Never do it the other way around.
```c
static void obj_release(struct urcu_ref *ref)
{
    struct obj *obj = caa_container_of(ref, struct obj, ref);
    rcu_read_lock();
    cds_lfht_del(ht, &obj->ht_node);   /* remove first */
    rcu_read_unlock();
    call_rcu(&obj->rcu_head, obj_rcu_free);  /* free after grace period */
}

static void obj_rcu_free(struct rcu_head *head)
{
    struct obj *obj = caa_container_of(head, struct obj, rcu_head);
    free(obj);
}
```
`cds_lfht_del` is idempotent — safe to call on an already-removed node.

**Explicit removal with outstanding refs**
To make an object unfindable immediately while deferring the free until all
refs drain:
```c
rcu_read_lock();
cds_lfht_del(ht, &obj->ht_node);   /* unfindable now */
rcu_read_unlock();
obj_put(obj);   /* drop creation ref; if last ref, release does del (no-op) + call_rcu */
```

**Drain at shutdown — use put(), not direct release()**
```c
/* WRONG: bypasses refcount, double-free if any thread holds a find ref */
rcu_read_lock();
cds_lfht_for_each_entry(ht, &iter, obj, ht_node) {
    cds_lfht_del(ht, &obj->ht_node);
    free(obj);   /* WRONG */
}
rcu_read_unlock();

/* CORRECT: advance iterator BEFORE put (put may trigger release -> del) */
rcu_read_lock();
cds_lfht_first(ht, &iter);
while ((node = cds_lfht_iter_get_node(&iter)) != NULL) {
    obj = caa_container_of(node, struct obj, ht_node);
    cds_lfht_next(ht, &iter);   /* advance BEFORE put */
    obj_put(obj);               /* drop creation ref */
}
rcu_read_unlock();
synchronize_rcu();
cds_lfht_destroy(ht, NULL);
```

**Iterator threads must be RCU-registered**
Any thread that calls `rcu_read_lock` and traverses the hash table must call
`rcu_register_thread()` at startup and `rcu_unregister_thread()` at exit.
Without registration, `rcu_read_lock` is a no-op and `call_rcu` callbacks
can fire while the unregistered thread appears to be in a read-side critical
section, causing use-after-free.

**call_rcu callbacks must not call synchronize_rcu**
A `call_rcu` callback runs in the RCU callback thread context. Calling
`synchronize_rcu()` from within a `call_rcu` callback deadlocks because
`synchronize_rcu` waits for all current callbacks to complete, including
the one that called it.
```c
/* WRONG: deadlock */
static void obj_rcu_free(struct rcu_head *head)
{
    struct obj *obj = ...;
    synchronize_rcu();   /* WRONG: deadlocks */
    free(obj);
}
```

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

## 7. NULL-Safe Cleanup Conventions

Many codebases implement `put()` functions that silently ignore NULL:
```c
void obj_put(struct obj *obj)
{
    if (!obj)
        return;
    /* ... */
}
```

When reviewing cleanup code on error paths and async state-transfer
paths, check whether the project's `put()` functions are NULL-safe.
If they are, a guard like `if (obj) obj_put(obj)` is redundant
(not a bug, but noise). If they are NOT NULL-safe, missing guards
on error paths that may leave the pointer unset are bugs.

**Detection:** Look for inconsistent guarding — some callers guarding,
others not — which indicates the NULL-safety contract is unclear or
inconsistent. Flag with NOTE; clarify the contract.

---

## Verification Output Requirements

For any reference counting issue:
1. Show the `get`/`put` sequence with file:line for each call.
2. For use-after-put: show the put and the subsequent access with locations.
3. For get-after-zero: show the counter value that permits the increment and
   why it is possible to reach zero before the increment.
4. For ownership issues: quote the function signature and all call sites affected.
5. For RCU hash table lifecycle: show the creation ref, the lookup (get_unless_zero),
   and the destruction ordering (removal before free).
